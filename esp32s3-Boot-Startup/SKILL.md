---
name: esp32s3-boot-startup
version: 1.0.0
description: Boot & Startup skill สำหรับ ESP32-S3 — ครอบคลุม boot flow ตั้งแต่ power-on ถึง app_main(), boot mode selection (strapping pins), RTC memory boot, deep sleep wake stub programming, retention, และ custom bootloader considerations
triggers:
  - boot
  - startup
  - app_main
  - bootloader
  - RTC memory
  - RTC FAST
  - RTC SLOW
  - deep sleep
  - wake stub
  - sleep mode
  - retention
  - strapping pins
  - reset
  - power on
  - wake up
  - light sleep
  - modem sleep
  - RTC_CNTL
---

# Skill: Boot & Startup for ESP32-S3

> **ขอบเขต:** ใช้สำหรับเข้าใจและเขียนโค้ดที่เกี่ยวข้องกับกระบวนการบูต, deep sleep, RTC memory, และ wake stub บน ESP32-S3
>
> **แหล่งข้อมูลหลัก:** ESP32-S3 TRM v1.8 Chapter 8 (Chip Boot Control) + Chapter 10 (Low-power Management / RTC_CNTL) + Chapter 4 (System and Memory, RTC section) + [ESP-IDF Sleep Modes](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/sleep_modes.html) + [ESP-IDF App Startup Flow](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/startup.html)

---

## 1. ภาพใหญ่: Boot Flow ตั้งแต่ Power-On ถึง app_main()

```
┌─────────────────────────────────────────────────┐
│  0. Power-On / Hardware Reset                    │
│     └─ อ่าน strapping pins (GPIO0, GPIO45,        │
│        GPIO46, GPIO3) เพื่อกำหนด boot mode       │
├─────────────────────────────────────────────────┤
│  1. ROM Bootloader (First-stage, in ROM)         │
│     └─ อยู่ใน Internal ROM (0x4000_0000)         │
│     └─ ตรวจสอบ strapping + eFuse                │
│     └─ เลือก boot mode (SPI Flash / Direct Boot)  │
├─────────────────────────────────────────────────┤
│  2. Second-stage Bootloader (in Flash)          │
│     └─ อยู่ที่ address 0x4200_0000+               │
│     └─ โหลด partition table                     │
│     └─ โหลด application image                   │
│     └─ ถ้ามี flash encryption / secure boot:       │
│        ถอดรหัส + ตรวจสอบลายเซ็น                  │
│     └─ โหลด app เข้า SRAM (instruction + data)   │
├─────────────────────────────────────────────────┤
│  3. Application Startup                          │
│     └─ โค้ดเริ่มที่ call_start_cpu0 (in ROM)      │
│     └─ ตั้งค่า cache, MMU                         │
│     └─ call_start_cpu1 (ถ้ามี dual-core)         │
│     └─ รันโค้ด ESP-IDF startup (start_cpu0_default)│
│        • เปิด FreeRTOS scheduler                 │
│        • ตั้งค่า FreeRTOS tick                     │
│        • สร้าง main task                         │
│        • เรียก app_main()                        │
├─────────────────────────────────────────────────┤
│  4. app_main() — โค้ดของผู้ใช้                    │
│     └─ return ได้ — scheduler ยังรันต่อ          │
└─────────────────────────────────────────────────┘
```

> **แหล่งเว็บ:** [ESP-IDF App Startup Flow — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/startup.html) — "The application startup flow is: 1. Second-stage bootloader executes; 2. Second-stage bootloader loads application image; 3. Application startup code runs (call_start_cpu0 in ROM); 4. ESP-IDF startup code runs (start_cpu0_default); 5. Main task is created and app_main is called"

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 8 — "In most practical scenarios, this program is the 2nd stage bootloader, which later boots the target application"

---

## 2. Boot Mode Selection — Strapping Pins และ eFuse

### 2.1 Strapping Pins ที่ใช้ตอน Power-On

| Strapping Pin | หน้าที่ | ค่าเริ่มต้น (Pull-up/down) |
|---|---|---|
| **GPIO0** | Boot mode selection (ร่วมกับ GPIO46) | Pull-up (default = 1) |
| **GPIO46** | Boot mode selection + ROM message printing | Pull-down (default = 0) |
| **GPIO45** | VDD_SPI voltage (3.3V vs 1.8V) | Pull-down (default = 0 → 3.3V) |
| **GPIO3** | JTAG signal source | Pull-up (default = 1) |

> **แหล่งข้อมูล:** POM — "The chip allows for configuring the following boot parameters through strapping pins and eFuse parameters at power-up or a hardware reset, without microcontroller interaction"

### 2.2 Boot Mode จาก GPIO0 และ GPIO46

| GPIO46 | GPIO0 | Boot Mode |
|---|---|---|
| 0 (Low) | 0 (Low) | **SPI Flash Boot** (default ปกติ) — โหลดจาก flash |
| 0 (Low) | 1 (High) | **Joint Download Boot** (UART0/USB download) |
| 1 (High) | 0 (Low) | SPI Boot (เหมือนกรณีแรก) |
| 1 (High) | 1 (High) | Joint Download Boot (เหมือนกรณีที่สอง) |

**⚠️ ค่าเริ่มต้น:** GPIO0 = pull-up, GPIO46 = pull-down → **SPI Flash Boot** (โหมดปกติ)

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 8 — "To change the strapping bit values, users can apply external pull-down/pull-up resistors, or use host MCU GPIOs to control the voltage level of these pins when powering on ESP32-S3. After the reset is released, the strapping pins work as normal-function pins"

### 2.3 Direct Boot Mode (โหมดพิเศษ — ไม่ผ่าน 2nd-stage bootloader)

```
Direct Boot:
  • โหลดโค้ดจาก flash ตรงเข้า memory (ไม่ผ่าน bootloader 2)
  • สูงสุดที่ app ขนาด 1.5 MB
  • รองรับเฉพาะ single-core (CPU1 ปิด)
  • ไม่รองรับ Secure Boot
  • ใช้สำหรับ application ขนาดเล็กที่ต้องการ boot เร็ว

เปิดใช้งาน:
  • เขียนค่า 0xaedb041d ที่ address 0x42000000 (คำแรกของไฟล์ bin)
  • หรือตั้งค่าใน sdkconfig:
    CONFIG_BOOTLOADER_DIRECT_FLASH_BOOT=y
```

> **แหล่งเว็บ:** [ESP-IDF Boot — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/boot.html)

---

## 3. การเปลี่ยน Strapping Pins — ความเสี่ยงและวิธีปลอดภัย

### 3.1 ⚠️ ความเสี่ยงของ GPIO0

**อันตราย:** การใช้ GPIO0 เป็น output pin ใน application อาจทำให้ chip บูตเข้าโหมดผิด (เช่น Download Boot) ถ้า pin มีค่าผิดตอน reset

### 3.2 วิธีปลอดภัย: รอจนกว่าจะผ่าน boot

```c
// ✅ ปลอดภัย: เปลี่ยน GPIO0 หลัง boot เสร็จ
void app_main(void) {
    // รอจน chip บูตเสร็จและ app_main ทำงาน
    // ตอนนี้ strapping pins ทำหน้าที่เป็น normal GPIO แล้ว
    
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << GPIO_NUM_0),
        .mode = GPIO_MODE_OUTPUT,
        // ... 
    };
    gpio_config(&io_conf);
}
```

### 3.3 หลีกเลี่ยงการใช้ strapping pins เป็น input ที่มี pull-up/down ภายนอก

ถ้า GPIO0 ต้องต่อ pull-down ภายนอก → ทุกครั้งที่ reset จะบูตเข้า Download Boot ไม่ใช่ SPI Boot

> **แหล่งเว็บ:** [ESP32 Forum — Strapping Pins](https://esp32.com/viewtopic.php?t=9893)

---

## 4. RTC Memory — สถานะข้าม Deep Sleep

### 4.1 ภาพรวม RTC Memory

```
RTC FAST Memory (8 KB)
  • Address: 0x600F_E000 – 0x600F_FFFF
  • เข้าถึงได้โดย CPU เท่านั้น (ULP ไม่ได้)
  • ใช้เก็บ instructions และ data ที่ต้องข้าม deep sleep
  • ใช้เป็น wake stub (เร็วกว่า RTC SLOW)

RTC SLOW Memory (8 KB)
  • Address: 0x5000_0000 – 0x5000_1FFF
  • เข้าถึงได้โดยทั้ง CPU และ ULP co-processor
  • ใช้แชร์ข้อมูลระหว่าง CPU และ ULP
  • ใช้เป็น wake stub ได้ (ช้ากว่า RTC FAST เล็กน้อย)
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 4 — "RTC FAST Memory (8 KB): RTC FAST memory can only be accessed by the CPU, and cannot be accessed by the ULP co-processor. It is generally used to store instructions and data that needs to persist across a deep sleep. RTC SLOW Memory (8 KB): The RTC SLOW memory can be accessed by both the CPU and the ULP co-processor"

### 4.2 ใช้ RTC Memory ใน C Code

```c
#include "esp_sleep.h"

// ตัวแปรที่เก็บใน RTC SLOW Memory (เก็บข้าม deep sleep)
RTC_NOINIT_ATTR static uint32_t boot_count;
RTC_NOINIT_ATTR static uint16_t sensor_data[64];

void app_main(void) {
    boot_count++;
    printf("Boot count: %d\n", boot_count);

    // เข้า deep sleep 10 วินาที
    esp_sleep_enable_timer_wakeup(10 * 1000000);
    esp_deep_sleep_start();
}
```

> **แหล่งเว็บ:** [ESP-IDF RTC Memory — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/misc_system_api.html) — "Values placed in this area will survive after a system reset (watchdog, software reset, etc.) or a deep sleep"

### 4.3 แยกประเภทของ RTC Memory Attribute

| Attribute | วางใน | ลักษณะเฉพาะ |
|---|---|---|
| `RTC_NOINIT_ATTR` | RTC SLOW Memory | ไม่ initialize ตอน boot (คงค่าเดิม) |
| `RTC_DATA_ATTR` | RTC SLOW Memory | initialize ตอน boot |
| `RTC_RODATA_ATTR` | RTC SLOW Memory | read-only data |
| `RTC_IRAM_ATTR` | RTC FAST Memory | instructions (wake stub) |
| `RTC_SLOW_MEM` (linker script) | RTC SLOW Memory | manual linker placement |

---

## 5. Sleep Modes — ภาพรวม

ESP32-S3 มี 3 sleep modes หลัก:

| Mode | CPU | SRAM/ROM | RTC Peripherals | RTC Memory | Wakeup Time | Power |
|---|---|---|---|---|---|---|
| **Modem Sleep** | รัน (idle) | on | on | on | สั้นมาก | ~20–80 mA |
| **Light Sleep** | paused | retained | on | on | สั้น | ~0.8 mA |
| **Deep Sleep** | off | off (สูญเสีย) | บางส่วน on | on | ยาว (SPI boot) | ~10–150 μA |

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 10 — "The wakeup time from Deep-sleep mode is much longer, compared to Light-sleep and Modem-sleep modes, because the ROMs and RAMs are both powered down in this case, and the CPU needs more time for SPI booting"

### 5.1 Light Sleep

```c
#include "esp_sleep.h"

// ตั้งค่า wake source
esp_sleep_enable_timer_wakeup(5 * 1000000);  // 5 วินาที
esp_sleep_enable_gpio_wakeup(BUTTON_GPIO, ESP_SLEEP_WAKEUP_LOW);

// เข้า light sleep
esp_light_sleep_start();

// ตอนตื่น: โค้ดทำงานต่อจากที่ค้างไว้
```

### 5.2 Deep Sleep

```c
#include "esp_sleep.h"

void app_main(void) {
    // ตั้งค่า wake source
    esp_sleep_enable_timer_wakeup(10 * 1000000);
    esp_sleep_enable_ext0_wakeup(GPIO_NUM_0, 0);  // ตื่นเมื่อ GPIO0 = 0

    // เข้า deep sleep
    esp_deep_sleep_start();

    // ⚠️ โค้ดหลังจากนี้จะไม่ทำงาน (chip จะ reset ตอนตื่น)
    // ตื่นแล้วจะเริ่มตั้งแต่ app_main ใหม่
}
```

> **แหล่งเว็บ:** [ESP-IDF Sleep Modes — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/sleep_modes.html)

---

## 6. Wake Sources สำหรับ Deep Sleep

| Wake Source | API | ใช้ได้ใน |
|---|---|---|
| **Timer** | `esp_sleep_enable_timer_wakeup(us)` | Light + Deep |
| **GPIO (ext0)** | `esp_sleep_enable_ext0_wakeup(pin, level)` | Deep (1 pin, RTC GPIO เท่านั้น) |
| **GPIO (ext1)** | `esp_sleep_enable_ext1_wakeup(mask, mode)` | Deep (หลาย pins, RTC GPIO) |
| **GPIO (light sleep)** | `esp_sleep_enable_gpio_wakeup(pin, level)` | Light Sleep (ทุก GPIO) |
| **Touchpad** | `esp_sleep_enable_touchpad_wakeup()` | Deep + Light |
| **ULP** | `esp_sleep_enable_ulp_wakeup()` | Deep |
| **UART** | `esp_sleep_enable_uart_wakeup(port, threshold)` | Light Sleep |

> **แหล่งเว็บ:** [ESP-IDF Sleep Modes — Wake Sources](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/sleep_modes.html)

**⚠️ ข้อจำกัด:**
- `ext0` และ `ext1` ใช้ได้กับ **RTC GPIO** เท่านั้น (ไม่ใช่ทุก GPIO)
- Deep sleep สามารถใช้ได้หลาย wake source พร้อมกัน

---

## 7. Deep Sleep Wake Stub — การเขียน Wake Stub

### 7.1 Wake Stub คืออะไร?

Wake stub คือ โค้ดขนาดเล็ก (สูงสุด 8 KB) ที่ทำงานทันทีหลังตื่นจาก deep sleep **ก่อน**ที่จะผ่าน SPI booting และ 2nd-stage bootloader ทำให้ตื่นเร็วขึ้นมาก

```
Deep Sleep Wake:
  ┌─────────────────────────────────┐
  │  วิธีปกติ (ช้า):                 │
  │  CPU ตื่น → SPI boot → 2nd-stage  │
  │  → ESP-IDF startup → app_main    │
  │  → เวลา: ~200 ms                │
  ├─────────────────────────────────┤
  │  วิธี Wake Stub (เร็ว):          │
  │  CPU ตื่น → รัน stub ใน RTC mem   │
  │  → ตัดสินใจ: กลับ sleep หรือ        │
  │    บูตเต็มรูปแบบ                  │
  │  → เวลา: ~1–5 ms               │
  └─────────────────────────────────┘
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 10 — "users can store codes (so called 'deep sleep wake stub' of up to 8 KB) either in RTC fast memory or RTC slow memory, which doesn't require the above-mentioned SPI booting, thus speeding up the wakeup process"

### 7.2 RTC SLOW Memory Boot (Method 1)

```
1. เซ็ต RTC_CNTL_PROCPU_STAT_VECTOR_SEL = 0
2. เข้า sleep
3. หลัง CPU ตื่น, reset vector เริ่มที่ 0x5000_0000 (RTC SLOW)
   แทน 0x4000_0400 (ROM bootloader) → ไม่ผ่าน SPI booting
4. โค้ดใน RTC SLOW Memory ทำงานทันที
5. โค้ดนี้ต้องการการ initialize แบบ partial C environment เท่านั้น
```

### 7.3 RTC FAST Memory Boot (Method 2)

```
1. เซ็ต RTC_CNTL_PROCPU_STAT_VECTOR_SEL = 1
2. คำนวณ CRC ของ RTC FAST Memory → เก็บใน RTC_CNTL_RTC_STORE7_REG
3. เซ็ต entry address ใน RTC_CNTL_RTC_STORE6_REG
4. เข้า sleep
5. หลังตื่น:
   - ROM unpacking + initialization
   - คำนวณ CRC อีกครั้ง
   - ถ้า CRC ตรง → กระโดดไป entry address ใน RTC FAST Memory
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 10 Section 10.6 (RTC Boot) + Figure 10.6-1 (ESP32-S3 Boot Flow)

### 7.4 ใช้ ESP-IDF API สำหรับ Wake Stub

ESP-IDF มี API ที่ทำให้เขียน wake stub ง่ายขึ้น — ไม่ต้องเขียน assembly เอง:

```c
#include "esp_sleep.h"
#include "esp_wake_stub.h"

// โค้ด wake stub — รันใน RTC memory ทันทีหลังตื่น
void IRAM_ATTR my_wake_stub(void) {
    // ⚠️ โค้ดนี้รันใน environment จำกัด:
    //   - ไม่มี FreeRTOS
    //   - ไม่มี heap
    //   - ใช้ได้เฉพาะ RTC memory + ROM functions
    //   - ห้ามใช้ printf (ใช้ esp_rom_printf แทน)
    
    // ตัวอย่าง: เช็คเงื่อนไข ก่อนบูตเต็มรูปแบบ
    if (check_sensor_threshold()) {
        // ยังไม่ถึงเงื่อนไข → กลับ sleep อีก
        esp_wake_stub_sleep(&my_wake_stub, 10000000);  // 10 วินาที
    } else {
        // ถึงเงื่อนไข → บูตเต็มรูปแบบ (เรียก app_main)
        esp_wake_stub_reset();  // สังให้บูตเต็มรูปแบบ
    }
}

void app_main(void) {
    // ตั้ง wake stub ก่อนเข้า deep sleep
    esp_sleep_set_wake_stub(my_wake_stub);
    
    esp_sleep_enable_timer_wakeup(10 * 1000000);
    esp_deep_sleep_start();
}
```

> **แหล่งเว็บ:** [ESP-IDF Sleep Stubs — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/sleep_modes.html) — "Wake stub runs in a limited environment (no RTOS, no heap, only RTC memory and ROM functions)"

**⚠️ ข้อจำกัดของ Wake Stub:**
- ทำงานใน environment จำกัด (no FreeRTOS, no heap)
- ใช้ได้เฉพาะ ROM functions + RTC memory
- ขนาดโค้ดสูงสุด 8 KB
- ห้ามใช้ `printf` — ใช้ `esp_rom_printf` แทน
- ห้ามใช้ peripherals ที่ต้องการ clock config เต็มรูปแบบ

---

## 8. Deep Sleep Retention — บันทึก CPU State

### 8.1 Retention คืออะไร?

Retention คือ การบันทึก CPU register state (428 registers) ลง SRAM ก่อนเข้า deep sleep และ restore กลับตอนตื่น ทำให้โค้ดทำงานต่อจากที่ค้างไว้ (เหมือน light sleep แต่ใช้พลังงานต่ำกว่า)

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 10 — "Use Retention DMA to store CPU information before the chip enters sleep. Restore information from Retention DMA to CPU after CPU wakes up"

### 8.2 การทำงาน

```
ก่อนเข้า Deep Sleep:
  1. เปิด Retention function: RTC_CNTL_RETENTION_EN = 1
  2. Retention DMA บันทึก 428 CPU registers + 4 config words
     ลง SRAM ที่จองไว้ (สูงสุด 432 words = 1728 bytes)
  3. เข้า deep sleep

หลังตื่น:
  4. Retention DMA restore ค่าจาก SRAM กลับเข้า CPU registers
  5. โค้ดทำงานต่อจากที่ค้างไว้
```

### 8.3 ข้อจำกัด Retention

- ต้องจอง SRAM อย่างน้อย 432 words ก่อนเข้า sleep
- SRAM ส่วนนี้ไม่สามารถใช้เก็บข้อมูลอื่นได้ขณะ retention active
- ใช้ได้กับ CPU0 เท่านั้น (ไม่รองรับ dual-core retention แบบเต็ม)
- ใช้ร่วมกับ wake stub ได้

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 10 Section 10.5 (Retention Function)

---

## 9. Custom Bootloader — ข้อควรระวัง

ESP-IDF อนุญาตให้แก้ไข 2nd-stage bootloader ได้:

### 9.1 เปิดใช้ Custom Bootloader

```
# idf.py menuconfig
Bootloader config → 
  → Custom bootloader version (ใส่ path ของโค้ด custom)
```

### 9.2 โครงสร้างไฟล์ Custom Bootloader

```
my_project/
├── bootloader_components/        ← โฟลเดอร์นี้
│   └── custom_bootloader/        ← ชื่อ component
│       ├── CMakeLists.txt
│       └── custom_bootloader.c   ← โค้ด custom
└── main/
    └── main.c
```

### 9.3 Hook Points ใน Bootloader

```c
// bootloader_components/custom_bootloader/custom_bootloader.c
#include "esp_bootloader.h"

// Hook: ทำงานก่อนโหลด partition table
void bootloader_pre_callback(void) {
    // เช่น: ตรวจสอบ hardware revision, เลือก OTA partition
}

// Hook: ทำงานหลังโหลดแต่ก่อนเริ่ม app
void bootloader_post_callback(void) {
    // เช่น: ตรวจสอบ image hash, ตั้งค่าพิเศษ
}
```

> **แหล่งเว็บ:** [ESP-IDF Bootloader — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/bootloader.html)

**⚠️ ข้อควรระวัง:**
- Bootloader ทำงานใน environment จำกัด (no FreeRTOS, no heap)
- ห้ามใช้ functions ที่อยู่ใน flash (ต้องใช้ ROM functions หรือวางใน IRAM)
- ผิดพลาดได้ → chip อาจบูตไม่ได้ (ต้อง erase flash ผ่าน esptool)

---

## 10. การตื่นจาก Deep Sleep — ตรวจสอบ Wake Reason

```c
#include "esp_sleep.h"

void app_main(void) {
    // ตรวจสอบ wake reason
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
    
    switch (cause) {
        case ESP_SLEEP_WAKEUP_TIMER:
            printf("ตื่นจาก timer\n");
            break;
        case ESP_SLEEP_WAKEUP_EXT0:
            printf("ตื่นจาก GPIO (ext0)\n");
            break;
        case ESP_SLEEP_WAKEUP_EXT1:
            printf("ตื่นจาก GPIO (ext1)\n");
            break;
        case ESP_SLEEP_WAKEUP_TOUCHPAD:
            printf("ตื่นจาก touchpad\n");
            break;
        case ESP_SLEEP_WAKEUP_ULP:
            printf("ตื่นจาก ULP\n");
            break;
        case ESP_SLEEP_WAKEUP_UNDEFINED:
        default:
            printf("ไม่ใช่การตื่นจาก deep sleep (power-on หรือ reset)\n");
            break;
    }
}
```

> **แหล่งเว็บ:** [ESP-IDF Sleep Wakeup Cause](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/sleep_modes.html)

---

## 11. RTC_CNTL Register — สำหรับ Low-Level Control

### 11.1 RTC_CNTL Register หลัก

| Register | Address | หน้าที่ |
|---|---|---|
| `RTC_CNTL_OPTIONS0_REG` | `0x6000_8000 + 0x000` | Boot config, RTC options |
| `RTC_CNTL_STATE0_REG` | `0x6000_8000 + 0x004` | RTC state machine |
| `RTC_CNTL_CNTL_RST_EN_REG` | `0x6000_8000 + ...` | RTC reset control |
| `RTC_CNTL_WAKEUP_STATE_REG` | `0x6000_8000 + ...` | Wakeup source status |
| `RTC_CNTL_RETENTION_CTRL_REG` | `0x6000_8000 + ...` | Retention enable |
| `RTC_CNTL_STORE0_REG` – `STORE7_REG` | — | Scratch registers (4 bytes each) |
| `RTC_CNTL_PROCPU_STAT_VECTOR_SEL` | (bit in OPTIONS0) | เลือก RTC boot vector |

### 11.2 RTC_CNTL_STORE Registers — สำหรับเก็บค่าข้าม Sleep

มี 8 scratch registers (STORE0–STORE7) แต่ละตัว 4 bytes ใช้เก็บค่าข้าม reset และ deep sleep:

```c
// ตัวอย่าง: ใช้ RTC_CNTL_STORE แบบ low-level
#define RTC_CNTL_STORE0_REG (*(volatile uint32_t *)(0x60008000 + 0x060))
#define RTC_CNTL_STORE1_REG (*(volatile uint32_t *)(0x60008000 + 0x064))

void save_boot_info(void) {
    RTC_CNTL_STORE0_REG = 0xDEADBEEF;  // ค่าข้าม deep sleep
    RTC_CNTL_STORE1_REG = boot_reason;
}

uint32_t read_boot_info(void) {
    return RTC_CNTL_STORE0_REG;  // อ่านคืนหลังตื่น
}
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 10 — RTC_CNTL registers

### 11.3 RTC_CNTL_PROCPU_STAT_VECTOR_SEL

Bit สำคัญที่ควบคุมว่าจะบูตจาก RTC memory หรือ ROM:

```c
// อ่านค่าปัจจุบัน
uint32_t options0 = REG_READ(RTC_CNTL_OPTIONS0_REG);
uint32_t vector_sel = (options0 >> RTC_CNTL_PROCPU_STAT_VECTOR_SEL_S) 
                      & RTC_CNTL_PROCPU_STAT_VECTOR_SEL_V;

// vector_sel = 0 → บูตจาก RTC SLOW (0x5000_0000)
// vector_sel = 1 → บูตจาก RTC FAST (ผ่าน CRC check)
```

---

## 12. โครงสร้างไฟล์ ESP-IDF สำหรับ Deep Sleep Project

```
my_sleep_project/
├── CMakeLists.txt
├── sdkconfig.defaults
├── main/
│   ├── CMakeLists.txt
│   ├── main.c                  ← app_main + sleep logic
│   └── wake_stub.c             ← wake stub code
└── components/
    └── my_rtc_utils/
        ├── CMakeLists.txt
        ├── include/
        │   └── my_rtc_utils.h
        └── my_rtc_utils.c       ← RTC memory helpers
```

### 12.1 sdkconfig.defaults ที่จำเป็น

```
# Deep sleep
CONFIG_BOOTLOADER_SKIP_RTC_MEM_INIT=y    # อย่า initialize RTC memory ตอน boot (เพื่อรักษาค่า)

# Wake stub
CONFIG_ESP_SLEEP_WAKE_STUB=y              # เปิดใช้ wake stub API

# RTC memory
CONFIG_ESP_RTC_MEM_SIZE=8192             # ขนาด RTC memory (default 8KB)

# RTC clock source
CONFIG_ESP32S3_RTC_CLK_SRC_INT_RC=y     # ใช้ internal 150kHz RC (default)
```

> **แหล่งเว็บ:** [ESP-IDF Configuration — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/kconfig.html)

---

## 13. Reset Reasons — ตรวจสอบสาเหตุการ Reset

```c
#include "esp_reset_reason.h"
#include "esp_system.h"

void app_main(void) {
    esp_reset_reason_t reason = esp_reset_reason();
    
    switch (reason) {
        case ESP_RST_POWERON:
            printf("Power-on reset\n");
            break;
        case ESP_RST_EXT:
            printf("External reset (CHIP_PU pin)\n");
            break;
        case ESP_RST_SW:
            printf("Software reset (esp_restart())\n");
            break;
        case ESP_RST_PANIC:
            printf("Panic/crash reset\n");
            break;
        case ESP_RST_INT_WDT:
            printf("Interrupt watchdog reset\n");
            break;
        case ESP_RST_TASK_WDT:
            printf("Task watchdog reset\n");
            break;
        case ESP_RST_DEEPSLEEP:
            printf("Wake from deep sleep\n");
            break;
        case ESP_RST_BROWNOUT:
            printf("Brownout reset\n");
            break;
        default:
            printf("Unknown reset reason: %d\n", reason);
            break;
    }
}
```

> **แหล่งเว็บ:** [ESP-IDF Reset Reason](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/misc_system_api.html)

---

## 14. Power Management — Clock Configuration ก่อน Sleep

### 14.1 Dynamic Frequency Scaling (DFS)

```c
#include "esp_pm.h"

// ตั้งค่า power management
esp_pm_config_t pm_config = {
    .max_freq_mhz = 240,            // ความถี่สูงสุด (เมื่อ CPU ใช้งาน)
    .min_freq_mhz = 80,            // ความถี่ต่ำสุด (เมื่อ CPU idle)
    .light_sleep_enable = false,   // ไม่ใช้ auto light sleep
};
esp_pm_configure(&pm_config);

// เมื่อ CPU idle → ลดความถี่อัตโนมัติ (ประหยัดพลังงาน)
// เมื่อมี task ทำงาน → เพิ่มความถี่อัตโนมัติ
```

### 14.2 ล็อกความถี่ CPU (กัน DFS ขณะทำงานสำคัญ)

```c
#include "esp_pm.h"

esp_pm_lock_handle_t lock;
esp_pm_lock_create(ESP_PM_CPU_FREQ_MAX, 0, "my_lock", &lock);

// ล็อกความถี่สูงสุดขณะทำงานสำคัญ
esp_pm_lock_acquire(lock);
// ... โค้ดที่ต้องการ CPU 240 MHz ...
esp_pm_lock_release(lock);
```

> **แหล่งเว็บ:** [ESP-IDF Power Management — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/power_management.html)

---

## 15. Checklist ก่อนเขียน Deep Sleep / Wake Stub Code

- [ ] ตรวจสอบ wake reason ใน `app_main()` (`esp_sleep_get_wakeup_cause()`)
- [ ] ใช้ `RTC_NOINIT_ATTR` สำหรับตัวแปรที่ต้องข้าม deep sleep
- [ ] เลือก wake source ที่เหมาะสม (timer, GPIO ext0/ext1, touch, ULP)
- [ ] ถ้าใช้ GPIO wake: ตรวจสอบว่าเป็น RTC GPIO สำหรับ deep sleep
- [ ] Wake stub ใช้ ROM functions เท่านั้น (ห้ามใช้ printf, ห้าม malloc)
- [ ] Wake stub มี `IRAM_ATTR` และ `RTC_IRAM_ATTR` (หรือวางใน RTC FAST)
- [ ] ถ้าใช้ retention: จอง SRAM 432 words ก่อน sleep
- [ ] ถ้าใช้ custom bootloader: ตรวจสอบว่าไม่ใช้ flash functions
- [ ] อย่าใช้ strapping pins (GPIO0, GPIO46, GPIO45, GPIO3) เป็น wake source โดยไม่ระวัง
- [ ] ตั้ง `CONFIG_BOOTLOADER_SKIP_RTC_MEM_INIT=y` ถ้าต้องการรักษาค่า RTC memory
- [ ] ทดสอบ wake time จริง (ใช้ logic analyzer หรือ GPIO toggle)

---

## 16. Common Mistakes ที่ต้องหลีกเลี่ยง

| ผิด | ถูก | เหตุผล |
|---|---|---|
| ใช้ `printf` ใน wake stub | ใช้ `esp_rom_printf` | Wake stub ไม่มี driver เต็มรูปแบบ |
| ใช้ GPIO ปกติสำหรับ deep sleep wake | ใช้ RTC GPIO | Deep sleep ตื่นได้จาก RTC GPIO เท่านั้น |
| ลืม `RTC_NOINIT_ATTR` บนตัวแปรที่ต้องข้าม sleep | ใส่ `RTC_NOINIT_ATTR` | ตัวแปรจะ reset ตอนตื่น |
| ใช้ `malloc` ใน wake stub | ใช้ pre-allocated buffer | Wake stub ไม่มี heap |
| คาดหวังว่าโค้ดหลัง `esp_deep_sleep_start()` จะทำงาน | ย้ายโค้ดไป `app_main` และตรวจ wake reason | Deep sleep ตื่นแล้ว reset ใหม่ |
| ใช้ GPIO0 เป็น wake pin โดยไม่ระวัง | ระวัง boot mode conflict | GPIO0 เป็น strapping pin ของ boot mode |
| Custom bootloader ใช้ flash functions | ใช้ ROM functions | Bootloader ทำงานก่อน flash init |
| ลืม `CONFIG_BOOTLOADER_SKIP_RTC_MEM_INIT` | ตั้งค่านี้ถ้าต้องการรักษา RTC memory | Bootloader อาจ initialize RTC memory ใหม่ |
| ลืม release retention memory หลังใช้ | ปิด retention เมื่อไม่ใช้ | SRAM ที่ retention จองจะไม่พร้อมใช้งาน |
| ใช้ ext0 กับ GPIO ที่ไม่ใช่ RTC GPIO | เช็คว่าเป็น RTC GPIO | ext0 รองรับ RTC GPIO เท่านั้น |

---

## 17. แหล่งข้อมูลอ้างอิง

| หัวข้อ | แหล่งที่มา |
|---|---|
| Boot flow + boot mode selection | ESP32-S3 TRM v1.8 Chapter 8 |
| RTC memory types | ESP32-S3 TRM v1.8 Chapter 4 |
| Sleep modes + retention + RTC boot | ESP32-S3 TRM v1.8 Chapter 10 |
| RTC_CNTL registers | ESP32-S3 TRM v1.8 Chapter 10 |
| Strapping pins + boot mode | POM Datasheet v2.2 |
| ESP-IDF app startup flow | [ESP-IDF App Startup Flow — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/startup.html) |
| ESP-IDF sleep modes | [ESP-IDF Sleep Modes — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/sleep_modes.html) |
| ESP-IDF wake stub | [ESP-IDF Sleep Stubs](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/sleep_modes.html) |
| ESP-IDF RTC memory API | [ESP-IDF Misc System API](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/misc_system_api.html) |
| ESP-IDF reset reason | [ESP-IDF Reset Reason](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/misc_system_api.html) |
| ESP-IDF power management | [ESP-IDF Power Management](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/power_management.html) |
| ESP-IDF custom bootloader | [ESP-IDF Bootloader — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/bootloader.html) |
| Direct boot mode | [ESP-IDF Boot — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/boot.html) |
| Strapping pins discussion | [ESP32 Forum — Strapping Pins](https://esp32.com/viewtopic.php?t=9893) |