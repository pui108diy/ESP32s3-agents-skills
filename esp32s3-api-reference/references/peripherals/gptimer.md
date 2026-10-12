# gptimer (Timer Group) — v2.0

> ESP-IDF v5.2–v6.1 · ESP32-S3 · **สถานะ: ตรวจเชิงลึกกับ header v6.1 แล้ว (Phase 2) — เหลือคิวเล็กน้อยที่ §10**
> แหล่งยืนยัน: gptimer.h v6.1 (ไฟล์แนบ session นี้), TRM v1.8 ch.12 + POM + Migration Guide 5.x→6.x (ฐานความรู้), GitHub v5.2.1/master + docs.espressif.com (เว็บ)
> เปลี่ยนจาก v1.1: set/get_raw_count → uint64_t · ETM แยกหัวข้อ (header แยก gptimer_etm.h) · เพิ่ม get_resolution/get_captured_count · แก้ component เป็น esp_driver_gptimer · "~5 µs" ปรับเป็น heuristic · เพิ่ม Kconfig IRAM

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. S3 มี **2 timer groups (TIMG0/1) × 2 = 4 ตัว** (TRM ยืนยัน) — `gptimer_new_timer()` ตัวที่ 5 ได้ **`ESP_ERR_NOT_FOUND`** ไม่ใช่ NO_MEM
2. ตัวนับ **54-bit** นับขึ้น/ลง + auto-reload; prescaler 16-bit หาร 2–65536 (ตั้ง 1 จริง ๆ หาร 2 — TRM)
3. ลำดับ: `new_timer → register_event_callbacks → enable() → start()` — **register ครั้งแรกต้องก่อน enable** (header ระบุชัด)
4. `gptimer_start()` ได้ `ESP_ERR_INVALID_STATE` **ทั้ง**กรณี "ยังไม่ enabled" และ "กำลัง run อยู่แล้ว"
5. Callback `on_alarm` รันใน **ISR environment เสมอ** (header note ตรง ๆ) — ห้าม `printf`/`ESP_LOGx`/delay
6. Legacy driver (`driver/timer.h`, ชุด `timer_group_*`) deprecated v5.0 **ถูกลบใน v6** (migration guide ยืนยัน)
7. `gptimer_set_raw_count`/`gptimer_get_raw_count` ใช้ **`uint64_t`** (ยืนยัน header v6.1)
8. `resolution_hz` = tick ต่อวินาที — ไม่ใช่ความถี่ PWM; clock ไม่เสถียร (เช่น RC_FAST) ใช้ `gptimer_get_resolution()` ดูค่าจริงหลัง calibration
9. `gptimer_set_alarm_action(t, NULL)` = **ปิด** alarm; เรียกจาก ISR ได้; แก้ `edata->alarm_value` ใน callback ได้ทันที
10. ETM API อยู่ **header แยก `driver/gptimer_etm.h`** (gptimer.h ดึงมาให้แล้ว) — ตารางเทียบ esp_timer/gptimer/FreeRTOS Timer อยู่ esp_timer.md §5

## §1. Include & CMakeLists (ยืนยันแล้ว)
```c
#include "driver/gptimer.h"   // ดึงมาให้อัตโนมัติ: stdint.h, stdbool.h, esp_err.h,
                              // driver/gptimer_types.h (struct/enum/callback typedef),
                              // driver/gptimer_etm.h (ETM API)
```
- **CMakeLists `REQUIRES`:**
  - **v5.2:** `driver` (path จริง `components/driver/gptimer/include/driver/gptimer.h`)
  - **v5.3+ / v6.x:** **`esp_driver_gptimer`** (`components/esp_driver_gptimer/include/driver/gptimer.h`)
  - ⚠️ ไม่ใช่ `esp_driver_timer` (v1.1 ของไฟล์นี้เคยเขียนผิด — แก้แล้ว)

## §2. Hardware Facts (TRM ESP32-S3 v1.8 ch.12 — ยืนยันครบ + POM)
- **2 timer groups (TIMG0, TIMG1) × 2 GPTimer = 4 ตัว**; แต่ละ group มี Main System Watchdog แยก (ch.13 — อย่าปนกับ gptimer)
- ตัวนับ **54-bit time-base counter** (TB_CLK) นับขึ้น/ลงได้ + auto-reload-capable
- **Prescaler 16-bit หาร 2–65536** — TRM: ตั้งค่า 1 → ตัวหารจริง = 2; แก้ prescaler ตอน timer enabled → ผลลัพธ์ unpredictable (ต้อง disable ก่อน)
- Clock ต่อ timer: **APB_CLK (80 MHz) หรือ XTAL_CLK (40 MHz)** (ฟิลด์ TIMG_Tx_USE_XTAL)
- ⚠️ อย่าสับสน 3 ตัว: **gptimer** (TIMG) ≠ **esp_timer** (SYSTIMER) ≠ **FreeRTOS tick** (SYSTIMER)

## §3. Structs (ยืนยันทุกฟิลด์จาก gptimer.h v6.1)
```c
typedef struct {
    gptimer_clock_source_t clk_src;      // GPTIMER_CLK_SRC_DEFAULT / _APB / _XTAL (ค่า enum ใน gptimer_types.h — ดู §10)
    gptimer_count_direction_t direction; // GPTIMER_COUNT_UP / GPTIMER_COUNT_DOWN
    uint32_t resolution_hz;              // 1 tick = 1/resolution_hz วินาที
    int intr_priority;                   // 0 = driver จัดสรร priority ต่ำ (1,2,3) ให้เอง
    struct {
        uint32_t intr_shared: 1;         // แชร์หมายเลข interrupt กับ peripheral อื่นได้
        uint32_t allow_pd: 1;            // ✅ มีจริงใน v6.1 — อนุญาต power-down ตอน sleep
                                         //    (trade-off: กิน RAM เพิ่มเพื่อเก็บ register context)
    } flags;
} gptimer_config_t;

typedef struct {
    uint64_t alarm_count;                // ค่าตัวนับที่ trigger alarm
    uint64_t reload_count;               // ค่าโหลดกลับ — มีผลเมื่อ auto_reload เท่านั้น
    struct {
        uint32_t auto_reload_on_alarm: 1; // true = periodic, false = one-shot
    } flags;
} gptimer_alarm_config_t;

typedef struct {
    uint64_t count_value;                // ค่าตัวนับ ณ ตอน alarm (อ่านได้ใน callback)
    uint64_t alarm_value;                // ค่า alarm ที่ trigger — **แก้ไขได้ใน callback ทันที**
} gptimer_alarm_event_data_t;            // (นิยามใน gptimer_types.h)

typedef struct {
    gptimer_alarm_cb_t on_alarm;
} gptimer_event_callbacks_t;             // header note: callbacks รันใน ISR environment ทั้งหมด

typedef bool (*gptimer_alarm_cb_t)(gptimer_handle_t timer,
                                   const gptimer_alarm_event_data_t *edata, void *user_ctx);
// คืน true = มี high-priority task ถูกปลุก (ให้ RTOS ย้ายไปรันต่อทันที)
```

## §4. Alarm & Count API (ยืนยันลายเซ็นจาก header v6.1)
```c
// --- สร้าง/ทำลาย ---
esp_err_t gptimer_new_timer(const gptimer_config_t *config, gptimer_handle_t *ret_timer);
// คืน: ESP_OK / ESP_ERR_INVALID_ARG / ESP_ERR_NO_MEM / **ESP_ERR_NOT_FOUND** (timer หมด) / ESP_FAIL
// ตัวที่สร้างใหม่อยู่ state "init"
esp_err_t gptimer_del_timer(gptimer_handle_t timer);

// --- จ่ายไฟ/สถานะ ---
esp_err_t gptimer_enable(gptimer_handle_t timer);    // init → enable; ถ้า enabled อยู่แล้ว = ESP_ERR_INVALID_STATE
esp_err_t gptimer_disable(gptimer_handle_t timer);
esp_err_t gptimer_start(gptimer_handle_t timer);     // enable → run; ถ้าไม่ได้ enable หรือ run อยู่แล้ว = ESP_ERR_INVALID_STATE
esp_err_t gptimer_stop(gptimer_handle_t timer);      // run → enable

// --- ค่าตัวนับ (uint64_t ทั้งคู่!) ---
esp_err_t gptimer_set_raw_count(gptimer_handle_t timer, uint64_t value);
esp_err_t gptimer_get_raw_count(gptimer_handle_t timer, uint64_t *value);   // จะ trigger software capture
esp_err_t gptimer_get_captured_count(gptimer_handle_t timer, uint64_t *value);
// ↑ ต่างจาก get_raw_count: ไม่ trigger capture — คืนค่าที่ capture ไว้ล่าสุด (เช่นจาก ETM event ภายนอก)
//   ใช้ใน ISR ได้; อยู่ IRAM เมื่อเปิด CONFIG_GPTIMER_CTRL_FUNC_IN_IRAM
esp_err_t gptimer_get_resolution(gptimer_handle_t timer, uint32_t *out_resolution);
// ↑ clock ไม่เสถียร (เช่น RC_FAST) มี calibration → resolution จริงอาจต่างจากที่ config

// --- Alarm ---
esp_err_t gptimer_set_alarm_action(gptimer_handle_t timer, const gptimer_alarm_config_t *config);
// config = NULL → ปิด alarm; เรียกใน ISR context ได้ (อัปเดต alarm ใหม่ได้ทันทีใน callback ใด ๆ)
// ⚠️ เปิด CONFIG_GPTIMER_CTRL_FUNC_IN_IRAM แล้ว: gptimer_alarm_config_t ต้องอยู่ static data section
//    ห้ามเป็น const/read-only (เช่น static const) — header ระบุเอง

esp_err_t gptimer_register_event_callbacks(gptimer_handle_t timer,
                                           const gptimer_event_callbacks_t *cbs, void *user_data);
// ⚠️ ครั้งแรกต้องเรียก "ก่อน" gptimer_enable; ยกเลิก = ตั้งสมาชิก callback เป็น NULL แล้วเรียกซ้ำ
```

## §5. State Machine, Callbacks & ETM
```
new_timer → [init] → register_event_callbacks → enable() → [enable] → start() → [run]
                                    ↑ ต้องมาก่อน enable เสมอ (ครั้งแรก)
[run] → stop() → [enable] → disable() → [init] → del_timer()
```
- Error ตาม state: enable ซ้ำ → `ESP_ERR_INVALID_STATE` / start โดยไม่ enable **หรือ** ตอน run อยู่ → `ESP_ERR_INVALID_STATE`
- **กฎ callback:** รันใน ISR เสมอ · คืน `true` เมื่อปลุก high-priority task · แก้ `edata->alarm_value` ได้ (มีผลหลัง callback คืนค่า)
- **Kconfig:** `CONFIG_GPTIMER_ISR_CACHE_SAFE` (callback + ฟังก์ชันที่มันเรียกต้องอยู่ IRAM) · `CONFIG_GPTIMER_CTRL_FUNC_IN_IRAM` (ครอบ start/stop/set_raw_count/get_raw_count/get_captured_count/set_alarm_action — รันได้ตอน cache ปิด)
- **Power management:** `allow_pd = true` → อนุญาตให้ power domain ดับตอน sleep แต่กิน RAM เพิ่ม (context save)

### ETM (Event Task Matrix) — header แยก `driver/gptimer_etm.h`
```c
esp_err_t gptimer_new_etm_event(gptimer_handle_t timer, const gptimer_etm_event_config_t *config,
                                esp_etm_event_handle_t *out_event);
esp_err_t gptimer_new_etm_task(gptimer_handle_t timer, const gptimer_etm_task_config_t *config,
                               esp_etm_task_handle_t *out_task);
// gptimer_etm_event_config_t มีฟิลด์ gptimer_etm_event_type_t event_type (ค่า enum — §10)
// gptimer_etm_task_config_t  มีฟิลด์ gptimer_etm_task_type_t  task_type  (ค่า enum — §10)
```

## §6. VFS
- ไม่มี — gptimer ไม่มีการผูก VFS (N/A)

## §7. กับดักที่พบบ่อย (ผูก HALLUCINATION-HOT-LIST หมวด GPT — ดูรหัสจริงที่หน้า index)
- [COMPILE] GPTIMER-01: legacy `timer_group_init()`/`timer_isr_register()` (`driver/timer.h`) **ลบใน v6** → ใช้ gptimer (เดิม 19)
- [RUNTIME] GPTIMER-02: ต้อง `gptimer_enable()` ก่อน `gptimer_start()` — ข้าม = `ESP_ERR_INVALID_STATE`; `start()` ตอน run อยู่แล้วก็ error เดียวกัน — อย่าวินิจฉัยผิด (เดิม 20 · ขยาย)
- [COMPILE] GPTIMER-03: `gptimer_set_raw_count`/`gptimer_get_raw_count` เป็น **`uint64_t`** ไม่ใช่ `unsigned long`/`uint32_t` (ยืนยัน header v6.1)
- [COMPILE] GPTIMER-04: ETM API อยู่ **header แยก `driver/gptimer_etm.h`** (มาตั้งแต่ v5.2) + มีทั้ง event และ task
- [COMPILE] GPTIMER-05: CMakeLists — v5.2 = `driver` / v5.3+ = **`esp_driver_gptimer`** (ไม่ใช่ esp_driver_timer)
- [RUNTIME] GPTIMER-06: `gptimer_register_event_callbacks()` ครั้งแรกต้อง**ก่อน** `gptimer_enable()` — หลัง enable แล้ว = `ESP_ERR_INVALID_STATE` (header ระบุเอง)
- [RUNTIME] GPTIMER-07: `gptimer_new_timer()` ตัวที่ 5 → **`ESP_ERR_NOT_FOUND`** (S3 มี 4 ตัว) — อย่า assume NO_MEM
- [RUNTIME] GPTIMER-08: `auto_reload_on_alarm = false` → one-shot ยิงครั้งเดียวแล้วเงียบ — ต้อง set alarm ใหม่/แก้ `edata->alarm_value` ใน callback
- [SILENT] GPTIMER-09: "~5 µs ขั้นต่ำของระยะ alarm" = **heuristic จากประสบการณ์** — ไม่พบใน header/docs ทางการ ห้ามอ้างว่า docs ระบุ
- [SILENT] GPTIMER-10: เปิด `CONFIG_GPTIMER_CTRL_FUNC_IN_IRAM` แล้ววาง `gptimer_alarm_config_t` เป็น const/read-only → พังตอน cache ปิด — ต้อง `static` (ไม่ใช่ `static const`)

- `resolution_hz` ไม่ใช่ความถี่ PWM — คือ tick/วินาที (1 MHz → 1 tick = 1 µs)
-  `gptimer_set_alarm_action(timer, NULL)` = **ปิด** alarm ไม่ใช่ล้างค่าเดิม
- TRM: prescaler ตั้ง 1 → หารจริง 2; แก้ prescaler ตอน enabled = unpredictable (driver ครอบด้วย resolution_hz แต่ควรรู้)

## §8. ตัวอย่างมินิมัล (alarm ทุก 1 วินาที)
```c
#include "driver/gptimer.h"

static bool on_alarm_cb(gptimer_handle_t timer, const gptimer_alarm_event_data_t *edata, void *user_ctx)
{
    // ISR context: set flag / give semaphore เท่านั้น
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

    gptimer_alarm_config_t alarm_config = {  // ⚠️ ถ้าเปิด CTRL_FUNC_IN_IRAM: ต้อง static ห้าม const
        .alarm_count = 1000000,
        .reload_count = 0,
        .flags.auto_reload_on_alarm = true,
    };
    ESP_ERROR_CHECK(gptimer_set_alarm_action(gptimer, &alarm_config));

    gptimer_event_callbacks_t cbs = { .on_alarm = on_alarm_cb };
    ESP_ERROR_CHECK(gptimer_register_event_callbacks(gptimer, &cbs, NULL));  // ← ก่อน enable

    ESP_ERROR_CHECK(gptimer_enable(gptimer));
    ESP_ERROR_CHECK(gptimer_start(gptimer));
}
```

## §9. v6.x Notes (ยืนยันแล้ว)
- **Legacy `driver/timer.h` ถูกลบใน v6** (deprecated ตั้งแต่ v5.0) — migration guide ยืนยัน; โค้ดเก่าต้อง migrate มา gptimer
- **Component:** v5.2 = `driver` → v5.3+ = `esp_driver_gptimer` (ยืนยัน GitHub v5.2.1 + master + migration guide)
- **v6: driver headers ไม่ include FreeRTOS headers โดยนัย** (migration guide) — gptimer API เองไม่มีพารามิเตอร์ FreeRTOS type แต่ callback ที่เรียก `xQueueSendFromISR` ฯลฯ ต้อง include `freertos/FreeRTOS.h` + `freertos/queue.h` เอง
- `allow_pd` มีจริงใน v6.1 (ยืนยัน) — เวอร์ชันที่เริ่มมียังไม่ verified (§10)

## §10. จุดที่ยังต้องตรวจซ้ำ (เหลือน้อยแล้ว)
1. **`gptimer_types.h` v6.1 เนื้อแท้** — enum ค่าครบทุกตัว (`gptimer_clock_source_t` บน S3 มี DEFAULT/APB/XTAL จริงไหม, `gptimer_count_direction_t`, start/stop cmd) + ยืนยันฟิลด์ `gptimer_alarm_event_data_t` จาก v6.1 ตรง ๆ (ปัจจุบันยืนยันผ่าน v5.2.1 + docs แล้ว)
2. **`gptimer_etm.h` v6.1 เนื้อแท้** — ค่า enum `gptimer_etm_event_type_t`/`gptimer_etm_task_type_t` + config struct ฟิลด์เต็ม
3. Kconfig defaults ของ `CONFIG_GPTIMER_CTRL_FUNC_IN_IRAM` / `CONFIG_GPTIMER_ISR_CACHE_SAFE` ต่อเวอร์ชัน
4. `SOC_ETM_SUPPORTED` บน S3 — ยืนยันกับ soc_caps ว่า ETM ใช้ได้จริงบน S3
5. `allow_pd` เริ่มมีตั้งแต่เวอร์ชันใด (v1.1 เขียน v5.3+ — ยังไม่ verified)