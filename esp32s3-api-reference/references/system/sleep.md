# Sleep — v2.2
> Target: ESP32-S3 | ESP-IDF v5.2 – v6.1
> อ้างอิง: TRM v1.8 ch.10 (Low-power Management) | esp_sleep.h + esp_pm.h | Migration Guide v6.1
> เปลี่ยนแปลง v2.2: แปลงโครงสร้างตาม skill v2.7 (§0 TL;DR + §10 จุดตรวจซ้ำ + ผูกรหัส Hot-List) · **แก้ pitfall ค้างจาก v2.1 ข้อ 3** (`esp_sleep_enable_timer_wakeup()` คืน `esp_err_t` ครอบ `ESP_ERROR_CHECK` ได้ — ข้อความเดิมในไฟล์ยังเขียนกลับด้าน ไม่ตรงกับที่แก้ไว้ใน Hot-List แล้ว)

## 0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. `esp_deep_sleep_start()` เป็น **void ไม่ return** — โค้ดหลังบรรทัดนี้ไม่ทำงาน (ตื่น = รีบูต)
2. ตัวแปรคงข้าม deep sleep ต้อง `RTC_DATA_ATTR` — global/static ธรรมดาหายหมด
3. **Deep-sleep ปลุกด้วย GPIO ได้เฉพาะ RTC GPIO 0–21** (ext0/ext1) — GPIO 22–48 ปลุกไม่ได้
4. `gpio_wakeup_enable()` = **Light-sleep เท่านั้น** (+ `esp_sleep_enable_gpio_wakeup()`)
5. WiFi/BT/UART0/UART1 เป็น wake source ของ **Light-sleep เท่านั้น** (TRM 10.4-3)
6. บน S3 `ext1` ใช้ **io_mask = bit ตามเลข GPIO ตรง ๆ** (1ULL << 0..21) ไม่ใช่ bit ลำดับของ RTC IO
7. v6.0+: `esp_sleep_enable_ext1_wakeup()` → variant **`_io`** · `esp_sleep_get_wakeup_cause()` → **`_causes()`** คืน bitmap
8. UART wake: นับ **rising edges** (S3 ขั้นต่ำ 3), ไบต์แรกไม่เข้า driver ต้องส่งซ้ำ — ใช้ byte ที่มี edge เยอะ เช่น 0xFF (ASCII '0' ไม่พอ)
9. คูณเวลาเป็น µs ใช้ `ULL` เสมอ (`3600ULL * 1000000ULL`) — int ล้น
10. internal pull ของ RTC domain อยู่ได้ต่อเมื่อ **RTC_PERIPH เปิด** — ใช้ `esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_ON)` (หรือเปิด `CONFIG_ESP_SLEEP_RTC_RESET_ALL`)

## 1. Include & CMakeLists
```c
#include "esp_sleep.h"        // sleep API ทั้งหมด — อยู่ใน component พื้นฐาน ไม่ต้องเพิ่ม REQUIRES
#include "esp_attr.h"         // RTC_DATA_ATTR
#include "driver/rtc_io.h"    // rtc_gpio_pullup_en ฯลฯ (ext0/ext1 + pull ของ RTC domain)
#include "driver/uart.h"      // uart_set_wakeup_threshold (v5.x) — เมื่อใช้ UART wake
#include "esp_pm.h"           // esp_pm_configure — automatic light sleep
```

## 2. Hardware Facts (TRM v1.8 ch.10)
- **Power modes:** Active → Modem-sleep → **Light-sleep** (CPU halt, resume ต่อจากเดิม) → **Deep-sleep** (เหลือ RTC เท่านั้น, ตื่น = รีบูต)
- **Wake source ต่อโหมด (S3):**

| Wake source | Light-sleep | Deep-sleep |
|---|---|---|
| RTC timer | ✓ | ✓ |
| EXT0 / EXT1 (RTC GPIO 0–21) | ✓ | ✓ |
| GPIO (ทุกขา) | ✓ (`gpio_wakeup_enable`) | ✗ |
| TOUCH / ULP-FSM / ULP-RISC-V / XTAL_32K / BROWNOUT | ✓ | ✓ |
| Wi-Fi / BT / UART0 / UART1 | ✓ | ✗ |

- **ข้อจำกัด:** EXT1 signal ต้องถูกค้างนานกว่า 3 รอบ RTC slow clock · WiFi/BT wake ต้องสลับ Active/Modem-sleep/Light-sleep · stack ของ task ต้องอยู่ internal DRAM ไม่ใช่ PSRAM

## 3. Deep Sleep API
```c
void      esp_deep_sleep(uint64_t time_in_us);          // เข้า sleep ทันที + ตั้ง timer
void      esp_deep_sleep_start(void);                    // เข้า sleep ตามที่ config ไว้ — void ไม่ return!
esp_err_t esp_sleep_enable_timer_wakeup(uint64_t time_in_us);   // คืน esp_err_t — ครอบ ESP_ERROR_CHECK ได้
esp_err_t esp_sleep_enable_ext0_wakeup(gpio_num_t gpio_num, int level);  // RTC GPIO เท่านั้น
esp_err_t esp_sleep_enable_ext1_wakeup(uint64_t io_mask, esp_sleep_ext1_wakeup_mode_t mode);
// ⚠️ v6.0+: ตัวข้างบน deprecated → esp_sleep_enable_ext1_wakeup_io(io_mask, mode)
//   บน S3: io_mask = bit ตามเลข GPIO (1ULL << 0 ... 1ULL << 21)
esp_err_t esp_sleep_enable_touchpad_wakeup(void);
esp_err_t esp_sleep_enable_ulp_wakeup(void);
uint64_t  esp_sleep_get_ext1_wakeup_status(void);        // คืน bitmask — หาเลข GPIO ด้วย __builtin_ctzll()
// ⚠️ v6.0+: → esp_sleep_get_ext1_wakeup_status_io()
```

## 4. Light Sleep API
```c
esp_err_t esp_light_sleep_start(void);
esp_err_t esp_sleep_enable_gpio_wakeup(void);            // arm หลัง gpio_wakeup_enable() ครบทุกขา (ดู gpio.md §6)
esp_err_t esp_sleep_enable_uart_wakeup(int uart_num);    // UART0/1 เท่านั้น
esp_err_t esp_sleep_enable_wifi_wakeup(void);
esp_err_t esp_sleep_enable_bt_wakeup(void);
```
**Automatic light sleep (WiFi/PM ทำงาน):**
```c
esp_pm_config_t pm_config = {
    .max_freq_mhz = 80,
    .min_freq_mhz = 10,       // S3: 10 MHz
    .light_sleep_enable = true,
};
ESP_ERROR_CHECK(esp_pm_configure(&pm_config));
// ⚠️ เปิด auto light sleep แล้วห้ามเรียก esp_light_sleep_start() เอง — ระบบ PM จัดการ
```

## 5. ตรวจสาเหตุการตื่น
```c
esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();  // v5.x — ESP_SLEEP_WAKEUP_UNDEFINED = บูตแรก
// ⚠️ v6.0+: deprecated → uint64_t esp_sleep_get_wakeup_causes(void)
//   bitmap — หลาย source ปลุกพร้อมกันได้: if (causes & ESP_SLEEP_WAKEUP_TIMER) { ... }
```
- enum: `UNDEFINED / TIMER / EXT0 / EXT1 / GPIO / TOUCHPAD / UART / WIFI / BT / ULP / RISCV` (v6 rename type → `esp_sleep_source_t` ชื่อเดิมใช้ได้)

## 6. คงค่าข้าม Sleep
- ตัวแปร: `RTC_DATA_ATTR static int boot_count;`
- ระดับขา: `gpio_hold_en()` + `gpio_deep_sleep_hold_en()` / `rtc_gpio_hold_en()` และลำดับปลด hold หลังตื่น — **ดู gpio.md §6** (ครบทั้งหมดอยู่ที่นั่น)
- internal pull ของ RTC: `rtc_gpio_pullup_en()` แล้ว **ต้อง** `esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_ON)` ไม่งั้น pull หายเมื่อ RTC_PERIPH ดับ

## 7. กับดักที่พบบ่อย (ผูก HALLUCINATION-HOT-LIST v3.2)
1. โค้ดหลัง `esp_deep_sleep_start()` คิดว่าจะได้รัน → ไม่มีวัน — void ไม่ return → (§3)
2. global/static ธรรมดาคงค่าข้าม deep sleep → ต้อง `RTC_DATA_ATTR` → (§6)
3. `esp_sleep_enable_timer_wakeup()` คิดว่าคืน uint64_t ห้ามครอบ `ESP_ERROR_CHECK` → **ผิด** (ค้างจาก v1.1 — แก้แล้ว) — คืน `esp_err_t` ครอบได้; จุดระวังจริงคือ overflow → **SLEEP-01 / SLEEP-07**
4. ปลุก deep sleep ด้วย GPIO 22–48 → ไม่ได้ เฉพาะ RTC GPIO 0–21 → **SLEEP-02**
5. `gpio_wakeup_enable()` ใช้กับ deep sleep → Light-sleep เท่านั้น → **SLEEP-03**
6. WiFi/BT/UART ปลุก deep sleep → Light-sleep เท่านั้น → **SLEEP-04**
7. v6 ใช้ ext1/get_wakeup_cause ตัวเดิม → deprecated → `_io` / `_causes` → **SLEEP-05 / SLEEP-08**
8. UART wake: UART2 ไม่ได้ · ไบต์แรกไม่เข้า driver · threshold นับ rising edges → **SLEEP-06 / UART-06**
9. `3600 * 1000000` ใน int → overflow ใช้ `ULL` → **SLEEP-07**
10. ext1 io_mask คิดเป็น bit ลำดับ RTC IO → บน S3 คือ bit ตาม**เลข GPIO ตรง ๆ** → (§3)
11. ใช้ internal pull โดยไม่เปิด RTC_PERIPH → pull หาย → (§6)
12. stack ใน PSRAM แล้วเข้า light/deep sleep → พัง — ต้อง internal DRAM → (§2)

## 8. ตัวอย่างมินิมัล
```c
#include "esp_sleep.h"
#include "esp_attr.h"
#include "driver/rtc_io.h"
#include "esp_log.h"

RTC_DATA_ATTR static int boot_count = 0;

void app_main(void)
{
    boot_count++;
    switch (esp_sleep_get_wakeup_cause()) {          // v6+: esp_sleep_get_wakeup_causes() bitmap
    case ESP_SLEEP_WAKEUP_EXT1: {
        uint64_t mask = esp_sleep_get_ext1_wakeup_status();
        ESP_LOGI("sleep", "woke by GPIO%d (boot #%d)", __builtin_ctzll(mask), boot_count);
        break;
    }
    case ESP_SLEEP_WAKEUP_TIMER:
        ESP_LOGI("sleep", "woke by timer (boot #%d)", boot_count);
        break;
    default:
        ESP_LOGI("sleep", "power-on reset");
        break;
    }

    rtc_gpio_pullup_en(GPIO_NUM_0);                  // pull ของ RTC domain
    ESP_ERROR_CHECK(esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_ON)); // กัน pull หาย
    ESP_ERROR_CHECK(esp_sleep_enable_ext1_wakeup(1ULL << GPIO_NUM_0, ESP_EXT1_WAKEUP_ANY_LOW));
    esp_sleep_enable_timer_wakeup(60ULL * 1000000ULL);   // 60 s — ULL!
    esp_deep_sleep_start();                          // ไม่ return
}
```

## 9. v6.x Notes
- `esp_sleep_enable_ext1_wakeup()` / `esp_sleep_get_ext1_wakeup_status()` / `esp_sleep_get_wakeup_cause()` — **deprecated ตั้งแต่ v6.0** → variant `_io` / `esp_sleep_get_wakeup_causes()` (bitmap)
- type rename: `esp_sleep_wakeup_cause_t` → `esp_sleep_source_t` (ชื่อ enum เดิมยังใช้ได้)
- UART wake มีชุดใหม่ `uart_wakeup_setup()` + `uart_wakeup_cfg_t` (คู่ `uart_set_wakeup_threshold` เดิม)

## 10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
- `esp_sleep_enable_ext1_wakeup_io()` / `_status_io()` — มีตั้งแต่ IDF เวอร์ชันไหน (v5.4? ยืนยันใน esp_sleep.h ของเวอร์ชันเป้าหมาย)
- `esp_sleep_get_wakeup_causes()` — เช็คว่ามีใน v5.5 หรือเพิ่มมาใน v6.0 จริง ๆ
- ค่า `min_freq_mhz` ขั้นต่ำของ S3 กับ PM (10 MHz — ตาม docs; ทดสอบบนบอร์ดถ้าโหมดกินไฟสำคัญ)
- `uart_wakeup_cfg_t` โครงสร้าง field ตามเวอร์ชัน