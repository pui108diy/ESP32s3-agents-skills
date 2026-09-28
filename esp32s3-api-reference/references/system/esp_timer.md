# ESP Timer (High-Resolution Timer) — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "esp_timer.h"   // component: esp_timer (init อัตโนมัติตอนบูต — ไม่ต้องเรียก esp_timer_init เอง)
```

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 11 — SYSTIMER)
- **esp_timer สร้างบน SYSTIMER**: ตัวนับ **52-bit × 2 ตัว** (UNIT0/1) + **comparator 52-bit × 3 ตัว** (COMP0–2)
- นับด้วย CNT_CLK **~16 MHz** (สังเคราะห์จาก XTAL_CLK 40 MHz — ค่าเฉลี่ยใน 2 รอบนับ)
- Alarm 2 โหมด: **Target mode** (ครั้งเดียว) และ **Period mode** (เป็นรอบ)
- SYSTIMER เป็นแหล่งเวลาของระบบ — ระบบใช้ comparator บางตัวสร้าง **FreeRTOS tick** ด้วย ดังนั้น esp_timer กับ OS ใช้ฮาร์ดแวร์ชิ้นเดียวกัน

## 🎯 ภาพใหญ่: esp_timer vs gptimer vs FreeRTOS Software Timer

| | **esp_timer** | **gptimer** | **FreeRTOS Timer** |
|---|---|---|---|
| ฮาร์ดแวร์พื้นฐาน | SYSTIMER (52-bit, ~16 MHz) | TIMG0/1 (54-bit × 4 ตัว) | นับจาก FreeRTOS tick (ซึ่งก็มาจาก SYSTIMER) |
| ความละเอียด | **µs** | กำหนด `resolution_hz` เอง | เท่า tick (ms) — หยาบ |
| จำนวน timer | ไม่จำกัด (software list + heap) | **4 ตัว** บน S3 | ไม่จำกัด (heap) |
| Callback รันที่ | esp_timer task (default) หรือ ISR | **ISR เสมอ** | Timer Service task |
| Period ต่ำสุดที่ practical | ~50 µs (docs ระบุชัด) | ~5 µs | ~1 tick |
| เหมาะกับ | periodic งานทั่วไปหลายตัว, timeout, sampling | timing จำเพาะ, ETM, hardware event | periodic หยาบ ๆ ผูก RTOS |
| จุดอ่อน | ทุก callback แชร์ task เดียว — ตัวหนึ่งหน่วง ตัวอื่นเลื่อน | มีแค่ 4 ตัว | resolution หยาบ |

**กฎง่าย ๆ:** งาน "เรียก callback เป็นรอบ" ใช้ esp_timer; งาน "ต้อง sync กับฮาร์ดแวร์/ความแม่นระดับ ISR/ETM" ใช้ gptimer; งาน "delay แบบ ms ในโลก RTOS" ใช้ vTaskDelay/FreeRTOS timer

## Structs
```c
typedef void (*esp_timer_cb_t)(void *arg);   // ← void* ตัวเดียว

typedef struct {
    esp_timer_cb_t callback;
    void *arg;                               // ส่งเข้า callback
    esp_timer_dispatch_t dispatch_method;    // ESP_TIMER_TASK (default) / ESP_TIMER_ISR / ESP_TIMER_EXT ⚠️
    const char *name;                        // ใช้ใน esp_timer_dump()
    bool skip_unhandled_events;              // periodic: ข้าม event ที่ callback ไม่ทัน แทนการเข้าคิว
} esp_timer_create_args_t;
```

### Dispatch methods
| ค่า | Callback รันที่ | ข้อควรระวัง |
|---|---|---|
| `ESP_TIMER_TASK` (default) | **esp_timer task** (priority สูง) | ถูกหน่วงได้ถ้ามี task priority สูงกว่า เช่น ตอนเขียน flash |
| `ESP_TIMER_ISR` | **ตัว ISR โดยตรง** | ต้องเปิด `CONFIG_ESP_TIMER_SUPPORTS_ISR_DISPATCH_METHOD` (default ปิด); callback ต้อง IRAM-safe |
| `ESP_TIMER_EXT` ⚠️ | task แยกตาม level | มีใน IDF รุ่นใหม่ — ตรวจ `esp_timer.h` ของเวอร์ชันที่ใช้ |

## Function Signatures (exact)
```c
// --- สร้าง/ทำลาย ---
esp_err_t esp_timer_create(const esp_timer_create_args_t *create_args, esp_timer_handle_t *out_handle);
esp_err_t esp_timer_delete(esp_timer_handle_t timer);        // timer ต้องหยุดอยู่ก่อน

// --- เริ่ม/หยุด (หน่วยไมโครวินาทีทั้งหมด!) ---
esp_err_t esp_timer_start_once(esp_timer_handle_t timer, uint64_t timeout_us);
esp_err_t esp_timer_start_periodic(esp_timer_handle_t timer, uint64_t period_us);
esp_err_t esp_timer_stop(esp_timer_handle_t timer);
esp_err_t esp_timer_restart(esp_timer_handle_t timer, uint64_t timeout_us);   // เริ่มนับใหม่

// --- เวลาระบบ ---
int64_t  esp_timer_get_time(void);        // µs ตั้งแต่บูต — เรียกได้จาก ISR/task ไหนก็ได้
uint64_t esp_timer_get_next_alarm(void);  // v5+
esp_err_t esp_timer_dump(FILE *stream);   // สถิติ timer ทั้งหมด (ใช้ชื่อจาก .name)

// --- API รุ่นใหม่ (พบใน master/v6) ⚠️ ตรวจ header ก่อนใช้ ---
// esp_timer_start_periodic_at(timer, period_us, start_us);   // เริ่มตอนเวลาที่กำหนด
// esp_timer_restart_at(timer, timeout_us, time_us);
// esp_timer_start_once_at(timer, timeout_us, fire_time_us);
```

## กับดัก Hallucination
1. หน่วยเป็น **µs** — `esp_timer_start_periodic(t, 1000)` = 1 มิลลิวินาที ไม่ใช่ 1 วินาที
2. Periodic ที่สั้นกว่า **50 µs** ไม่ practical (กิน CPU ทั้งลูก) — งานละเอียดระดับนั้นใช้ gptimer/DMA
3. Default dispatch = **task ไม่ใช่ ISR** — แต่ทุก timer ในระบบ (รวมของ WiFi stack) แชร์ task เดียว: callback ยาว = หน่วง timer ตัวอื่นทั้งระบบ
4. Callback ควรสั้น ห้าม block; อย่าเรียก `esp_timer_delete()` กับ timer ตัวเองจากใน callback ของมัน
5. `esp_timer_get_time()` คืน `int64_t` → printf ด้วย `%lld` / `PRId64` และเทียบกับ `int64_t` เท่านั้น (อย่าเก็บใน `int32_t` — ล้นใน ~35 นาที)
6. ไม่ต้อง (และไม่ควร) เรียก `esp_timer_init()` — startup code ทำแล้ว
7. `esp_timer_start_*()` ซ้ำกับ timer ที่กำลังรันอยู่ → error (`ESP_ERR_INVALID_STATE`)
8. ความแม่นจริงขึ้นกับ load ของ esp_timer task — อย่าใช้เป็น PWM/สัญญาณ timing จำเพาะ

## ตัวอย่างมินิมัล (periodic + จับเวลา)
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