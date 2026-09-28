# MCPWM (Motor Control PWM) — New Driver — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "driver/mcpwm.h"    // v5.3+/v6: component esp_driver_mcpwm (ชื่อ header เดิม)
```

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 36 + POM)
- **2 groups: MCPWM0 / MCPWM1** → `group_id` = 0 หรือ 1
- แต่ละ group: prescaler 1 ตัว + **3 PWM timers + 3 PWM operators + capture module (CAP0–2)**
- แต่ละ operator: 2 comparators + 2 generators (PWMxA/PWMxB) + **dead-time แยก rising/falling อิสระ** + fault + carrier (modulate ด้วย high-frequency carrier สำหรับ gate driver แบบ isolated)
- Software override ผลลัพธ์แบบ asynchronous ได้; event ทุกตัว trigger interrupt ได้; timer sync ภายใน/ภายนอกได้
- **Capture เป็น submodule อิสระ** — ใช้ได้แม้ไม่มี operator (docs v6: "one dedicated timer and several independent channels", pulse ที่ GPIO → เก็บ timestamp + interrupt)

## Pipeline ของ new driver (v5.0+)
```
timer (time base) → operator (ผูกเข้า timer) → comparator (จุดเทียบค่า) → generator (event → level ที่ GPIO)
ลำดับเรียก: new_timer → new_operator → connect_timer → new_comparator → new_generator
            → set actions → mcpwm_timer_enable → mcpwm_timer_start_stop
```

## Structs (ฟิลด์ที่ใช้จริง)
```c
typedef struct {
    int group_id;                        // 0/1 (S3) — ต้องตรงกันทุก submodule!
    mcpwm_clock_source_t clk_src;        // MCPWM_CLK_SRC_DEFAULT
    uint32_t resolution_hz;              // เช่น 1000000 → 1 tick = 1 µs
    mcpwm_timer_count_mode_t count_mode; // MCPWM_TIMER_COUNT_MODE_UP / _DOWN / _UP_DOWN
    uint32_t period_ticks;               // คาบ PWM ในหน่วย tick
    struct { uint32_t update_period_on_empty: 1; uint32_t update_period_on_sync: 1; } flags;
} mcpwm_timer_config_t;

typedef struct { int group_id; } mcpwm_operator_config_t;   // + flags update_gen_action_on_tez/tep/sync ⚠️

typedef struct {
    struct { uint32_t update_cmp_on_tez: 1; uint32_t update_cmp_on_tep: 1; uint32_t update_cmp_on_sync: 1; } flags;
} mcpwm_comparator_config_t;

typedef struct { int gen_gpio_num; } mcpwm_generator_config_t;

typedef struct { int group_id; mcpwm_capture_clock_source_t clk_src; uint32_t resolution_hz; } mcpwm_capture_timer_config_t;
typedef struct {  // capture channel
    int gpio_num; int prescale;
    struct { uint32_t pos_edge: 1; uint32_t neg_edge: 1; uint32_t pull_up: 1; uint32_t pull_down: 1; } flags;
} mcpwm_capture_channel_config_t;
```

## Action Macros (สร้าง action struct + ปิดท้ายด้วย …_END ในตัว variadic)
```c
MCPWM_GEN_TIMER_EVENT_ACTION(timer_dir, timer_event, gen_action)
MCPWM_GEN_COMPARE_EVENT_ACTION(timer_dir, comparator, gen_action)
// timer_dir: MCPWM_TIMER_DIRECTION_UP / _DOWN
// timer_event: MCPWM_TIMER_EVENT_EMPTY / _FULL
// gen_action: MCPWM_GEN_ACTION_KEEP / _LOW / _HIGH / _TOGGLE
```

## Function Signatures (exact)
```c
// --- Timer ---
esp_err_t mcpwm_new_timer(const mcpwm_timer_config_t *config, mcpwm_timer_handle_t *ret_timer);
esp_err_t mcpwm_del_timer(mcpwm_timer_handle_t timer);
esp_err_t mcpwm_timer_enable(mcpwm_timer_handle_t timer);     // ← ต้องเรียกก่อน start เสมอ!
esp_err_t mcpwm_timer_disable(mcpwm_timer_handle_t timer);
esp_err_t mcpwm_timer_start_stop(mcpwm_timer_handle_t timer, mcpwm_timer_start_stop_cmd_t command);
// command ⚠️ enum ไม่ใช่ bool: MCPWM_TIMER_START_NO_STOP / MCPWM_TIMER_START_STOP_EMPTY / MCPWM_TIMER_START_STOP_FULL

// --- Operator ---
esp_err_t mcpwm_new_operator(const mcpwm_operator_config_t *config, mcpwm_operator_handle_t *ret_operator);
esp_err_t mcpwm_del_operator(mcpwm_operator_handle_t operator);
esp_err_t mcpwm_operator_connect_timer(mcpwm_operator_handle_t operator, mcpwm_timer_handle_t timer); // ← ลืม = ไม่มีสัญญาณ!
esp_err_t mcpwm_operator_set_brake_mode(mcpwm_operator_handle_t operator, mcpwm_operator_brake_mode_t brake_mode);
// MCPWM_OPER_BRAKE_MODE_CBC (cycle-by-cycle) / _OST (one-shot)

// --- Comparator ---
esp_err_t mcpwm_new_comparator(mcpwm_operator_handle_t oper, const mcpwm_comparator_config_t *config,
                               mcpwm_cmpr_handle_t *ret_cmpr);
esp_err_t mcpwm_comparator_set_compare_value(mcpwm_cmpr_handle_t cmpr, uint32_t cmp_ticks); // ต้อง < period_ticks!
esp_err_t mcpwm_del_comparator(mcpwm_cmpr_handle_t cmpr);

// --- Generator ---
esp_err_t mcpwm_new_generator(mcpwm_operator_handle_t oper, const mcpwm_generator_config_t *config,
                              mcpwm_gen_handle_t *ret_gen);
esp_err_t mcpwm_generator_set_action_on_timer_event(mcpwm_gen_handle_t gen, mcpwm_gen_timer_event_action_t action);
esp_err_t mcpwm_generator_set_actions_on_timer_event(mcpwm_gen_handle_t gen, ...);           // variadic + END
esp_err_t mcpwm_generator_set_action_on_compare_event(mcpwm_gen_handle_t gen, mcpwm_gen_compare_event_action_t action);
esp_err_t mcpwm_generator_set_actions_on_compare_event(mcpwm_gen_handle_t gen, ...);         // variadic + END
esp_err_t mcpwm_generator_set_force_level(mcpwm_gen_handle_t gen, int level, bool hold_on);
esp_err_t mcpwm_del_generator(mcpwm_gen_handle_t gen);

// --- Fault (hardware brake) ---
esp_err_t mcpwm_new_gpio_fault(const mcpwm_gpio_fault_config_t *config, mcpwm_fault_handle_t *ret_fault);
esp_err_t mcpwm_generator_set_action_on_fault_event(mcpwm_gen_handle_t gen, mcpwm_gen_fault_event_action_t action);
esp_err_t mcpwm_soft_fault_activate(mcpwm_fault_handle_t fault);

// --- Capture (อิสระจาก operator) ---
esp_err_t mcpwm_new_capture_timer(const mcpwm_capture_timer_config_t *config, mcpwm_capture_timer_handle_t *ret_cap_timer);
esp_err_t mcpwm_capture_timer_enable(mcpwm_capture_timer_handle_t cap_timer);
esp_err_t mcpwm_capture_timer_start(mcpwm_capture_timer_handle_t cap_timer);
esp_err_t mcpwm_new_capture_channel(mcpwm_capture_timer_handle_t cap_timer,
                                    const mcpwm_capture_channel_config_t *config,
                                    mcpwm_capture_channel_handle_t *ret_cap_channel);
esp_err_t mcpwm_capture_channel_register_event_callbacks(mcpwm_capture_channel_handle_t cap_channel,
                                                         const mcpwm_capture_event_callbacks_t *cbs, void *user_data);
// cbs.on_cap: bool (*)(ch, const mcpwm_capture_event_data_t *edata, void *ctx)  — ISR context!
// edata: { mcpwm_capture_edge_t cap_edge; uint32_t cap_value; }
```

## Version Marker
- Legacy `driver/mcpwm.h` แบบเก่า (`mcpwm_gpio_init`, `mcpwm_init`, `mcpwm_set_duty`, `mcpwm_set_duty_in_us`) — deprecated v5.0 **ถูกลบใน v6** → ใช้ pipeline ใหม่เท่านั้น
- **Dead time**: ⚠️ ตรวจ header `mcpwm_gen.h` / docs ของเวอร์ชันที่ใช้ — IDF รุ่น v5.2+ มี API dead-time เฉพาะ (`mcpwm_generator_set_dead_time` + config struct แยก); วิธี classic ที่เสถียรทุกเวอร์ชัน = ใช้ comparator 2 ตัวต่างกันเท่า dead time + generator คู่ A/B

## กับดัก Hallucination
1. โค้ด legacy `mcpwm_init()` + `mcpwm_set_duty()` → **v6 ลบแล้ว** → ต้อง new driver pipeline
2. ลืม `mcpwm_operator_connect_timer()` → operator ไม่มี timing source → **สัญญาณไม่ออกเลย** (compile ผ่าน!)
3. `mcpwm_timer_start_stop(timer, true)` → ผิด — พารามิเตอร์เป็น **enum** (`MCPWM_TIMER_START_NO_STOP` ฯลฯ)
4. ข้าม `mcpwm_timer_enable()` ก่อน `start_stop` → `ESP_ERR_INVALID_STATE`
5. `group_id` ไม่ตรงกันระหว่าง timer/operator/generator → error ตอนสร้าง
6. `cmp_ticks ≥ period_ticks` → สัญญาณผิดรูป/หาย (compare ต้อง **น้อยกว่า** period เสมอ)
7. Servo 50 Hz: `resolution_hz=1MHz, period_ticks=20000` (20 ms) — คิด period เป็นวินาทีตรง ๆ = เพี้ยน 1000 เท่า
8. `on_cap` callback รันใน ISR — ห้าม printf; คืน true เมื่อปลุก task
9. ต้องการ 3 phase → ใช้ timer เดียว + operator 3 ตัว (ไม่ใช่ timer 3 ตัว — จะไม่ sync กัน)

## ตัวอย่างมินิมัล (servo 50 Hz ที่ GPIO 4)
```c
#include "driver/mcpwm.h"

void servo_init(void)
{
    mcpwm_timer_handle_t timer = NULL;
    mcpwm_timer_config_t timer_config = {
        .group_id = 0,
        .clk_src = MCPWM_CLK_SRC_DEFAULT,
        .resolution_hz = 1000000,                       // 1 tick = 1 µs
        .count_mode = MCPWM_TIMER_COUNT_MODE_UP,
        .period_ticks = 20000,                          // 20 ms = 50 Hz
    };
    ESP_ERROR_CHECK(mcpwm_new_timer(&timer_config, &timer));

    mcpwm_operator_handle_t oper = NULL;
    ESP_ERROR_CHECK(mcpwm_new_operator(&(mcpwm_operator_config_t){ .group_id = 0 }, &oper));
    ESP_ERROR_CHECK(mcpwm_operator_connect_timer(oper, timer));       // ← ห้ามลืม

    mcpwm_cmpr_handle_t comparator = NULL;
    ESP_ERROR_CHECK(mcpwm_new_comparator(oper, &(mcpwm_comparator_config_t){ .flags.update_cmp_on_tez = true }, &comparator));
    ESP_ERROR_CHECK(mcpwm_comparator_set_compare_value(comparator, 1500));  // 1.5 ms = กลาง

    mcpwm_gen_handle_t generator = NULL;
    ESP_ERROR_CHECK(mcpwm_new_generator(oper, &(mcpwm_generator_config_t){ .gen_gpio_num = 4 }, &generator));
    ESP_ERROR_CHECK(mcpwm_generator_set_action_on_timer_event(generator,
        MCPWM_GEN_TIMER_EVENT_ACTION(MCPWM_TIMER_DIRECTION_UP, MCPWM_TIMER_EVENT_EMPTY, MCPWM_GEN_ACTION_HIGH)));
    ESP_ERROR_CHECK(mcpwm_generator_set_action_on_compare_event(generator,
        MCPWM_GEN_COMPARE_EVENT_ACTION(MCPWM_TIMER_DIRECTION_UP, comparator, MCPWM_GEN_ACTION_LOW)));

    ESP_ERROR_CHECK(mcpwm_timer_enable(timer));
    ESP_ERROR_CHECK(mcpwm_timer_start_stop(timer, MCPWM_TIMER_START_NO_STOP));   // ← enum!
}
```