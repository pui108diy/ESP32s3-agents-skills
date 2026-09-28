# PCNT (Pulse Counter) — New Driver — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "driver/pcnt.h"     // v5.3+/v6: component esp_driver_pcnt
```

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 38)
- **4 units อิสระ** นับได้ −32768..32767 (16-bit signed)
- **แต่ละ unit มี 2 channels** ทำงานอิสระหรือจับคู่กัน (quadrature)
- **1 glitch filter ต่อ unit** — กรอง pulse สั้นกว่า threshold (หน่วย APB cycles)
- Event ที่ตั้ง watch point ได้: **zero, high limit, low limit, threshold 2 ค่า**

## Structs
```c
typedef struct {
    int low_limit;             // ต้อง < 0
    int high_limit;            // ต้อง > 0
    struct { uint32_t accum_count: 1; } flags;   // สะสมทะลุ limit ได้ (v5.x)
    int intr_priority;
} pcnt_unit_config_t;

typedef struct { uint32_t max_glitch_ns; } pcnt_glitch_filter_config_t;

typedef struct {
    int edge_gpio_num;         // ขาสัญญาณนับ (−1 ถ้าไม่ใช้)
    int level_gpio_num;        // ขาคุมทิศทาง (−1 ถ้าไม่ใช้)
    struct { uint32_t invert_edge_input: 1; uint32_t invert_level_input: 1; } flags;
} pcnt_chan_config_t;

typedef struct {
    int watch_point_value;
    uint32_t status;
} pcnt_watch_event_data_t;

typedef struct {
    pcnt_watch_cb_t on_reach;  // ISR context!
} pcnt_event_callbacks_t;
typedef bool (*pcnt_watch_cb_t)(pcnt_unit_handle_t unit, const pcnt_watch_event_data_t *edata, void *user_ctx);
```

## Function Signatures (exact)
```c
esp_err_t pcnt_new_unit(const pcnt_unit_config_t *config, pcnt_unit_handle_t *ret_unit);
esp_err_t pcnt_del_unit(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_set_glitch_filter(pcnt_unit_handle_t unit, const pcnt_glitch_filter_config_t *config);

esp_err_t pcnt_new_channel(pcnt_unit_handle_t unit, const pcnt_chan_config_t *config, pcnt_channel_handle_t *ret_chan);
esp_err_t pcnt_del_channel(pcnt_channel_handle_t chan);
esp_err_t pcnt_channel_set_edge_action(pcnt_channel_handle_t chan,
                pcnt_channel_edge_action_t pos_act, pcnt_channel_edge_action_t neg_act);
// PCNT_CHANNEL_EDGE_ACTION_HOLD / _INCREASE / _DECREASE
esp_err_t pcnt_channel_set_level_action(pcnt_channel_handle_t chan,
                pcnt_channel_level_action_t high_act, pcnt_channel_level_action_t low_act);
// PCNT_CHANNEL_LEVEL_ACTION_KEEP / _INVERSE / _HOLD

esp_err_t pcnt_unit_add_watch_point(pcnt_unit_handle_t unit, int watch_point);    // ค่าต้องอยู่ใน (low,high)
esp_err_t pcnt_unit_remove_watch_point(pcnt_unit_handle_t unit, int watch_point);
esp_err_t pcnt_unit_register_event_callbacks(pcnt_unit_handle_t unit,
                const pcnt_event_callbacks_t *cbs, void *user_data);

esp_err_t pcnt_unit_enable(pcnt_unit_handle_t unit);    // ← ต้องเรียกก่อน start เสมอ!
esp_err_t pcnt_unit_disable(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_start(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_stop(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_clear_count(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_get_count(pcnt_unit_handle_t unit, int *value);  // int แบบ signed
```

## กับดัก Hallucination
1. โค้ด legacy `pcnt_config()` + `pcnt_evt_t` (`driver/pcnt.h` เวอร์ชันเก่า) — deprecated v5.0 **ถูกลบใน v6** → ใช้ new driver เท่านั้น
2. ข้าม `pcnt_unit_enable()` แล้ว `pcnt_unit_start()` → `ESP_ERR_INVALID_STATE`
3. `low_limit` ต้อง**ติดลบ**, `high_limit` ต้องบวก — ตั้งสองค่าเป็นบวก → error
4. พฤติกรรมชน limit (accum_count = 0): ชน high → **รีเซ็ตเป็น 0**, ชน low → **ค้างที่ค่านั้น** ⚠️ ยืนยันกับ docs ของเวอร์ชันที่ใช้; จะนับทะลุให้เปิด `.flags.accum_count`
5. `on_reach` รันใน **ISR** — ห้าม printf; คืน true เมื่อปลุก task
6. ใช้ 1 channel นับปุ่มกด → ตั้งแค่ `edge_gpio_num` (ใส่ `level_gpio_num = -1`) และ level_action ทั้งคู่เป็น KEEP
7. Encoder AB-phase ต้อง 2 phase ใน **channel เดียว** (edge = A, level = B) ตามตัวอย่าง — ไม่ใช่ 2 channels
8. นับความถี่สูงมาก (> ระดับ MHz) → PCNT โอเค แต่กด `pcnt_unit_get_count()` บ่อยเกิน = สูญ CPU; ใช้ watch point + ISR แทน polling
9. ลืม glitch filter กับ encoder/สวิตช์หลวม → นับเพี้ยนจาก noise

## ตัวอย่างมินิมัล (quadrature encoder)
```c
#include "driver/pcnt.h"

void encoder_init(void)
{
    pcnt_unit_handle_t unit = NULL;
    pcnt_unit_config_t unit_cfg = { .high_limit = 100, .low_limit = -100 };
    ESP_ERROR_CHECK(pcnt_new_unit(&unit_cfg, &unit));

    pcnt_glitch_filter_config_t filter = { .max_glitch_ns = 1000 };   // กรอง glitch 1 µs
    ESP_ERROR_CHECK(pcnt_unit_set_glitch_filter(unit, &filter));

    pcnt_chan_config_t chan_cfg = { .edge_gpio_num = 4, .level_gpio_num = 5 };  // A, B
    pcnt_channel_handle_t chan = NULL;
    ESP_ERROR_CHECK(pcnt_new_channel(unit, &chan_cfg, &chan));
    ESP_ERROR_CHECK(pcnt_channel_set_edge_action(chan,
        PCNT_CHANNEL_EDGE_ACTION_INCREASE, PCNT_CHANNEL_EDGE_ACTION_DECREASE));
    ESP_ERROR_CHECK(pcnt_channel_set_level_action(chan,
        PCNT_CHANNEL_LEVEL_ACTION_KEEP, PCNT_CHANNEL_LEVEL_ACTION_INVERSE));

    ESP_ERROR_CHECK(pcnt_unit_enable(unit));
    ESP_ERROR_CHECK(pcnt_unit_clear_count(unit));
    ESP_ERROR_CHECK(pcnt_unit_start(unit));

    int count = 0;
    pcnt_unit_get_count(unit, &count);      // ตำแหน่ง encoder ปัจจุบัน
}
```