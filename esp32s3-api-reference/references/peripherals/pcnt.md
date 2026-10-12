# PCNT (Pulse Counter) — New Driver — v1.1

> ESP-IDF v5.x/v6.x · ESP32-S3 · **สถานะ: Phase 1 แปลงโครงสร้าง §0–§10 แล้ว — ยังไม่ได้ตรวจเชิงลึกกับ header จริง (ดู §10)**
> เนื้อหาทั้งหมดย้ายจากไฟล์เดิม ไม่มีการเติมข้อมูลเทคนิคใหม่ที่ยังไม่ผ่านการตรวจ

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. S3 มี **4 units อิสระ** นับได้ **−32768 ถึง 32767** (16-bit signed) — แต่ละ unit มี **2 channels**
2. New driver เท่านั้น — legacy `pcnt_config()` + `pcnt_evt_t` deprecated v5.0 **ถูกลบใน v6**
3. ลำดับ: `pcnt_new_unit → pcnt_new_channel → set actions → pcnt_unit_enable → pcnt_unit_start` — ข้าม `enable` = error
4. `low_limit` ต้อง**ติดลบ**, `high_limit` ต้อง**เป็นบวก**
5. `accum_count = 0` (default): ชน high → รีเซ็ตเป็น 0, ชน low → ค้างที่ค่านั้น
6. Watch point ตั้งได้: **zero, high limit, low limit, threshold 2 ค่า** — ค่าต้องอยู่ในช่วง (low, high)
7. `on_reach` รันใน **ISR** — ห้าม `printf`; คืน `true` เพื่อปลุก task
8. Encoder AB-phase: 2 phase ใน **channel เดียว** (edge = A, level = B) — **ไม่ใช่ 2 channels**
9. นับปุ่มกด: ตั้งแค่ `edge_gpio_num` (`level_gpio_num = -1`) + level action ทั้งคู่เป็น KEEP
10. อย่า polling `pcnt_unit_get_count()` บ่อย (สูญ CPU) — ใช้ watch point + ISR แทน; อย่าลืม glitch filter กับ encoder/สวิตช์

## §1. Include & CMakeLists
```c
#include "driver/pcnt.h"     // v5.3+/v6: component esp_driver_pcnt
```
- **CMakeLists:** ⚠️ ยังไม่ได้ตรวจว่าต้องระบุ `esp_driver_pcnt` (หรือ `driver` ใน v5.2) ใน `REQUIRES` หรือถูกดึงมาแบบ transitively — อยู่ในคิวตรวจ §10

## §2. Hardware Facts (TRM ESP32-S3 v1.8 บทที่ 38)
- **4 units อิสระ** นับได้ **−32768 ถึง 32767** (16-bit signed)
- แต่ละ unit มี **2 channels** (ทำงานอิสระ หรือจับคู่ทำ quadrature)
- **1 glitch filter ต่อ unit** — กรอง pulse สั้นกว่า threshold (หน่วย APB cycles ในระดับฮาร์ดแวร์; driver รับค่าเป็น ns)
- Event watch point: **zero, high limit, low limit และ threshold 2 ค่า**

## §3. Structs & Driver Setup
```c
typedef struct {
    int low_limit;          // ต้อง < 0
    int high_limit;         // ต้อง > 0
    struct { uint32_t accum_count: 1; } flags;   // 1 = สะสมค่าเกิน limit (int64 ใน software)
    int intr_priority;      // 0 = default
} pcnt_unit_config_t;

typedef struct {
    uint32_t max_glitch_ns; // กรอง pulse สั้นกว่านี้
} pcnt_glitch_filter_config_t;

typedef struct {
    int edge_gpio_num;      // ขาสัญญาณขอบ (A ของ encoder)
    int level_gpio_num;     // ขาระดับ (B ของ encoder); ปุ่มกด → ใส่ -1
    struct { uint32_t invert_edge_input: 1; uint32_t invert_level_input: 1; } flags;
} pcnt_chan_config_t;

typedef struct {
    int watch_point_value;
    uint32_t status;
} pcnt_watch_event_data_t;

typedef struct {
    pcnt_event_cb_t on_reach;   // ISR context
} pcnt_event_callbacks_t;
```

## §4. Function Signatures
```c
// --- Unit ---
esp_err_t pcnt_new_unit(const pcnt_unit_config_t *config, pcnt_unit_handle_t *ret_unit);
esp_err_t pcnt_del_unit(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_set_glitch_filter(pcnt_unit_handle_t unit, const pcnt_glitch_filter_config_t *config);

// --- Channel ---
esp_err_t pcnt_new_channel(pcnt_unit_handle_t unit, const pcnt_chan_config_t *config, pcnt_channel_handle_t *ret_chan);
esp_err_t pcnt_del_channel(pcnt_channel_handle_t chan);
esp_err_t pcnt_channel_set_edge_action(pcnt_channel_handle_t chan,
                pcnt_channel_edge_action_t pos_act, pcnt_channel_edge_action_t neg_act);
esp_err_t pcnt_channel_set_level_action(pcnt_channel_handle_t chan,
                pcnt_channel_level_action_t high_act, pcnt_channel_level_action_t low_act);

// --- Watch point / Event ---
esp_err_t pcnt_unit_add_watch_point(pcnt_unit_handle_t unit, int watch_point);    // ค่าต้องอยู่ใน (low,high)
esp_err_t pcnt_unit_remove_watch_point(pcnt_unit_handle_t unit, int watch_point);
esp_err_t pcnt_unit_register_event_callbacks(pcnt_unit_handle_t unit,
                const pcnt_event_callbacks_t *cbs, void *user_data);

// --- Control / Status ---
esp_err_t pcnt_unit_enable(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_disable(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_start(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_stop(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_clear_count(pcnt_unit_handle_t unit);
esp_err_t pcnt_unit_get_count(pcnt_unit_handle_t unit, int *value);
```

## §5. Patterns การใช้งาน
- **Quadrature encoder (AB-phase):** 2 phase ใน **channel เดียว** (edge = A, level = B) ตามตัวอย่าง §8 — ไม่ใช่ 2 channels
- **นับปุ่มกด (1 channel):** ตั้งแค่ `edge_gpio_num` (ใส่ `level_gpio_num = -1`) และ level action ทั้งคู่เป็น `KEEP`
- **นับความถี่สูงมาก (> ระดับ MHz):** PCNT โอเค แต่การกด `pcnt_unit_get_count()` บ่อยเกิน = สูญ CPU — ใช้ watch point + ISR แทน polling

## §6. VFS
- ไม่มี — PCNT ไม่มีการผูก VFS (N/A สำหรับไฟล์นี้)

## §7. กับดักที่พบบ่อย (⚠️ รอผูกรหัส HALLUCINATION-HOT-LIST ในรอบถัดไป)
1. **Legacy:** ห้ามใช้ `pcnt_config()` และ `pcnt_evt_t` ของเวอร์ชันเก่า (deprecated v5.0, ถูกลบใน v6)
2. ต้องเรียก `pcnt_unit_enable()` ก่อน `pcnt_unit_start()` เสมอ
3. `low_limit` ต้องติดลบ, `high_limit` ต้องเป็นบวก
4. หาก `accum_count = 0`: ชน high จะรีเซ็ตเป็น 0, ชน low จะค้างที่ค่านั้น
5. `on_reach` รันใน ISR — ห้าม `printf`; หากต้องการปลุก task ให้คืนค่า `true`
6. ปุ่มกดให้ตั้ง `level_gpio_num = -1`; Encoder AB-phase ต้องใช้ 2 phase ใน **1 channel**
7. เลี่ยงการ polling ด้วย `pcnt_unit_get_count()` บ่อยเกินไป ให้ใช้ watch point + ISR แทน
8. อย่าลืมตั้งค่า **glitch filter** หากใช้กับ encoder หรือสวิตช์หลวม — ไม่งั้นนับเพี้ยนจาก noise

## §8. ตัวอย่างมินิมัล (quadrature encoder)
```c
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

## §9. v6.x Notes
- โค้ด legacy `pcnt_config()` + `pcnt_evt_t` (`driver/pcnt.h` เวอร์ชันเก่า) — deprecated v5.0 **ถูกลบใน v6** → ใช้ new driver เท่านั้น

## §10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
> header เป้าหมาย (tag v6.1): `components/esp_driver_pcnt/include/driver/pcnt.h` (path ต้องยืนยัน)
1. สิ่งที่ต้องระบุใน CMakeLists `REQUIRES` ต่อเวอร์ชัน (v5.2: `driver` / v5.3+: `esp_driver_pcnt`) + path จริงของ header
2. `pcnt_unit_config_t` ฟิลด์เต็ม — `intr_priority` มีตั้งแต่เวอร์ชันไหน; `accum_count` พฤติกรรมจริง (นับเกิน limit แบบ int64?)
3. Glitch filter — driver รับ `max_glitch_ns` แต่ TRM บอก threshold เป็น APB cycles: ยืนยันการแปลงและขอบเขตค่า
4. Signature เต็มของ `on_reach` callback + `pcnt_watch_event_data_t.status` มีความหมายว่าอะไร
5. Watch point แบบ threshold 2 ค่า — ใช้ `pcnt_unit_add_watch_point()` ตัวเดียวรองรับทุกแบบ หรือมี API แยก
6. จำนวน units (4) + channels ต่อ unit (2) บน S3 — ยืนยันกับ soc_caps (`SOC_PCNT_UNITS_PER_GROUP` ฯลฯ)
7. Enum ค่าทั้งหมดของ `pcnt_channel_edge_action_t` / `pcnt_channel_level_action_t` vs ตัวอย่างใน §8
8. v5.2 ใช้ path/ชื่อ component อะไร
9. ผูกรหัส HALLUCINATION-HOT-LIST ให้ §7 (ทำพร้อมกับอัปเดต Hot-List)