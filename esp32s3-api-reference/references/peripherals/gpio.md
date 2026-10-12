# GPIO — v2.2
> Target: ESP32-S3 | ESP-IDF v5.2 – v6.1
> อ้างอิง: TRM v1.8 ch.5–6 (GPIO/IO MUX) + ch.10 (Low-power) | Datasheet v2.2 | driver/gpio.h + driver/rtc_io.h + esp_sleep.h
> เปลี่ยนแปลง v2.2: แปลงโครงสร้างตาม skill v2.7 (§0 TL;DR + §10 จุดตรวจซ้ำ + ผูกรหัส Hot-List) · เพิ่ม CMakeLists `esp_driver_gpio` · แยก §5 ISR Pattern / §6 Deep-sleep Hold & Wake-cause

## 0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. **ไม่มี GPIO 22–25** — เลขวิ่ง 0–21 แล้วกระโดดไป 26–48 (รวม 45 ขา)
2. **GPIO 26–32 ห้ามใช้เสมอ** (SPI0/1 flash/PSRAM) · **33–37 ห้ามเฉพาะโมดูล Octal Flash/PSRAM** (N8R8/N16R8/R8V) — Quad-only (N8, N8R2) ใช้ได้
3. **RTC GPIO = 0–21 เท่านั้น** — กลุ่มเดียวที่ปลุก Deep-sleep (ext0/ext1) ได้
4. `gpio_wakeup_enable()` = **Light-sleep เท่านั้น** (+ `esp_sleep_enable_gpio_wakeup()`)
5. `gpio_isr_t` = `void (*)(void *arg)` — ส่งเลข pin ผ่าน `args`; mask ใช้ `1ULL << n`
6. ห้าม `printf`/`ESP_LOGx` ใน ISR — set flag หรือ `xQueueSendFromISR()` เท่านั้น
7. คง digital GPIO ข้าม Deep-sleep: `gpio_hold_en()` **+** `gpio_deep_sleep_hold_en()` (RTC GPIO ใช้ `rtc_gpio_hold_en()`)
8. ปลด hold หลังตื่น: ตั้ง direction+level ให้ได้ระดับเดิม**ก่อน** `gpio_hold_dis()` — สลับลำดับ = ระดับกระโดด
9. `gpio_pad_select_gpio()` ลบใน v6 → `gpio_reset_pin()`
10. v6.0+: `esp_sleep_get_wakeup_causes()` คืน **bitmap** (ตัวเดิม deprecated); ext1 ใช้ variant `_io`

## 1. Include & CMakeLists

```c
#include "driver/gpio.h"     // gpio_num_t + API หลัก
#include "driver/rtc_io.h"   // RTC GPIO (ขา 0–21): rtc_gpio_init / rtc_gpio_hold_en
#include "esp_sleep.h"       // sleep wakeup: esp_sleep_enable_gpio_wakeup / ext0 / ext1 / get_wakeup_cause
```

```cmake
# v5.2:        REQUIRES driver
# v5.3+ / v6.x: แยก component — ระบุตรง:
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_driver_gpio)
```

## 2. Hardware Facts (TRM v1.8 + Datasheet v2.2)
- GPIO ใช้ได้จริง: **0–21 และ 26–48** (45 ขา) — **ไม่มี GPIO 22–25** (เลขกระโดด 21 → 26)
- **GPIO 0–21 = RTC GPIO** (โดเมน VDD3P3_RTC) — กลุ่มเดียวที่คง output และปลุกได้ใน Deep-sleep
- **GPIO 26–32**: SPI0/1 ต่อ flash/PSRAM ในชิป (SPICS1, SPIHD, SPIWP, SPICS0, SPICLK, SPIQ, SPID) — ห้ามใช้เด็ดขาด
- **GPIO 33–37**: กลายเป็น SPIIO4–7 + SPIDQS เมื่อใช้ **Octal Flash หรือ Octal PSRAM** — ห้ามใช้เฉพาะกรณีนั้น; โมดูล Quad-only (N8, N8R2) ใช้เป็น GPIO ได้ปกติ
- Strapping pins (ระวัง pull ตอนบูต): **GPIO0, GPIO45 (VDD_SPI), GPIO46 (boot mode + ROM log)**
- **GPIO3 (JTAG source)**: มีผลเฉพาะเมื่อ EFUSE_STRAP_JTAG_SEL=1 — default คือ **ignore** (JTAG วิ่งผ่าน USB_SERIAL_JTAG เสมอ); **ขานี้ไม่มี internal pull — ห้ามลอย**
- USB-Serial-JTAG: **GPIO19 (D-), GPIO20 (D+)** | JTAG ภายนอก: **GPIO39–42** | UART0: **GPIO43 (TX), GPIO44 (RX)**
- Drive strength (IO_MUX_MCU_DRV): 0 = ~5 mA, 1 = ~10 mA, 2 = ~20 mA (default), 3 = ~40 mA — **GPIO17/18 ลำดับพิเศษ: 1 = ~20 mA, 2 = ~10 mA**
- บอร์ด: DevKitC-1 (WROOM-1) **ไม่มี GPIO26, 33, 34 บน header**; DevKitM-1 (MINI-1) ดึงออกครบ 26/33–37

## 3. Structs & Enums
```c
typedef struct {
    uint64_t pin_bit_mask;        // หลาย pin: (1ULL<<4) | (1ULL<<5)
    gpio_mode_t mode;
    gpio_pullup_t pull_up_en;     // GPIO_PULLUP_ENABLE / DISABLE
    gpio_pulldown_t pull_down_en; // GPIO_PULLDOWN_ENABLE / DISABLE
    gpio_int_type_t intr_type;    // GPIO_INTR_DISABLE/POSEDGE/NEGEDGE/ANYEDGE/LOW_LEVEL/HIGH_LEVEL
} gpio_config_t;

typedef enum {
    GPIO_DRIVE_CAP_0 = 0,  // ~5 mA
    GPIO_DRIVE_CAP_1 = 1,  // ~10 mA (GPIO17/18: ~20 mA)
    GPIO_DRIVE_CAP_2 = 2,  // ~20 mA — default (GPIO17/18: ~10 mA)
    GPIO_DRIVE_CAP_3 = 3,  // ~40 mA
} gpio_drive_cap_t;
```

## 4. Function Signatures (exact)
```c
// ===== Core =====
esp_err_t gpio_config(const gpio_config_t *pGPIOConfig);
esp_err_t gpio_reset_pin(gpio_num_t gpio_num);            // ← แทน gpio_pad_select_gpio (deprecated v5.0, removed v6)
esp_err_t gpio_set_level(gpio_num_t gpio_num, uint32_t level);
int       gpio_get_level(gpio_num_t gpio_num);
esp_err_t gpio_set_direction(gpio_num_t gpio_num, gpio_mode_t mode);
esp_err_t gpio_input_enable(gpio_num_t gpio_num);
esp_err_t gpio_set_pull_mode(gpio_num_t gpio_num, gpio_pull_mode_t pull);  // enum คนละชุดกับ gpio_config_t!
esp_err_t gpio_set_drive_capability(gpio_num_t gpio_num, gpio_drive_cap_t strength);
esp_err_t gpio_get_drive_capability(gpio_num_t gpio_num, gpio_drive_cap_t *strength);

// ===== Interrupt =====
esp_err_t gpio_set_intr_type(gpio_num_t gpio_num, gpio_int_type_t intr_type);
esp_err_t gpio_install_isr_service(int intr_alloc_flags); // ESP_INTR_FLAG_IRAM ถ้า ISR อยู่ใน IRAM
esp_err_t gpio_isr_handler_add(gpio_num_t gpio_num, gpio_isr_t isr_handler, void *args);
esp_err_t gpio_isr_handler_remove(gpio_num_t gpio_num);
esp_err_t gpio_intr_enable(gpio_num_t gpio_num);
esp_err_t gpio_intr_disable(gpio_num_t gpio_num);
typedef void (*gpio_isr_t)(void *arg);                    // ← รับ void* ตัวเดียวเท่านั้น!

// ===== Wakeup (Light-sleep) =====
esp_err_t gpio_wakeup_enable(gpio_num_t gpio_num, gpio_int_type_t intr_type); // Light-sleep เท่านั้น!
esp_err_t gpio_wakeup_disable(gpio_num_t gpio_num);

// ===== Hold =====
esp_err_t gpio_hold_en(gpio_num_t gpio_num);              // ล็อกขาข้าม soft reset + ระหว่าง Light-sleep
esp_err_t gpio_hold_dis(gpio_num_t gpio_num);             // หลังตื่น Deep-sleep: ดู §6 ก่อนเรียก!
void      gpio_deep_sleep_hold_en(void);                  // จำเป็นถ้าจะคงค่า digital GPIO ระหว่าง Deep-sleep
void      gpio_deep_sleep_hold_dis(void);
```

```c
// ===== RTC GPIO (ขา 0–21 เท่านั้น) — #include "driver/rtc_io.h" =====
esp_err_t rtc_gpio_init(gpio_num_t gpio_num);
esp_err_t rtc_gpio_set_direction(gpio_num_t gpio_num, rtc_gpio_mode_t mode); // เช่น RTC_GPIO_MODE_INPUT_ONLY
esp_err_t rtc_gpio_hold_en(gpio_num_t gpio_num);          // hold RTC GPIO ระหว่าง Deep-sleep ได้โดยตรง
esp_err_t rtc_gpio_hold_dis(gpio_num_t gpio_num);
```

## 5. ISR Pattern (โครงมาตรฐาน)
- ISR ต้อง `IRAM_ATTR` + สั้นที่สุด: **set flag หรือ `xQueueSendFromISR()` + `portYIELD_FROM_ISR()`** เท่านั้น — ห้าม `printf`/`ESP_LOGx` (ISR รันบน interrupt stack)
- ลำดับ: `gpio_config()` → `gpio_install_isr_service()` → `gpio_isr_handler_add(pin, isr, (void *)pin)` — ข้ามขั้น install = `ESP_ERR_INVALID_STATE`
- งานหนัก (log, parse) ทำใน task ปกติที่รอ queue — ดูตัวอย่างเต็มใน §8

## 6. Deep-sleep: Hold & Wake-cause
**Wake sources สรุป (ตารางเต็มอยู่ sleep.md):**
- Light-sleep: `gpio_wakeup_enable()` ทีละขา (RTC หรือ digital ก็ได้) → `esp_sleep_enable_gpio_wakeup()` → `esp_light_sleep_start()`
- Deep-sleep: **ext0/ext1 เท่านั้น** และใช้ได้เฉพาะ RTC GPIO 0–21
- ⚠️ เปิด `CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP` → ต้อง `rtc_gpio_init()` + `rtc_gpio_set_direction(..., RTC_GPIO_MODE_INPUT_ONLY)` ตั้งขาเป็น input ก่อน

**คงค่าขาข้าม Deep-sleep:**
```c
gpio_set_level(GPIO_NUM_38, 1);
gpio_hold_en(GPIO_NUM_38);           // ล็อกค่า (คงข้าม soft reset ด้วย)
gpio_deep_sleep_hold_en();           // บวกอันนี้จึงคงค่า digital GPIO ระหว่าง Deep-sleep
// RTC GPIO 0–21: ใช้ rtc_gpio_hold_en() แทน
```

**ปลด hold หลังตื่น (ลำดับสำคัญ):**
```c
gpio_set_direction(GPIO_NUM_38, GPIO_MODE_OUTPUT);  // 1) ตั้งโหมดก่อน
gpio_set_level(GPIO_NUM_38, 1);                     // 2) ตั้งระดับเดิม
gpio_hold_dis(GPIO_NUM_38);                         // 3) จึงปลด hold
// สลับลำดับ → ขากลับ default (input) ระดับเปลี่ยนทันที
```

**ตรวจสาเหตุการตื่น:**
```c
esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();   // v5.x
// ESP_SLEEP_WAKEUP_UNDEFINED = บูตแรก / EXT0 / EXT1 / TIMER / GPIO (light) / TOUCH / UART / BT / WIFI
// ⚠️ v6.0+: deprecated → uint64_t esp_sleep_get_wakeup_causes(void)
//   คืน bitmap (หลาย source ปลุกพร้อมกันได้): if (causes & ESP_SLEEP_WAKEUP_TIMER) { ... }
```

## 7. กับดักที่พบบ่อย (ผูก HALLUCINATION-HOT-LIST v3.1)
1. ISR ประกาศ `void isr(int gpio_num)` → ผิด ต้อง `void isr(void *arg)` → **GPIO-01**
2. ลืม `gpio_install_isr_service()` ก่อน `gpio_isr_handler_add()` → `ESP_ERR_INVALID_STATE`
3. `1UL << 31` ไม่พอสำหรับ pin_bit_mask → ต้อง `1ULL`
4. `gpio_config_t` ใช้ `gpio_pullup_t` แต่ `gpio_set_pull_mode()` ใช้ `gpio_pull_mode_t` — enum คนละชุด
5. ห้าม `printf`/`ESP_LOGx` ใน ISR → **GPIO-03**
6. ใช้ GPIO 22–25 → ไม่มีบน S3 → **GPIO-04**
7. `gpio_wakeup_enable()` ปลุก Deep-sleep → ไม่ได้ Light-sleep เท่านั้น → **SLEEP-03**
8. `gpio_hold_en()` อย่างเดียวคง digital GPIO ระหว่าง Deep-sleep ไม่ได้ → ต้องบวก `gpio_deep_sleep_hold_en()` → **GPIO-05**
9. GPIO33–37: ห้าม/ใช้ได้ตามโมดูล (Octal = ห้าม, Quad-only = ได้) → **SPI-03**
10. ป้าย FSPID/FSPICLK (GPIO35/36) เป็น alternate function ของ GPSPI2 ไม่ใช่ขา flash → **SPI-04**
11. GPIO3 ไม่มี internal pull ห้ามลอย → **GPIO-06**
12. `gpio_hold_dis()` ทันทีหลังตื่น → ระดับกระโดด ตั้ง direction+level ก่อน → **GPIO-08**
13. `esp_sleep_get_wakeup_cause()` บน v6.0+ → deprecated ใช้ `_causes()` bitmap → **SLEEP-08**
14. เปิด power-down ใน Light-sleep แล้วข้าม `rtc_gpio_init()` → wakeup เสีย → **GPIO-07**

## 8. ตัวอย่างมินิมัล
```c
#include <inttypes.h>
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "esp_log.h"

#define BTN GPIO_NUM_4
static const char *TAG = "btn";
static QueueHandle_t gpio_evt_queue = NULL;

static void IRAM_ATTR btn_isr(void *arg)            // ISR: สั้นที่สุด ห้าม printf/ESP_LOGx
{
    BaseType_t hpw = pdFALSE;
    uint32_t gpio_num = (uint32_t)arg;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, &hpw);
    portYIELD_FROM_ISR(hpw);
}

static void btn_task(void *arg)                     // งานหนักทำใน task ปกติ
{
    uint32_t io_num;
    for (;;) {
        if (xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {
            ESP_LOGI(TAG, "GPIO[%" PRIu32 "] intr, val: %d", io_num, gpio_get_level(io_num));
        }
    }
}

void app_main(void)
{
    gpio_config_t io = {
        .pin_bit_mask = 1ULL << BTN,
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .intr_type = GPIO_INTR_NEGEDGE,
    };
    ESP_ERROR_CHECK(gpio_config(&io));

    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));
    ESP_ERROR_CHECK(gpio_install_isr_service(ESP_INTR_FLAG_IRAM));
    ESP_ERROR_CHECK(gpio_isr_handler_add(BTN, btn_isr, (void *)BTN));  // ส่งเลข pin ผ่าน args

    xTaskCreate(btn_task, "btn_task", 2048, NULL, 10, NULL);
}
```
(ตัวอย่าง sleep/wakeup/hold อยู่ใน §6 — ใช้จาก那里ได้เลย)

## 9. v6.x Notes
- `gpio_pad_select_gpio()` **removed** → `gpio_reset_pin()` (deprecated ตั้งแต่ v5.0)
- `esp_sleep_enable_ext1_wakeup()` deprecated ตั้งแต่ v6.0 → variant `_io` + ตรวจสถานะด้วย bitmask
- `esp_sleep_get_wakeup_cause()` deprecated ตั้งแต่ v6.0 → `esp_sleep_get_wakeup_causes()` (bitmap)

## 10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
- `esp_sleep_get_wakeup_cause` vs `_causes` และ `esp_sleep_enable_ext1_wakeup_io()` — มีตั้งแต่เวอร์ชันไหนของ IDF เป้าหมาย (ยืนยัน behavior ใน v6.x แล้ว แต่ v5.2–v5.5 ยังไม่ได้เปิด esp_sleep.h เทียบ)
- `gpio_deep_sleep_hold_en()` กับ GPIO 33–37 บนโมดูล Octal — ขาที่ SPI0/1 ยึดอยู่ hold ไม่ได้อยู่แล้ว (ไม่มีเอกสารยืนยันตรง ๆ — ทดสอบหรือเลี่ยง)