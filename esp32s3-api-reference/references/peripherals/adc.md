# ADC Oneshot (SAR ADC) — ESP-IDF v5.x/v6.x (ESP32-S3)

## Includes
```c
#include "esp_adc/adc_oneshot.h"    // ← อยู่ใน component esp_adc ไม่ใช่ driver/!
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h" // curve/line fitting scheme
```

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 39)
- **2 × 12-bit SAR ADC** วัดได้จาก **20 pin**
- **ADC1 = GPIO1–10** (ADC_CHANNEL_0–9), **ADC2 = GPIO11–20** (ADC_CHANNEL_0–9)
- Vref ภายใน = **1100 mV** (โดยการออกแบบ); วัดค่าเกิน Vref ต้องเปิด attenuation
- Attenuation บน S3: **0 / 2.5 / 6 / 12 dB** (TRM pattern table) — ที่ 12 dB วัดได้ราว 0–3.1V
- **ADC2 ใช้ร่วมกับ WiFi** → อ่านไม่ได้ขณะ WiFi ทำงาน → ให้ใช้ ADC1

## ตาราง GPIO ↔ Channel (TRM ตาราง 39.3-1)
| ADC | GPIO | Channel |
|---|---|---|
| ADC1 | GPIO1–GPIO10 | ADC_CHANNEL_0 … ADC_CHANNEL_9 |
| ADC2 | GPIO11–GPIO20 | ADC_CHANNEL_0 … ADC_CHANNEL_9 |

## Structs
```c
typedef struct {
    adc_unit_t unit_id;        // ADC_UNIT_1 / ADC_UNIT_2
    adc_ulp_mode_t ulp_mode;   // v5.0–v5.1: ADC_ULP_MODE_DISABLE; v5.2+ เป็น bool false ⚠️
} adc_oneshot_unit_init_cfg_t;

typedef struct {
    adc_bitwidth_t bitwidth;   // ADC_BITWIDTH_12 (v5.1+ มี ADC_BITWIDTH_DEFAULT)
    adc_atten_t atten;         // ADC_ATTEN_DB_0 / _2_5 / _6 / _12
} adc_oneshot_chan_cfg_t;

typedef struct {
    adc_unit_t unit_id;
    adc_channel_t chan;
    adc_atten_t atten;
    adc_bitwidth_t bitwidth;
} adc_cali_curve_fitting_config_t;   // S3 ใช้ curve fitting
```

## Function Signatures (exact — ชุดเต็ม)
```c
// --- Unit ---
esp_err_t adc_oneshot_new_unit(const adc_oneshot_unit_init_cfg_t *init_config,
                               adc_oneshot_unit_handle_t *ret_unit);
esp_err_t adc_oneshot_del_unit(adc_oneshot_unit_handle_t handle);

// --- Channel ---
esp_err_t adc_oneshot_config_channel(adc_oneshot_unit_handle_t handle, adc_channel_t channel,
                                     const adc_oneshot_chan_cfg_t *config);

// --- Read ---
esp_err_t adc_oneshot_read(adc_oneshot_unit_handle_t handle, adc_channel_t channel, int *out_raw);
// คืนค่าดิบ 0–4095 (12 บิต) — ไม่ใช่ mV!

// --- IO ↔ Channel helper ---
esp_err_t adc_oneshot_io_to_channel(int io_num, adc_unit_t *unit_id, adc_channel_t *channel);
esp_err_t adc_oneshot_channel_to_io(adc_unit_t unit_id, adc_channel_t channel, int *io_num);

// --- Calibration (S3 = curve fitting) ---
esp_err_t adc_cali_create_scheme_curve_fitting(const adc_cali_curve_fitting_config_t *config,
                                               adc_cali_handle_t *ret_handle);
esp_err_t adc_cali_raw_to_voltage(adc_cali_handle_t handle, int raw, int *voltage); // คืน mV
esp_err_t adc_cali_delete_scheme(adc_cali_handle_t handle);
esp_err_t adc_oneshot_get_calibrated_result(adc_oneshot_unit_handle_t handle, adc_cali_handle_t cali_handle,
                                            adc_channel_t chan, int *cali_result); // สะดวก: read+convert ในครั้งเดียว
```

## Version Markers
- `driver/adc.h` (legacy: `adc1_config_width`, `adc1_get_raw`, `hall_sensor_read`) — deprecated v5.0, **ถูกลบใน v6** → ใช้ `esp_adc/adc_oneshot.h` เท่านั้น
- `ADC_ATTEN_DB_11` (จาก ESP32 รุ่นแรก) → บน S3 ระดับที่ 4 คือ **12 dB** ตาม TRM → IDF เวอร์ชันใหม่ใช้ `ADC_ATTEN_DB_12` ⚠️ ตรวจ enum ใน `esp_adc/adc_types.h` ของเวอร์ชันที่ใช้
- Macro เช็ค chip: `#if ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED` (S3 = curve fitting; ESP32/S2/C2 = line fitting)

## กับดัก Hallucination
1. ส่ง GPIO number (เช่น 4) แทน `adc_channel_t` → GPIO4 = **ADC_CHANNEL_3** ของ ADC1 ไม่ใช่ channel 4
2. ใช้ **ADC2 ขณะ WiFi เปิด** → อ่าน fail/ค่าเพี้ยน → ย้ายมา ADC1
3. เอา raw 0–4095 ไปคูณเป็น voltage เอง → ค่าเพี้ยนมาก ต้องผ่าน calibration (`adc_cali_raw_to_voltage`)
4. `#include "driver/adc_oneshot.h"` → ผิด ต้องเป็น `esp_adc/adc_oneshot.h`
5. โค้ด legacy `adc1_config_channel_atten()` → ถูกลบแล้วใน v6
6. สัญญาณ analog เกิน ~3.1V (แม้เปิด 12 dB) → ตัดเกิน และเสี่ยงพัง pin
7. อ่านครั้งเดียวแล้วเชื่อ → ADC ค่อนข้าง noise ควรเฉลี่ยหลาย sample
8. Calibration handle สร้าง **ต่อ channel** (curve fitting config มี field chan) ไม่ใช่ต่อ unit

## ตัวอย่างมินิมัล (อ่าน GPIO4 = ADC1_CH3)
```c
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

void app_main(void)
{
    adc_oneshot_unit_handle_t adc1;
    adc_oneshot_unit_init_cfg_t init_cfg = { .unit_id = ADC_UNIT_1, .ulp_mode = ADC_ULP_MODE_DISABLE };
    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_cfg, &adc1));

    adc_oneshot_chan_cfg_t chan_cfg = { .bitwidth = ADC_BITWIDTH_DEFAULT, .atten = ADC_ATTEN_DB_12 };
    ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1, ADC_CHANNEL_3, &chan_cfg)); // GPIO4

    // calibration (curve fitting บน S3)
    adc_cali_handle_t cali = NULL;
    adc_cali_curve_fitting_config_t cali_cfg = {
        .unit_id = ADC_UNIT_1, .chan = ADC_CHANNEL_3,
        .atten = ADC_ATTEN_DB_12, .bitwidth = ADC_BITWIDTH_DEFAULT,
    };
    ESP_ERROR_CHECK(adc_cali_create_scheme_curve_fitting(&cali_cfg, &cali));

    int raw = 0, mv = 0;
    ESP_ERROR_CHECK(adc_oneshot_read(adc1, ADC_CHANNEL_3, &raw));
    ESP_ERROR_CHECK(adc_cali_raw_to_voltage(cali, raw, &mv));   // raw → mV
}
```