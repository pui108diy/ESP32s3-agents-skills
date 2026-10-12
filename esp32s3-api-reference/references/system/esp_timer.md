# esp_timer (High-Resolution Timer) — v2.0
> ESP-IDF v5.2–v6.1 · ESP32-S3 · **สถานะ: ตรวจเชิงลึกกับ header v6.1 แล้ว (Phase 2) — เหลือคิวเล็กน้อยที่ §10**
> แหล่งยืนยัน: esp_timer.h v6.1 (ไฟล์แนบ session นี้), TRM v1.8 ch.11 + Migration Guide (ฐานความรู้), docs.espressif.com v5.2.4/v6.1 (เว็บ)
> เปลี่ยนจาก v1.1: ลบ ESP_TIMER_EXT (ไม่มีจริง) · get_next_alarm → int64_t · start_once_at ลายเซ็นจริง 2 พารามิเตอร์ · เพิ่ม API ใหม่ 7 ตัว · ยืนยัน CMake REQUIRES · ยืนยัน 50 µs/20 µs จาก docs

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. หน่วยเวลาของ `esp_timer_start_*()` ทั้งหมดเป็น **µs** — `1000000` = 1 วินาที ไม่ใช่ ms
2. สร้างบนฮาร์ดแวร์ **SYSTIMER** (52-bit, ~16 MHz = XTAL_CLK 40 MHz ÷ 2.5 เฉลี่ย) — resolution ระดับ µs
3. **init อัตโนมัติตอนบูต** — `esp_timer_init()` เรียกซ้ำได้ `ESP_ERR_INVALID_STATE`
4. Dispatch มีแค่ 2 แบบ: `ESP_TIMER_TASK` (default) / `ESP_TIMER_ISR` (ต้องเปิด `CONFIG_ESP_TIMER_SUPPORTS_ISR_DISPATCH_METHOD`) — **ไม่มี ESP_TIMER_EXT** (ยืนยัน v6.1)
5. จำนวน timer **ไม่จำกัด** (software list + heap) — ต่างจาก gptimer ที่ S3 มี 4 ตัว
6. **Periodic ต่ำกว่า 50 µs ไม่ practical** (กิน CPU ส่วนใหญ่) และ **one-shot สั้นกว่า 20 µs จะถูก delay เป็น ~20 µs** (docs ทางการระบุชัด)
7. `esp_timer_get_time()` / `esp_timer_get_next_alarm()` คืน **`int64_t`** — printf ด้วย `%lld`/`PRId64`; next_alarm คืน **`INT64_MAX`** ถ้าไม่มี timer ติดตั้ง
8. State อสมมาตร: `start_*()` ตอนรันอยู่ → `ESP_ERR_INVALID_STATE` / **`restart()` ตอนหยุดอยู่ → `ESP_ERR_INVALID_STATE` เช่นกัน**
9. `esp_timer_stop()` = **disarm เท่านั้น** ไม่หยุด callback ที่กำลังรัน — ต้องการ guarantee ใช้ `esp_timer_stop_blocking()` (v6.1) หรือ poll `esp_timer_is_active()` จนได้ false
10. กฎเลือกใช้: periodic งานทั่วไป → **esp_timer** / timing จำเพาะ + ETM → **gptimer** / หยาบระดับ tick → **FreeRTOS Timer** (ตารางเต็ม §5) — esp_timer เองมี ETM alarm event ผ่าน `esp_timer_new_etm_alarm_event()`

## §1. Include & CMakeLists (ยืนยันแล้ว)
```c
#include "esp_timer.h"   // ดึงมาให้อัตโนมัติ: stdint.h, stdio.h, stdbool.h,
                         // esp_err.h, esp_etm.h, sdkconfig.h (ไม่มี FreeRTOS header — ใช้ FreeRTOS ใน callback ต้อง include เอง)
```
- **CMakeLists (mandatory — ยืนยันจาก docs ทางการ):** ต้องประกาศ `REQUIRES esp_timer` หรือ `PRIV_REQUIRES esp_timer` — component อื่นที่ใช้ esp_timer แบบ PRIV (เช่น esp_event) **ไม่** propagate include path ให้ app → ไม่ประกาศ = `fatal error: esp_timer.h: No such file or directory`

## §2. Hardware Facts (TRM ESP32-S3 v1.8 ch.11 + docs)
- **esp_timer สร้างบน SYSTIMER**: ตัวนับ **52-bit × 2 ตัว** (UNIT0/1) + **comparator 52-bit × 3 ตัว** (COMP0–2)
- นับด้วย CNT_CLK **~16 MHz** — สังเคราะห์จาก XTAL_CLK 40 MHz ÷ **2.5** (ค่าเฉลี่ย — fractional division)
- Alarm 2 โหมด: **Target mode** (ครั้งเดียว) และ **Period mode** (เป็นรอบ)
- SYSTIMER เป็นแหล่งเวลาของระบบ — FreeRTOS tick ก็มาจาก comparator ตัวหนึ่งของ SYSTIMER ดังนั้น esp_timer กับ OS ใช้ฮาร์ดแวร์ชิ้นเดียวกัน
- Hardware timer interrupt priority ตั้งผ่าน `CONFIG_ESP_TIMER_INTERRUPT_LEVEL` (ค่า 1–3) — จาก docs (ที่มา v5.2 ⚠️ รอ verify ชื่อใน Kconfig v6.1 — §10)

## §3. Structs & Dispatch (ยืนยันทุกฟิลด์จาก header v6.1)
```c
typedef struct esp_timer* esp_timer_handle_t;   // opaque
typedef void (*esp_timer_cb_t)(void* arg);

typedef enum {
    ESP_TIMER_TASK,     // Callback รันใน esp_timer task (default)
#if CONFIG_ESP_TIMER_SUPPORTS_ISR_DISPATCH_METHOD || __DOXYGEN__
    ESP_TIMER_ISR,      // Callback รันใน ISR — ต้องเปิด Kconfig ก่อน (ปิดอยู่ = enum นี้ไม่ถูกคอมไพล์!)
#endif
    ESP_TIMER_MAX,      // sentinel
} esp_timer_dispatch_t;                         // ⚠️ ไม่มี ESP_TIMER_EXT — ยืนยันแล้ว

typedef struct {
    esp_timer_cb_t callback;          // callback เมื่อ timeout
    void* arg;                        // ส่งเข้า callback
    esp_timer_dispatch_t dispatch_method; // ESP_TIMER_TASK (default) / ESP_TIMER_ISR
    const char* name;                 // ใช้ใน esp_timer_dump()
    bool skip_unhandled_events;       // ข้าม event ที่ไม่ได้ handle **ใน light-sleep** สำหรับ periodic
} esp_timer_create_args_t;
```

### Dispatch methods
| ค่า | Callback รันที่ | ข้อควรระวัง |
|---|---|---|
| `ESP_TIMER_TASK` (default) | esp_timer task (priority สูง) | ถูกหน่วงได้ถ้ามี task priority สูงกว่า เช่น ตอนเขียน flash |
| `ESP_TIMER_ISR` | ISR โดยตรง | ต้องเปิด `CONFIG_ESP_TIMER_SUPPORTS_ISR_DISPATCH_METHOD`; กฎจาก header: **ห้าม block, ห้ามเรียก FreeRTOS ที่ไม่ลงท้าย FromISR**; ปลุก task แล้วต้องการ yield → เรียก `esp_timer_isr_dispatch_need_yield()` (ไม่ใช่ return bool เหมือน gptimer!) |

```c
esp_err_t esp_timer_create(const esp_timer_create_args_t* create_args, esp_timer_handle_t* out_handle);
esp_err_t esp_timer_delete(esp_timer_handle_t timer);   // timer ต้องหยุดอยู่ก่อน
```

## §4. Start/Stop & System Time API (ยืนยันลายเซ็นจาก header v6.1)
```c
// --- init ระบบ (startup code ทำให้ — ห้ามเรียกเอง) ---
esp_err_t esp_timer_early_init(void);   // internal
esp_err_t esp_timer_init(void);         // เรียกซ้ำ = ESP_ERR_INVALID_STATE
esp_err_t esp_timer_deinit(void);       // ปกติ app ไม่ควรเรียก

// --- เริ่ม/หยุด (หน่วย µs ทั้งหมด) ---
esp_err_t esp_timer_start_once(esp_timer_handle_t timer, uint64_t timeout_us);
//    INVALID_STATE ถ้า timer กำลังรัน
esp_err_t esp_timer_start_periodic(esp_timer_handle_t timer, uint64_t period);
esp_err_t esp_timer_restart(esp_timer_handle_t timer, uint64_t timeout_us);
//    INVALID_STATE ถ้า timer "ไม่ได้" รันอยู่ (ตรงข้ามกับ start!)
esp_err_t esp_timer_stop(esp_timer_handle_t timer);              // disarm เท่านั้น — ไม่ preempt callback ที่รันอยู่
esp_err_t esp_timer_stop_blocking(esp_timer_handle_t timer, uint32_t timeout_ticks);   // v6.1: รอจนแน่ใจว่า callback จบ

// --- กลุ่ม _at (absolute time — ยืนยันใน v6.1; เวอร์ชันแรกที่มา §10) ---
esp_err_t esp_timer_start_once_at(esp_timer_handle_t timer, uint64_t alarm_us);
//    ⚠️ 2 พารามิเตอร์เท่านั้น! INVALID_ARG ถ้า alarm_us อยู่ในอดีต; INVALID_STATE ถ้ากำลังรัน
esp_err_t esp_timer_start_periodic_at(esp_timer_handle_t timer, uint64_t period_us, uint64_t first_alarm_us);
esp_err_t esp_timer_restart_at(esp_timer_handle_t timer, uint64_t period_us, uint64_t first_alarm_us);
//    restart_at: INVALID_STATE ถ้า "ไม่ได้" รันอยู่

// --- เวลาระบบ / ตรวจสอบ ---
int64_t  esp_timer_get_time(void);                     // µs ตั้งแต่บูต — เรียกได้จาก ISR/task ไหนก็ได้
int64_t  esp_timer_get_next_alarm(void);               // ครอบทุก dispatch method; INT64_MAX ถ้าไม่มี timer armed
int64_t  esp_timer_get_next_alarm_for_wake_up(void);   // ตัด timer ที่เปิด skip_unhandled_events (ใช้โดย power management)
esp_err_t esp_timer_get_period(esp_timer_handle_t timer, uint64_t *period);
esp_err_t esp_timer_get_expiry_time(esp_timer_handle_t timer, uint64_t *expiry);
bool      esp_timer_is_active(esp_timer_handle_t timer);
esp_err_t esp_timer_dump(FILE* stream);
//    ไม่เปิด CONFIG_ESP_TIMER_PROFILING = พิมพ์เฉพาะ timer ที่ armed; เปิด = ทุกตัว; ESP_ERR_NO_MEM ได้

// --- ISR dispatch helper + ETM ---
void esp_timer_isr_dispatch_need_yield(void);          // เรียกจาก ISR-dispatch callback เมื่อปลุก task priority สูง
esp_err_t esp_timer_new_etm_alarm_event(esp_etm_event_handle_t *out_event);   // esp_timer alarm → ETM event
```

## §5. ภาพใหญ่: esp_timer vs gptimer vs FreeRTOS Software Timer
| | **esp_timer** | **gptimer** | **FreeRTOS Timer** |
|---|---|---|---|
| ฮาร์ดแวร์พื้นฐาน | SYSTIMER (52-bit, ~16 MHz) | TIMG0/1 (54-bit × 4 ตัว) | FreeRTOS tick (จาก SYSTIMER) |
| ความละเอียด | **µs** | กำหนด `resolution_hz` เอง | เท่า tick (ms) — หยาบ |
| จำนวน timer | ไม่จำกัด (software list + heap) | **4 ตัว** บน S3 | ไม่จำกัด (heap) |
| Callback รันที่ | esp_timer task (default) หรือ ISR | **ISR เสมอ** | Timer Service task |
| กลไก yield ใน ISR | เรียก `esp_timer_isr_dispatch_need_yield()` | คืนค่า `bool` จาก callback | N/A |
| Period ต่ำสุดที่ practical | **50 µs periodic / ~20 µs one-shot** (docs ระบุชัด) | ~5 µs (heuristic) | ~1 tick |
| ETM | alarm event ได้ (`esp_timer_new_etm_alarm_event`) | event + task ได้ | ไม่มี |
| เหมาะกับ | periodic งานทั่วไปหลายตัว, timeout, sampling | timing จำเพาะ, ETM, hardware event | periodic หยาบ ๆ ผูก RTOS |
| จุดอ่อน | ทุก callback แชร์ task เดียว — ตัวหนึ่งหน่วง ตัวอื่นเลื่อน | มีแค่ 4 ตัว | resolution หยาบ |

**กฎง่าย ๆ:** งาน "เรียก callback เป็นรอบ" ใช้ esp_timer; งาน "sync กับฮาร์ดแวร์/ความแมนระดับ ISR/ETM ครบทั้ง event+task" ใช้ gptimer; งาน "delay แบบ ms ในโลก RTOS" ใช้ vTaskDelay/FreeRTOS timer

## §6. VFS
- ไม่มี — esp_timer ไม่มีการผูก VFS (N/A)

## §7. กับดักที่พบบ่อย (ผูก HALLUCINATION-HOT-LIST หมวด ESP_TIMER)
- [RUNTIME] ESP_TIMER-01: หน่วยเป็น **µs** — 1 วินาที = 1000000 (เดิม 25)
- [SILENT] ESP_TIMER-02: callback ทุกตัว (รวม WiFi stack) แชร์ esp_timer task เดียว — ตัวหนึ่งหน่วง ทั้งระบบเลื่อน (เดิม 26)
- [SILENT] ESP_TIMER-03: `esp_timer_get_time()` ต้องเก็บ `int64_t` — int32 ล้นใน ~35 นาที (เดิม 27)
- [COMPILE] ESP_TIMER-04: **`ESP_TIMER_EXT` ไม่มีอยู่จริง** — enum มีแค่ TASK/ISR (+MAX sentinel) ยืนยัน header v6.1
- [COMPILE] ESP_TIMER-05: `esp_timer_get_next_alarm()` คืน **`int64_t`** และ **`INT64_MAX`** ถ้าไม่มี timer armed — เทียบ `!= INT64_MAX` อย่าเทียบ `< 0`
- [COMPILE] ESP_TIMER-06: ลืม `REQUIRES esp_timer` ใน CMakeLists → `esp_timer.h: No such file or directory` (mandatory)
- [RUNTIME] ESP_TIMER-07: state อสมมาตร — `start_*()` ตอนรันอยู่ = INVALID_STATE / `restart()` ตอนหยุดอยู่ = INVALID_STATE เช่นกัน
- [RUNTIME] ESP_TIMER-08: `esp_timer_stop()` แค่ disarm ไม่ preempt callback ที่กำลังรัน → ต้องการ guarantee ใช้ `stop_blocking()` (v6.1) หรือ poll `is_active()`
- [RUNTIME] ESP_TIMER-09: ISR dispatch — ห้าม block, FreeRTOS เฉพาะ `*FromISR`; yield ด้วย `esp_timer_isr_dispatch_need_yield()` (ไม่ใช่ return bool แบบ gptimer)
- [RUNTIME] ESP_TIMER-10: `skip_unhandled_events` ทำงานเฉพาะ **light-sleep** — ไม่ใช่กลไก "callback ไม่ทัน" ตอนปกติ
- [SILENT] ESP_TIMER-11: periodic < **50 µs** กิน CPU ส่วนใหญ่ / one-shot < **20 µs** ถูก delay เป็น ~20 µs (docs ทางการระบุ)
- [RUNTIME] ESP_TIMER-12: ห้าม `esp_timer_delete()` ตัวเองจากใน callback ของมัน; callback ควรสั้น


## §8. ตัวอย่างมินิมัล (periodic + จับเวลา)
```c
#include "esp_timer.h"

static void periodic_cb(void *arg) { /* สั้น ๆ เท่านั้น: set flag/give semaphore */ }

void app_main(void)
{
    const esp_timer_create_args_t t_args = {
        .callback = &periodic_cb,
        .name = "sample_1s",
    };
    esp_timer_handle_t t;
    ESP_ERROR_CHECK(esp_timer_create(&t_args, &t));
    ESP_ERROR_CHECK(esp_timer_start_periodic(t, 1000000ULL));   // ← µs: 1 วินาที

    int64_t t0 = esp_timer_get_time();
    vTaskDelay(pdMS_TO_TICKS(100));
    ESP_LOGI("timer", "elapsed = %lld us", esp_timer_get_time() - t0);
}
```

## §9. v6.x Notes (ยืนยันแล้ว)
- **กลุ่ม `_at` ทั้ง 4 ตัว** (`start_once_at`, `start_periodic_at`, `restart_at`) มีจริงใน v6.1 — ลายเซ็นตรงตาม §4 (เวอร์ชันแรกที่เพิ่มยังไม่ verified — §10)
- **v6.1 มี:** `esp_timer_stop_blocking()` · `esp_timer_is_active()` · `esp_timer_get_period()` · `esp_timer_get_expiry_time()` · `esp_timer_get_next_alarm_for_wake_up()` (DeepSeek ว่าไม่มีใน v6.0 — ยังไม่ verify ด้วย header v6.0 โดยตรง)
- `ESP_TIMER_ISR` ถูกคอมไพล์เฉพาะเมื่อเปิด `CONFIG_ESP_TIMER_SUPPORTS_ISR_DISPATCH_METHOD` — ปิดอยู่แล้วใช้ค่านี้ = compile error
- `esp_timer_dump()` พฤติกรรมตาม `CONFIG_ESP_TIMER_PROFILING` (armed-only vs all)
- esp_timer.h ไม่ include FreeRTOS header ใด ๆ — callback ที่ใช้ FreeRTOS API ต้อง include เอง (สอดคล้องนโยบาย v6 เรื่อง implicit includes)
- **ETM:** `esp_timer_new_etm_alarm_event()` ทำ alarm ให้เป็น ETM event ได้ (header ดึง `esp_etm.h` มาด้วย)

## §10. จุดที่ยังต้องตรวจซ้ำ (เหลือน้อยแล้ว)
1. เวอร์ชันแรกของกลุ่ม `_at` / `stop_blocking` / `is_active` / `get_period` / `get_expiry_time` / `get_next_alarm_for_wake_up` (สอบ header v5.3/v5.4/v6.0 หรือ docs แต่ละเวอร์ชัน)
2. `CONFIG_ESP_TIMER_INTERRUPT_LEVEL` — ยืนยันว่ายังมีชื่อนี้ใน Kconfig v6.1 (ที่มาปัจจุบัน: docs v5.2)
3. `CONFIG_ESP_TIMER_PROFILING` — output fields เต็มของ dump
4. `CONFIG_ESP_TIMER_SUPPORTS_ISR_DISPATCH_METHOD` default ต่อเวอร์ชัน (v5.2/v5.3/v6.x)
5. ETM: `esp_etm.h` อยู่ component ใด (เพื่อ CMake REQUIRES เมื่อเรียก `esp_timer_new_etm_alarm_event`) + `SOC_ETM_SUPPORTED` บน S3
