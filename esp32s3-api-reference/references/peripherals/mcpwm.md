# MCPWM (Motor Control PWM) — New Driver — v1.1

> ESP-IDF v5.x/v6.x · ESP32-S3 · **สถานะ: Phase 1 แปลงโครงสร้าง §0–§10 แล้ว — ยังไม่ได้ตรวจเชิงลึกกับ header จริง (ดู §10)**
> เนื้อหาทั้งหมดย้ายจากไฟล์เดิม ไม่มีการเติมข้อมูลเทคนิคใหม่ที่ยังไม่ผ่านการตรวจ

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. โครงสร้างบน S3: **2 groups (MCPWM0/1) × 3 Timers × 3 Operators** + โมดูล Capture (CAP0–2) — capture ทำงาน**อิสระ**จาก operator (มี timer/channel ของตัวเอง)
2. Pipeline จำเป็น: `new_timer → new_operator → connect_timer → new_comparator → new_generator → set actions → timer_enable → timer_start_stop`
3. ลืม `mcpwm_operator_connect_timer()` → **ไม่มีสัญญาณ PWM ออกแม้โค้ด compile ผ่าน** (กับดักอันตรายที่สุดของ MCPWM)
4. ต้องเรียก `mcpwm_timer_enable()` ก่อน `mcpwm_timer_start_stop()` เสมอ — ข้าม = `ESP_ERR_INVALID_STATE`
5. `mcpwm_timer_start_stop()` รับ **enum** (`MCPWM_TIMER_START_NO_STOP` ฯลฯ) ห้ามส่ง `true/false`
6. `cmp_ticks` ต้อง **น้อยกว่า** `period_ticks` เสมอ
7. `group_id` ของทุก submodule (timer/operator/capture) ต้องเป็นค่าเดียวกัน
8. Callback `on_cap` รันใน **ISR context** — ห้าม `printf`
9. งาน 3-Phase: ใช้ **timer เดียว + operator 3 ตัว** (ใช้ timer 3 ตัว = สัญญาณไม่ sync กัน)
10. Legacy API (`mcpwm_init`, `mcpwm_set_duty`, `mcpwm_gpio_init` ฯลฯ) — deprecated v5.0 **ถูกลบใน v6** → ใช้ pipeline ใหม่เท่านั้น

## §1. Include & CMakeLists
```c
#include "driver/mcpwm.h"    // v5.3+/v6: component esp_driver_mcpwm (ชื่อ header เดิม)
```
- **CMakeLists:** ⚠️ ยังไม่ได้ตรวจว่าต้องระบุ `esp_driver_mcpwm` (หรือ `driver` ใน v5.2) ใน `REQUIRES` หรือถูกดึงมาแบบ transitively — อยู่ในคิวตรวจ §10

## §2. Hardware Facts (TRM ESP32-S3 v1.8 บทที่ 36 + POM)
- **2 กลุ่ม (MCPWM0 / MCPWM1)** แต่ละกลุ่มมี: **3 Timers, 3 Operators, โมดูล Capture (CAP0–2)**
- โมดูล Capture ทำงาน**อิสระจาก Operator** — มี timer และ channel ของตัวเอง (จับเวลาขอบสัญญาณภายนอกได้โดยไม่ผูกกับ PWM pipeline)

## §3. Structs & Driver Setup
```c
typedef struct {
    int group_id;                        // 0/1 (S3) — ต้องตรงกันทุก submodule!
    mcpwm_clock_source_t clk_src;        // MCPWM_CLK_SRC_DEFAULT
    uint32_t resolution_hz;              // เช่น 1000000 → 1 tick = 1 µs
    mcpwm_timer_count_mode_t count_mode; // MCPWM_TIMER_COUNT_MODE_UP / _DOWN / _UP_DOWN
    uint32_t period_ticks;               // คาบ PWM ในหน่วย tick
    struct { uint32_t update_period_on_empty: 1; uint32_t update_period_on_sync: 1; } flags;
} mcpwm_timer_config_t;

typedef struct { int group_id; } mcpwm_operator_config_t;   // + flags update_gen_action_on_tez/tep/sync ⚠️ (ยังไม่ verify — ดู §10)

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

## §4. Pipeline & Actions (ลำดับการใช้งาน — จำเป็น)
```
mcpwm_new_timer → mcpwm_new_operator → mcpwm_operator_connect_timer (ต้องระบุเพื่อเชื่อมโยง!)
→ mcpwm_new_comparator → mcpwm_new_generator → กำหนด Action → mcpwm_timer_enable → mcpwm_timer_start_stop
```

```c
// --- Timer ---
esp_err_t mcpwm_new_timer(const mcpwm_timer_config_t *config, mcpwm_timer_handle_t *ret_timer);
esp_err_t mcpwm_del_timer(mcpwm_timer_handle_t timer);
esp_err_t mcpwm_timer_enable(mcpwm_timer_handle_t timer);
esp_err_t mcpwm_timer_disable(mcpwm_timer_handle_t timer);
esp_err_t mcpwm_timer_start_stop(mcpwm_timer_handle_t timer, mcpwm_timer_start_stop_cmd_t command);

// --- Operator ---
esp_err_t mcpwm_new_operator(const mcpwm_operator_config_t *config, mcpwm_operator_handle_t *ret_operator);
esp_err_t mcpwm_del_operator(mcpwm_operator_handle_t operator);
esp_err_t mcpwm_operator_connect_timer(mcpwm_operator_handle_t operator, mcpwm_timer_handle_t timer);
esp_err_t mcpwm_operator_set_brake_mode(mcpwm_operator_handle_t operator, mcpwm_operator_brake_mode_t brake_mode);

// --- Comparator ---
esp_err_t mcpwm_new_comparator(mcpwm_operator_handle_t oper, const mcpwm_comparator_config_t *config,
                               mcpwm_cmpr_handle_t *ret_cmpr);
esp_err_t mcpwm_comparator_set_compare_value(mcpwm_cmpr_handle_t cmpr, uint32_t cmp_ticks);
esp_err_t mcpwm_del_comparator(mcpwm_cmpr_handle_t cmpr);

// --- Generator ---
esp_err_t mcpwm_new_generator(mcpwm_operator_handle_t oper, const mcpwm_generator_config_t *config,
                              mcpwm_gen_handle_t *ret_gen);
esp_err_t mcpwm_generator_set_action_on_timer_event(mcpwm_gen_handle_t gen, mcpwm_gen_timer_event_action_t action);
esp_err_t mcpwm_generator_set_actions_on_timer_event(mcpwm_gen_handle_t gen, ...);
esp_err_t mcpwm_generator_set_action_on_compare_event(mcpwm_gen_handle_t gen, mcpwm_gen_compare_event_action_t action);
esp_err_t mcpwm_generator_set_actions_on_compare_event(mcpwm_gen_handle_t gen, ...);
esp_err_t mcpwm_generator_set_force_level(mcpwm_gen_handle_t gen, int level, bool hold_on);
esp_err_t mcpwm_del_generator(mcpwm_gen_handle_t gen);
```

### Action Macros
```c
MCPWM_GEN_TIMER_EVENT_ACTION(timer_dir, timer_event, gen_action)
MCPWM_GEN_COMPARE_EVENT_ACTION(timer_dir, comparator, gen_action)
// timer_dir:  MCPWM_TIMER_DIRECTION_UP / _DOWN
// timer_event: MCPWM_TIMER_EVENT_EMPTY / _FULL
// gen_action: MCPWM_GEN_ACTION_KEEP / _LOW / _HIGH / _TOGGLE
```

## §5. Capture & Fault (Hardware Brake)
```c
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

// --- Fault (hardware brake) ---
esp_err_t mcpwm_new_gpio_fault(const mcpwm_gpio_fault_config_t *config, mcpwm_fault_handle_t *ret_fault);
esp_err_t mcpwm_generator_set_action_on_fault_event(mcpwm_gen_handle_t gen, mcpwm_gen_fault_event_action_t action);
esp_err_t mcpwm_soft_fault_activate(mcpwm_fault_handle_t fault);
```

## §6. VFS
- ไม่มี — MCPWM ไม่มีการผูก VFS (N/A สำหรับไฟล์นี้)

## §7. กับดักที่พบบ่อย (⚠️ รอผูกรหัส HALLUCINATION-HOT-LIST ในรอบถัดไป)
1. **API Legacy** (`mcpwm_gpio_init`, `mcpwm_init`, `mcpwm_set_duty`, `mcpwm_set_duty_in_us`) ถูกลบออกใน v6.0 → ต้องใช้ Pipeline ใหม่เท่านั้น
2. ข้าม `mcpwm_timer_enable()` ก่อน `mcpwm_timer_start_stop()` → `ESP_ERR_INVALID_STATE`
3. ลืม `mcpwm_operator_connect_timer()` → **ไม่มีสัญญาณ PWM ออกแม้โค้ดคอมไพล์ผ่าน**
4. `mcpwm_timer_start_stop` ต้องใช้ enum (เช่น `MCPWM_TIMER_START_NO_STOP`) ห้ามใช้ค่า `bool`
5. `cmp_ticks` ต้องน้อยกว่า `period_ticks` เสมอ
6. `group_id` ของทุกโมดูลที่เกี่ยวข้องต้องเป็นค่าเดียวกัน
7. Callback `on_cap` ทำงานใน ISR context — ห้ามใช้ `printf`
8. ต้องการ 3 phase → ใช้ **timer เดียว + operator 3 ตัว** (ไม่ใช่ timer 3 ตัว — จะไม่ sync กัน)

## §8. ตัวอย่างมินิมัล (servo 50 Hz ที่ GPIO 4)
```c
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

## §9. v6.x Notes
- **Legacy `driver/mcpwm.h` แบบเก่าถูกลบใน v6** — deprecated ตั้งแต่ v5.0 → ใช้ pipeline ใหม่เท่านั้น
- **Dead time:** IDF v5.2+ มี API dead-time เฉพาะ (`mcpwm_generator_set_dead_time` + config struct แยก); วิธี classic ที่เสถียรทุกเวอร์ชัน = ใช้ comparator 2 ตัวต่างกันเท่า dead time + generator คู่ A/B

## §10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
> header เป้าหมาย (tag v6.1): `components/esp_driver_mcpwm/include/driver/mcpwm.h` (path ต้องยืนยัน — ดูข้อ 1)
1. สิ่งที่ต้องระบุใน CMakeLists `REQUIRES` ต่อเวอร์ชัน (v5.2: `driver` / v5.3+: `esp_driver_mcpwm`) + path จริงของ header
2. `mcpwm_operator_config_t` — flags `update_gen_action_on_tez/tep/sync` มีจริงไหม (ไฟล์เดิม mark ⚠️ ไว้)
3. ⚠️ จาก migration guide v6 (ฐานความรู้ migration 5.x→6.x): `io_od_mode` ใน `mcpwm_generator_config_t` ถูกลบ → ต้องเรียก `gpio_od_enable()` เอง; `pull_up`/`pull_down` ถูกลบจาก config structures → ใช้ `gpio_set_pull_mode()` — ตรวจกับ header v6.1 แล้วเขียนลง §9
4. ⚠️ จาก migration guide v6 (ฐานความรู้): ฟังก์ชัน varg-style `mcpwm_generator_set_actions_on_timer_event` / `set_actions_on_compare_event` ถูกแทนด้วย helper structs/macros + dedicated setter functions — ตรวจว่า v6.1 ยังมีชุด varg อยู่ไหม ชื่อใหม่คืออะไร
5. Dead-time API (`mcpwm_generator_set_dead_time` + config struct) — ชื่อ/ลายเซ็นจริง เริ่มมีตั้งแต่เวอร์ชันใด
6. `mcpwm_capture_event_data_t` ฟิลด์เต็ม (`cap_edge`, `cap_value`) + signature เต็มของ `on_cap`
7. `mcpwm_operator_set_brake_mode` + enum `mcpwm_operator_brake_mode_t` ค่าทั้งหมด
8. จำนวน capture channels ต่อ group บน S3 + sync module API (มี `mcpwm_timer_set_sync_*` อีกไหม)
9. Struct ทุกฟิลด์เทียบ header v6.1 (โดยเฉพาะ flags ที่ยังไม่ verify)
10. ผูกรหัส HALLUCINATION-HOT-LIST ให้ §7 (ทำพร้อมกับอัปเดต Hot-List)