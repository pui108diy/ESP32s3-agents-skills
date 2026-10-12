# 3-7 ADC (SAR ADC) — v2.0.1 (Major Review)
> Target: ESP32-S3 | ESP-IDF v5.2 – v6.1
> อ้างอิง: TRM v1.8 ch.39 (On-Chip Sensors and Analog Signal Processing) | Datasheet v2.2 | adc_oneshot.h / adc_continuous.h / adc_cali.h v6.1 (header จริง) | adc_cali_schemes.h (S3: `ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED=1`) | soc_caps.h v6.1 | Migration Guide v6.1 | docs.espressif.com ADC oneshot/continuous/calibration (S3)
> เปลี่ยนแปลง v2.0: **แก้ hallucination v1.x 2 จุด** (cali "สร้างต่อ channel" — จริง chan เป็น reserved ไม่มีผล · DB_12 "วัดได้ 0–3.1V" — จริง ~2450 mV บน S3) · เพิ่ม section Continuous (DMA) + กับดัก DMA=ADC1 เท่านั้น · component = `esp_adc` (ไม่ใช่ esp_driver_*) · `adc_oneshot_get_calibrated_result` (v6.1) · `.format` deprecated (v6.1) · Vref จริง 1000–1200 mV · โครงสร้างใหม่ตาม skill v2.7

## 0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. Include เป็น **`esp_adc/adc_oneshot.h`** (component `esp_adc` — **ไม่ใช่** `driver/` และ**ไม่ใช่** `esp_driver_adc` แบบ peripheral อื่น)
2. **ADC1 = GPIO1–10 (CH0–9), ADC2 = GPIO11–20 (CH0–9)** → **GPIO4 = ADC1_CH3** (channel = เลข GPIO − 1)
3. **ADC2 ชนกับ WiFi** (arbiter แย่ง SAR ADC2 ระหว่าง RTC controller กับ PWDET) → งาน WiFi ใช้ ADC1
4. **Continuous (DMA) mode ใช้ได้กับ ADC1 เท่านั้นบน S3** — ADC2 ไม่มีทาง DMA
5. Attenuation บน S3: `ADC_ATTEN_DB_0 / _2_5 / _6 / _12` — **ไม่มี DB_11** (นั่นของ ESP32 รุ่นแรก)
6. ที่ DB_12 วัดได้สูงสุด **~2450 mV** (ไม่ใช่ 3100 แบบ ESP32)
7. raw 0–4095 → mV ต้องผ่าน **`adc_cali_raw_to_voltage()`** — ห้ามคูณเอง (Vref จริง 1000–1200 mV ต่างกันทุก chip)
8. S3 ใช้ **curve fitting** (`ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED=1`) — ไม่มี line fitting
9. cali config ผูกกับ **unit + atten (+bitwidth)** — field `chan` เป็น **reserved ไม่มีผล**
10. ลด noise: อ่านหลายครั้งแล้วเฉลี่ย (docs มีหัวข้อ Minimize Noise ทางการ)

## 1. Include & CMakeLists

```c
#include "esp_adc/adc_oneshot.h"      // oneshot (single conversion)
#include "esp_adc/adc_continuous.h"   // continuous (DMA) — เฉพาะงานที่ต้อง stream
#include "esp_adc/adc_cali.h"         // adc_cali_raw_to_voltage / adc_cali_delete_scheme
#include "esp_adc/adc_cali_scheme.h"  // adc_cali_create_scheme_curve_fitting
```

```cmake
# v5.0+ (รวม v5.2 – v6.1): ADC อยู่ใน component ชื่อ "esp_adc" — ชื่อพิเศษ ไม่ใช่ esp_driver_*!
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_adc)
```
⚠️ legacy: `driver/adc.h` (`adc1_config_channel_atten()` ฯลฯ) และ `esp_adc_cal.h` — deprecated ตั้งแต่ v5.0 และ**ถูกลบใน v6.0** (Migration Guide v6.1)

## 2. Hardware Facts (TRM ch.39 + soc_caps v6.1)
- **2 × 12-bit SAR ADC** — 10 channels ต่อ unit (20 ขา analog)
- **Mapping (TRM Table 39.3-1):** ADC1: GPIO1=CH0 … GPIO10=CH9 · ADC2: GPIO11=CH0 … GPIO20=CH9 → **GPIO4 = ADC1_CH3**, GPIO14 = ADC2_CH3
- **Vref:** 1100 mV โดยการออกแบบ (TRM 39.3.5) แต่**ค่าจริงต่างกันทุก chip ราว 1000–1200 mV** → ต้อง calibration
- **Attenuation (TRM 39.3.5):** 0 dB (k≈100%) · 2.5 dB (k≈75%) · 6 dB (k≈50%) · **12 dB (k≈25%)** — ช่วงวัดแนะนำ: DB_0 ≈ 0–950 · DB_2_5 ≈ 0–1250 · DB_6 ≈ 0–1750 · **DB_12 ≈ 0–2450 mV** (Datasheet S3 — ต่างจาก ESP32 DB_11 ที่ 3100)
- **SAR ADC2 Arbiter (TRM 39.3.8):** ADC2 ถูกคุมโดย RTC ADC2 controller กับ **PWDET (WiFi)** — ใช้ร่วมกันไม่ได้; Datasheet: "ADC2_CH… cannot be used with Wi-Fi simultaneously"
- **DMA = DIG ADC1 เท่านั้น (TRM 39.3.7.2):** digital controller + DMA ต่อกับ SAR ADC1 อย่างเดียว (`SOC_ADC_DIG_SUPPORTED_UNIT(UNIT) (UNIT==0)`) → **continuous mode ใช้ ADC1 ได้เท่านั้น**
- Pattern table: สูงสุด **24 entries** (`SOC_ADC_PATT_LEN_MAX`) — continuous สแกนหลาย channel ตามลำดับ pattern
- DMA data format (TRM 39.3.7.6): 32-bit word = 12-bit data + 3-bit ch_sel
- DIG ADC ใช้ 25 clock cycles ต่อ sample → sample rate จำกัดโดย DIGADC_SARCLK
- RTC ADC controller (oneshot path) ทำงานใน RTC domain — ปลุกด้วย ULP ได้

## 3. Oneshot API (adc_oneshot.h v6.1 จริง)
```c
typedef struct {
    adc_unit_t unit_id;      // ADC_UNIT_1 / ADC_UNIT_2
    adc_ulp_mode_t ulp_mode; // ADC_ULP_MODE_DISABLE / ADC_ULP_MODE_RISCV
} adc_oneshot_unit_init_cfg_t;
// ⚠️ ulp_mode เปลี่ยนชนิดตามเวอร์ชัน: v5.0–v5.1 enum · v5.2–v5.x bool · v6.1 กลับเป็น enum (ยืนยัน header)
//   — เปิด header ของเวอร์ชันที่ใช้ก่อน (§10)

typedef struct {
    adc_atten_t atten;       // ADC_ATTEN_DB_0 / _2_5 / _6 / _12
    adc_bitwidth_t bitwidth; // ADC_BITWIDTH_12 (หรือ ADC_BITWIDTH_DEFAULT)
} adc_oneshot_chan_cfg_t;

esp_err_t adc_oneshot_new_unit(const adc_oneshot_unit_init_cfg_t *init_config, adc_oneshot_unit_handle_t *ret_unit);
esp_err_t adc_oneshot_config_channel(adc_oneshot_unit_handle_t handle, adc_channel_t channel, const adc_oneshot_chan_cfg_t *config);
esp_err_t adc_oneshot_read(adc_oneshot_unit_handle_t handle, adc_channel_t chan, int *out_raw);
//   คืน ESP_ERR_TIMEOUT = ผลลัพธ์ invalid (เช่น ADC2 ถูก WiFi ยึด)
esp_err_t adc_oneshot_del_unit(adc_oneshot_unit_handle_t handle);
esp_err_t adc_oneshot_get_calibrated_result(adc_oneshot_unit_handle_t handle, adc_cali_handle_t cali_handle, adc_channel_t chan, int *cali_result);
//   ↑ convenience: อ่าน + แปลง mV ใน call เดียว (ยืนยันมีใน v6.1 — เช็คเวอร์ชันเก่าก่อนใช้ §10)

// helper แปลงเลขขา ↔ channel:
esp_err_t adc_oneshot_io_to_channel(int io_num, adc_unit_t *unit_id, adc_channel_t *channel);
esp_err_t adc_oneshot_channel_to_io(adc_unit_t unit_id, adc_channel_t channel, int *io_num);
```

## 4. Continuous API (DMA — ADC1 เท่านั้นบน S3)
```c
typedef struct {
    uint32_t max_store_buf_size;  // ขนาด pool รวม (bytes)
    uint32_t conv_frame_size;     // ต้องเป็นจำนวนเท่าของ SOC_ADC_DIGI_DATA_BYTES_PER_CONV (=4)
    struct { uint32_t flush_pool: 1; } flags;
} adc_continuous_handle_cfg_t;

typedef struct {
    adc_atten_t atten;
    adc_channel_t channel;
    adc_unit_t unit;
    adc_bitwidth_t bitwidth;
} adc_digi_pattern_config_t;      // จาก hal/adc_types.h

typedef struct {
    uint32_t pattern_num;                    // ≤ 24 (SOC_ADC_PATT_LEN_MAX)
    adc_digi_pattern_config_t *adc_pattern;
    uint32_t sample_freq_hz;                 // ช่วงตาม soc/soc_caps.h
    adc_digi_convert_mode_t conv_mode;       // ADC_CONV_SINGLE_UNIT_1 บน S3 (DMA ทำได้เฉพาะ ADC1!)
    adc_digi_output_format_t format __attribute__((deprecated));  // ⚠️ v6.1: deprecated — driver เลือก format เอง
} adc_continuous_config_t;

esp_err_t adc_continuous_new_handle(const adc_continuous_handle_cfg_t *hdl_config, adc_continuous_handle_t *ret_handle);
esp_err_t adc_continuous_config(adc_continuous_handle_t handle, const adc_continuous_config_t *config);
esp_err_t adc_continuous_start(adc_continuous_handle_t handle);
esp_err_t adc_continuous_stop(adc_continuous_handle_t handle);
esp_err_t adc_continuous_read(adc_continuous_handle_t handle, uint8_t *buf, uint32_t length_max, uint32_t *out_length, uint32_t timeout_ms);  // timeout หน่วย ms
esp_err_t adc_continuous_flush_pool(adc_continuous_handle_t handle);   // ⚠️ v6.1 ชื่อนี้ (ตัวเก่า adc_continuous_flush — เช็ค §10)
esp_err_t adc_continuous_register_event_callbacks(adc_continuous_handle_t handle, const adc_continuous_evt_cbs_t *cbs, void *user_data);
esp_err_t adc_continuous_deinit(adc_continuous_handle_t handle);
```
- Callback event: `on_conv_done` (ได้ `adc_continuous_evt_data_t {conv_frame_buffer, size}`) · `on_pool_ovf` — รันใน ISR context
- อ่านแบบ poll: `adc_continuous_read(handle, buf, sizeof(buf), &ret, timeout_ms)` → แกะ 12-bit data + 3-bit ch_sel เองตาม TRM 39.3.7.6


## 5. Calibration (curve fitting — S3)
```c
// esp_adc/adc_cali_scheme.h (v6.1 จริง)
typedef struct {
    adc_unit_t unit_id;       // ผูกกับ unit
    adc_channel_t chan;       // ⚠️ มีผลเฉพาะ chip ที่ define SOC_ADC_CALIB_CHAN_COMPENS_SUPPORTED
                              //    (calibration แบบ per-channel) — S3 ไม่ define → ไม่มีผลบน S3
    adc_atten_t atten;        // ผูกกับ atten
    adc_bitwidth_t bitwidth;
} adc_cali_curve_fitting_config_t;

esp_err_t adc_cali_create_scheme_curve_fitting(const adc_cali_curve_fitting_config_t *config, adc_cali_handle_t *ret_handle);
esp_err_t adc_cali_delete_scheme_curve_fitting(adc_cali_handle_t handle);   // delete เฉพาะ scheme — มีจริงใน header

// จาก esp_adc/adc_cali.h (ทางเลือกแบบ generic):
esp_err_t adc_cali_raw_to_voltage(adc_cali_handle_t handle, int raw, int *voltage);
esp_err_t adc_cali_delete_scheme(adc_cali_handle_t handle);
```
- **บน S3:** calibration ผูกกับ unit + atten (+bitwidth) เท่านั้น (`SOC_ADC_CALIB_CHAN_COMPENS_SUPPORTED` ไม่ถูก define) → สร้าง handle หนึ่งตัวต่อ unit+atten แล้วใช้กับทุก channel ของ unit นั้นได้ — ตั้ง `.chan` ไปก็ไม่มีผล
- **การ delete มี 2 ทางเลือก** — เลือกตัวใดตัวหนึ่ง: `adc_cali_delete_scheme_curve_fitting()` (เฉพาะ scheme) หรือ `adc_cali_delete_scheme()` (generic)
- **line fitting** (`adc_cali_line_fitting_config_t` + `adc_cali_create_scheme_line_fitting()` + `adc_cali_delete_scheme_line_fitting()`) มีใน API set แต่ถูกครอบด้วย `#if ADC_CALI_SCHEME_LINE_FITTING_SUPPORTED` ซึ่ง S3 ไม่ define (adc_cali_schemes.h ของ S3 มีแค่ `ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED 1`) → compile ออกบน S3 — อย่า copy จากโค้ด ESP32 รุ่นแรก (โดยเฉพาะ field `default_vref` ที่ `#if CONFIG_IDF_TARGET_ESP32` เท่านั้น)
- ทำไมต้อง calibrate: **Vref จริง 1000–1200 mV ต่างกันทุก chip** (ออกแบบ 1100 mV) — คูณเอง = เพี้ยนหลักสิบ mV ถึงร้อย mV
- ลด noise (docs "Minimize Noise"): อ่านหลาย sample แล้วเฉลี่ย · ตั้ง bitwidth ครบ 12 · แหล่งจ่าย/สายให้นิ่ง

## 6. กับดักที่พบบ่อย (ผูก HALLUCINATION-HOT-LIST v3.2)
1. ส่ง GPIO number แทน channel → **GPIO4 = ADC1_CH3** (channel = GPIO − 1) → **ADC-03**
2. ใช้ ADC2 ขณะ WiFi เปิด → fail/`ESP_ERR_TIMEOUT`/ค่าเพี้ยน → **ADC-01**
3. เอา raw ไปคูณเป็นแรงดันเอง → ต้อง `adc_cali_raw_to_voltage()` → **ADC-06**
4. `#include "driver/adc_oneshot.h"` หรือ `driver/adc.h` / `esp_adc_cal.h` → ผิด/ลบแล้ว → `esp_adc/*` → **ADC-04**
5. คิดว่า cali ต้องสร้างต่อ channel → ไม่ใช่ — `chan` เป็น reserved → **ADC-05**
6. ใช้ continuous/DMA กับ ADC2 → ไม่ได้ บน S3 → **ADC-07**
7. ใช้ `ADC_ATTEN_DB_11` → ของ ESP32 รุ่นแรก — S3 ใช้ `ADC_ATTEN_DB_12` → **ADC-02**
8. คิดว่า DB_12 วัดได้ 3.1V → ~2450 mV บน S3 → **ADC-08**
9. CMakeLists `REQUIRES esp_driver_adc` → ไม่มี component นี้ — ใช้ `esp_adc` → **ADC-04**
10. อ่านครั้งเดียวเชื่อเลย → ADC มี noise — เฉลี่ยหลาย sample → (§5)
11. สัญญาณเกินช่วง atten ที่ตั้ง → อิ่มตัว (ค่าติด 4095) — เลือก atten ให้เหมาะกับสัญญาณ → (§2)
12. `conv_frame_size` ไม่เป็นจำนวนเท่าของ 4 bytes → error → (§4)

## 7. ตัวอย่างมินิมัล (oneshot + calibration — แก้จาก v1.x: ตัด `.chan` ที่ไม่มีผลออก)
```c
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

void app_main(void)
{
    adc_oneshot_unit_handle_t adc1;
    adc_oneshot_unit_init_cfg_t init_cfg = { .unit_id = ADC_UNIT_1, .ulp_mode = ADC_ULP_MODE_DISABLE };
    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_cfg, &adc1));

    adc_oneshot_chan_cfg_t chan_cfg = { .atten = ADC_ATTEN_DB_12, .bitwidth = ADC_BITWIDTH_DEFAULT };
    ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1, ADC_CHANNEL_3, &chan_cfg));  // GPIO4

    adc_cali_handle_t cali = NULL;
    adc_cali_curve_fitting_config_t cali_cfg = {
        .unit_id = ADC_UNIT_1,          // ผูก unit + atten — ไม่ต้องระบุ channel
        .atten = ADC_ATTEN_DB_12,
        .bitwidth = ADC_BITWIDTH_DEFAULT,
    };
    ESP_ERROR_CHECK(adc_cali_create_scheme_curve_fitting(&cali_cfg, &cali));

    int raw = 0, mv = 0;
    uint32_t sum = 0;
    for (int i = 0; i < 16; i++) {              // เฉลี่ย 16 sample ลด noise
        ESP_ERROR_CHECK(adc_oneshot_read(adc1, ADC_CHANNEL_3, &raw));
        sum += raw;
    }
    ESP_ERROR_CHECK(adc_cali_raw_to_voltage(cali, sum / 16, &mv));  // raw → mV

    adc_cali_delete_scheme(cali);
    adc_oneshot_del_unit(adc1);
}
```

## 8. v6.x Notes
- legacy `driver/adc.h` + `esp_adc_cal.h` **ลบใน v6.0** (deprecated ตั้งแต่ v5.0)
- `adc_continuous_config_t.format` **deprecated ใน v6.1** — driver เลือก output format เอง (บน S3 มี format เดียว) — อย่าตั้งค่านี้ในโค้ดใหม่
- `adc_oneshot_get_calibrated_result()` — convenience อ่าน+แปลงใน call เดียว (ยืนยัน v6.1)

## 9. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
- **`ulp_mode` ชนิดตามเวอร์ชัน**: v5.0–v5.1 enum → v5.2–v5.x bool → v6.1 enum (ยืนยันแล้วจาก header แนบ) — เปิด adc_oneshot.h ของเวอร์ชันเป้าหมาย
- **`adc_oneshot_get_calibrated_result()`** — มีตั้งแต่เวอร์ชันไหน (ยืนยัน v6.1; ยังไม่เช็ค v5.2–v5.5)
- **`adc_continuous_flush_pool` vs `adc_continuous_flush`** — ชื่อตามเวอร์ชัน (ยืนยัน v6.1 = flush_pool)
- ช่วง **sample_freq_hz** ต่ำสุด/สูงสุดของ S3 จาก `SOC_ADC_SAMPLE_FREQ_THRES_LOW/HIGH` (ยังไม่ได้เปิด soc_caps ส่วนนี้ — เช็คก่อนตั้งค่า)
- ช่วงวัดแนะนำต่อ atten (950/1250/1750/2450 mV) — ตัวเลขจาก Datasheet/docs; ถ้างานวักด-sensitive ทดสอบจริง
- `SOC_ADC_DIGI_DATA_BYTES_PER_CONV` = 4 บน S3 (สำหรับ conv_frame_size) — เช็ค soc_caps ของเวอร์ชันที่ใช้