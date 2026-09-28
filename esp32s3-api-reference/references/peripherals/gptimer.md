# GPTimer (Timer Group) — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "driver/gptimer.h"
```

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 12)
- **2 timer groups (TIMG0, TIMG1) × 2 timers = 4 ตัว** บน S3 — `gptimer_new_timer()` ได้สูงสุด 4 handles
- ตัวนับ **54 บิต** นับขึ้น/ลงได้ + auto-reload
- Prescaler 16 บิต (หารได้ 2–65536)
- คล็อกต่อ timer: **APB_CLK (80 MHz) หรือ XTAL_CLK (40 MHz)**
- แต่ละ group มี Main System Watchdog แยกต่างหาก (บทที่ 13 — อย่าปนกับ gptimer)
- ⚠️ อย่าสับสน 3 ตัว: **gptimer** (TIMG, เหตุการณ์ฮาร์ดแวร์) ≠ **esp_timer** (SYSTIMER, callback ระดับ µs) ≠ **FreeRTOS tick** (SYSTIMER)

## Structs
```c
typedef struct {
    gptimer_clock_source_t clk_src;      // GPTIMER_CLK_SRC_DEFAULT / GPTIMER_CLK_SRC_APB / GPTIMER_CLK_SRC_XTAL
    gptimer_count_direction_t direction; // GPTIMER_COUNT_UP / GPTIMER_COUNT_DOWN
    uint32_t resolution_hz;              // 1 tick = 1/resolution_hz วินาที
    int intr_priority;                   // 0 = default
    struct {
        uint32_t intr_shared: 1;
        uint32_t allow_pd: 1;            // v5.3+ (อนุญาต power-down ตอน sleep)
    } flags;
} gptimer_config_t;

typedef struct {
    uint64_t alarm_count;                // ค่าตัวนับที่ trigger alarm
    uint64_t reload_count;               // ค่าที่โหลดกลับเมื่อ alarm (ถ้า auto_reload)
    struct {
        uint32_t auto_reload_on_alarm: 1; // true = ทำงานเป็น periodic, false = one-shot
    } flags;
} gptimer_alarm_config_t;

typedef struct {
    uint64_t count_value;                // ค่าตัวนับ ณ ตอน alarm
    uint64_t alarm_value;                // ค่า alarm ที่ trigger
} gptimer_alarm_event_data_t;

typedef struct {
    gptimer_alarm_cb_t on_alarm;
} gptimer_event_callbacks_t;

typedef bool (*gptimer_alarm_cb_t)(gptimer_handle_t timer,
                                   const gptimer_alarm_event_data_t *edata, void *user_ctx);
// รันใน ISR context — return true เมื่อปลุก task ให้ reschedule
```

## Function Signatures (exact — ชุดเต็ม)
```c
esp_err_t gptimer_new_timer(const gptimer_config_t *config, gptimer_handle_t *ret_timer);
esp_err_t gptimer_del_timer(gptimer_handle_t timer);
esp_err_t gptimer_enable(gptimer_handle_t timer);     // ← ต้องเรียกก่อน start เสมอ!
esp_err_t gptimer_disable(gptimer_handle_t timer);
esp_err_t gptimer_start(gptimer_handle_t timer);      // เรียกเมื่อ timer ยังไม่ enabled = ESP_ERR_INVALID_STATE
esp_err_t gptimer_stop(gptimer_handle_t timer);
esp_err_t gptimer_set_raw_count(gptimer_handle_t timer, unsigned long value);
esp_err_t gptimer_get_raw_count(gptimer_handle_t timer, unsigned long *value);
esp_err_t gptimer_set_alarm_action(gptimer_handle_t timer, const gptimer_alarm_config_t *config);
// ส่ง NULL = ปิด alarm
esp_err_t gptimer_register_event_callbacks(gptimer_handle_t timer,
                                           const gptimer_event_callbacks_t *cbs, void *user_data);
// ETM (Event Task Matrix) — v5.x:
esp_err_t gptimer_new_etm_event(gptimer_handle_t timer, const gptimer_etm_event_config_t *config,
                                esp_etm_event_handle_t *out_event);
```

## State Machine (สำคัญ)
```
new_timer → [init] → enable() → [ready] → start() → [run] → stop() → [ready] → disable() → del_timer
```
- `enable`/`disable` ผูกกับ power management — callback ต้อง register ก่อน `enable()`

## กับดัก Hallucination
1. **โค้ด legacy** `timer_group_init()`, `timer_set_alarm_value()`, `timer_isr_register()` (`driver/timer.h`) — deprecated ตั้งแต่ v5.0 **และถูกลบใน v6** → ต้องใช้ gptimer
2. ข้าม `gptimer_enable()` แล้วเรียก `gptimer_start()` → `ESP_ERR_INVALID_STATE`
3. Callback `on_alarm` รันใน **ISR** — ห้าม `printf`/`ESP_LOGx`/delay; ทำงานสั้น ๆ แล้ว return
4. `auto_reload_on_alarm = false` → alarm ยิงครั้งเดียวแล้วเงียบ (ต้อง set alarm ใหม่) — ต้องการ periodic ให้ตั้ง `true`
5. ระยะ alarm = `|alarm_count - reload_count|` — docs แนะนำ **ไม่ควรสั้นกว่า 5 µs**
6. `resolution_hz` ไม่ใช่ความถี่ PWM — มันคือ "tick ต่อวินาที" (เช่น 1 MHz → 1 tick = 1 µs)
7. `gptimer_set_alarm_action(timer, NULL)` = **ปิด** alarm ไม่ใช่ล้างค่าเดิม
8. จำนวน gptimer บน S3 มีแค่ **4 ตัว** — งาน periodic จำนวนมากควรใช้ esp_timer แทน

## ตัวอย่างมินิมัล (alarm ทุก 1 วินาที)
```c
#include "driver/gptimer.h"

static bool on_alarm_cb(gptimer_handle_t timer, const gptimer_alarm_event_data_t *edata, void *user_ctx)
{
    // ISR context: toggle flag / give semaphore เท่านั้น
    return false;
}

void app_main(void)
{
    gptimer_handle_t gptimer = NULL;
    gptimer_config_t timer_config = {
        .clk_src = GPTIMER_CLK_SRC_DEFAULT,
        .direction = GPTIMER_COUNT_UP,
        .resolution_hz = 1000000,          // 1 tick = 1 µs
    };
    ESP_ERROR_CHECK(gptimer_new_timer(&timer_config, &gptimer));

    gptimer_alarm_config_t alarm_config = {
        .alarm_count = 1000000,            // 1,000,000 µs = 1 s
        .reload_count = 0,
        .flags.auto_reload_on_alarm = true, // periodic
    };
    ESP_ERROR_CHECK(gptimer_set_alarm_action(gptimer, &alarm_config));

    gptimer_event_callbacks_t cbs = { .on_alarm = on_alarm_cb };
    ESP_ERROR_CHECK(gptimer_register_event_callbacks(gptimer, &cbs, NULL));

    ESP_ERROR_CHECK(gptimer_enable(gptimer));   // ← ก่อน start เสมอ
    ESP_ERROR_CHECK(gptimer_start(gptimer));
}
```