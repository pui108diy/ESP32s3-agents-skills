---
name: esp32s3-memory-map
version: 1.0.0
description: Memory Map skill สำหรับ ESP32-S3 — ครอบคลุม address ranges ทุก region, cache configuration, PMS permission control, และกฎการเข้าถึง memory อย่างปลอดภัย
triggers:
  - memory map
  - address
  - SRAM
  - ROM
  - peripheral register
  - base address
  - PSRAM
  - external RAM
  - flash
  - cache
  - ICache
  - DCache
  - permission
  - PMS
  - RTC memory
  - deep sleep
---

# Skill: Memory Map for ESP32-S3

> **ขอบเขต:** ใช้สำหรับเขียน C code บน ESP32-S3 ที่ต้องเข้าถึง memory โดยตรงหรือเข้าใจ address layout
>
> **แหล่งข้อมูลหลัก:** ESP32-S3 TRM v1.8 Chapter 4 (System and Memory) + Chapter 15 (Permission Control)

---

## 1. ภาพใหญ่: แผนผังหน่วยความจำ ESP32-S3

```
0x0000_0000 ─────────────────┐
                              │  Reserved
0x3BFF_FFFF ─────────────────┤
0x3C00_0000 ─────────────────┤  External RAM (DCache)     32 MB
0x3DFF_FFFF ─────────────────┤
0x3E00_0000 ─────────────────┤
                              │  Reserved / Internal memory (Data bus)
0x3FC8_7FFF ─────────────────┤
0x3FC8_8000 ─────────────────┤  Internal SRAM 1 (Data)    416 KB
0x3FCE_FFFF ─────────────────┤
0x3FCF_0000 ─────────────────┤  Internal SRAM 2 (Data)   64 KB
0x3FCF_FFFF ─────────────────┤
0x3FD0_0000 ─────────────────┤
                              │  Reserved
0x3FEF_FFFF ─────────────────┤
0x3FF0_0000 ─────────────────┤  Internal ROM 1 (Data)     128 KB
0x3FF1_FFFF ─────────────────┤
0x3FF2_0000 ─────────────────┤
                              │  Reserved
0x3FFF_FFFF ─────────────────┤
0x4000_0000 ─────────────────┤  Internal ROM 0 (Instr)    256 KB
0x4003_FFFF ─────────────────┤
0x4004_0000 ─────────────────┤  Internal ROM 1 (Instr)    128 KB
0x4005_FFFF ─────────────────┤
0x4006_0000 ─────────────────┤
                              │  Reserved
0x4036_FFFF ─────────────────┤
0x4037_0000 ─────────────────┤  Internal SRAM 0 (Instr)   32 KB
0x4037_7FFF ─────────────────┤
0x4037_8000 ─────────────────┤  Internal SRAM 1 (Instr)  416 KB
0x403D_FFFF ─────────────────┤
0x403E_0000 ─────────────────┤
                              │  Reserved
0x41FF_FFFF ─────────────────┤
0x4200_0000 ─────────────────┤  External Flash (ICache)  32 MB
0x43FF_FFFF ─────────────────┤
0x4400_0000 ─────────────────┤
                              │  Reserved
0x4FFF_FFFF ─────────────────┤
0x5000_0000 ─────────────────┤  RTC SLOW Memory            8 KB
0x5000_1FFF ─────────────────┤
0x5000_2000 ─────────────────┤
                              │  Reserved
0x5FFF_FFFF ─────────────────┤
0x6000_0000 ─────────────────┤  Peripherals              836 KB
0x600D_0FFF ─────────────────┤
0x600D_1000 ─────────────────┤
                              │  Reserved
0x600F_DFFF ─────────────────┤
0x600F_E000 ─────────────────┤  RTC FAST Memory            8 KB
0x600F_FFFF ─────────────────┘
```

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 4 Table 4.3-1 (Internal Memory) + Table 4.3-2 (External Memory) + POM Figure 4-1

---

## 2. Internal Memory — ตาราง Address แบบละเอียด

### 2.1 Instruction Bus (IBUS) — รันโค้ดได้

| Region | Low Address | High Address | Size | Bus | หมายเหตุ |
|---|---|---|---|---|---|
| Internal ROM 0 | `0x4000_0000` | `0x4003_FFFF` | 256 KB | IBUS | Primary bootloader + library functions |
| Internal ROM 1 | `0x4004_0000` | `0x4005_FFFF` | 128 KB | IBUS | Library functions |
| Internal SRAM 0 | `0x4037_0000` | `0x4037_7FFF` | 32 KB | IBUS | 16KB หรือ 32KB สามารถเป็น ICache |
| Internal SRAM 1 | `0x4037_8000` | `0x403D_FFFF` | 416 KB | IBUS+DBUS | รันโค้ดได้ + เก็บข้อมูลได้ |

### 2.2 Data Bus (DBUS) — เก็บข้อมูล (รันโค้ดไม่ได้ ถ้า PMS เปิด)

| Region | Low Address | High Address | Size | Bus | หมายเหตุ |
|---|---|---|---|---|---|
| Internal ROM 1 | `0x3FF0_0000` | `0x3FF1_FFFF` | 128 KB | DBUS | Mirror ของ ROM 1 (read-only) |
| Internal SRAM 1 | `0x3FC8_8000` | `0x3FCE_FFFF` | 416 KB | DBUS | Mirror ของ SRAM 1 (data access) |
| Internal SRAM 2 | `0x3FCF_0000` | `0x3FCF_FFFF` | 64 KB | DBUS | 32KB หรือ 64KB สามารถเป็น DCache |

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 4 Table 4.3-1 + Chapter 15 Table 15.3-3 (SRAM Address by block)

### 2.3 RTC Memory — ยังคงข้อมูลใน Deep Sleep

| Region | Low Address | High Address | Size | หมายเหตุ |
|---|---|---|---|---|
| RTC SLOW Memory | `0x5000_0000` | `0x5000_1FFF` | 8 KB | CPU + ULP เข้าถึงได้ |
| RTC FAST Memory | `0x600F_E000` | `0x600F_FFFF` | 8 KB | CPU เท่านั้น (ULP ไม่ได้) |

**⚠️ คุณสมบัติ RTC Memory:**
- ยัง powered ใน Deep Sleep mode
- ใช้เก็บ "deep sleep wake stub" สูงสุด 8 KB
- RTC FAST: เก็บ instructions และ data ที่ต้องข้าม deep sleep
- RTC SLOW: แชร์ข้อมูลระหว่าง CPU และ ULP co-processor

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 4 (RTC Memory) + Chapter 10 Section 10.6 (RTC Boot)

---

## 3. External Memory — Flash และ PSRAM

| Region | Low Address | High Address | Size | Bus | ผ่าน |
|---|---|---|---|---|---|
| External Flash | `0x4200_0000` | `0x43FF_FFFF` | 32 MB | IBUS | ICache |
| External RAM (PSRAM) | `0x3C00_0000` | `0x3DFF_FFFF` | 32 MB | DBUS | DCache |

**⚠️ ข้อจำกัด:**
- CPU เข้าถึง external memory ผ่าน cache เท่านั้น (ไม่ได้ตรง)
- ICache: 4-byte aligned reads/fetches
- DCache: 1/2/4/16-byte aligned reads/writes
- สูงสุด 1 GB flash + 1 GB PSRAM (แต่ mapped ได้ครั้งละ 32 MB)
- DMA descriptor **ห้ามวางใน PSRAM** — ต้องวางใน internal SRAM

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 4 Section 4.3.3.1 + [ESP-IDF External RAM](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/external-ram.html)

### 3.1 การเปิดใช้ PSRAM ใน ESP-IDF

```
# idf.py menuconfig
Component config → Hardware Settings → SPIRAM → Enable
CONFIG_SPIRAM=y
CONFIG_SPIRAM_USE_MALLOC=y              # ให้ malloc ใช้ PSRAM ได้
CONFIG_SPIRAM_SPEED=80MHz               # หรือ 120MHz สำหรับ Octal PSRAM
CONFIG_SPIRAM_BOOT_INIT=y               # init ตอน boot
```

**⚠️ ข้อควรระวัง:**
- `CONFIG_SPIRAM_SPEED=120` ต้องเป็น Octal PSRAM เท่านั้น
- Stack ของ task **ไม่ควร**อยู่ใน PSRAM (ยกเว้นเปิด `CONFIG_SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY`)
- อุณหภูมิเปลี่ยนมาก → PSRAM อาจมี read/write error ที่ 120MHz

> แหล่งเว็บ: [ESP-IDF Flash/PSRAM Config](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/flash_psram_config.html)

---

## 4. Cache Configuration

| Cache | ขนาดที่เป็นได้ | แหล่งที่มา | Block Size | Set Associative |
|---|---|---|---|---|
| ICache | 16 KB หรือ 32 KB | SRAM 0 | 16B หรือ 32B | 4-way หรือ 8-way |
| DCache | 32 KB หรือ 64 KB | SRAM 2 | 16B, 32B หรือ 64B | 4-way |

**⚠️ กฎสำคัญ:**
- ICache = 32 KB → block size ห้ามเป็น 16B
- DCache = 64 KB → block size ห้ามเป็น 16B
- ICache และ DCache แชร์กันทั้งสอง core (dual-core-shared)
- พื้นที่ที่เป็น cache จะถูก "ยืม" จาก SRAM — CPU เข้าถึงพื้นที่นั้นไม่ได้อีก

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 4 Section 4.3.3.2 (Cache Structure)

---

## 5. Peripheral Base Address Table

| Peripheral | Base Address | Size | Reset Control Register |
|---|---|---|---|
| UART0 | `0x6000_0000` | 4 KB | `SYSTEM_UART_RST` |
| SPI1 | `0x6000_2000` | 4 KB | `SYSTEM_SPI01_RST` |
| SPI0 | `0x6000_3000` | 4 KB | `SYSTEM_SPI01_RST` |
| GPIO | `0x6000_4000` | 4 KB | — |
| eFuse Controller | `0x6000_7000` | 4 KB | — |
| RTC_CNTL (Low-Power Mgmt) | `0x6000_8000` | 4 KB | — |
| IO MUX | `0x6000_9000` | 4 KB | — |
| I2S0 | `0x6000_F000` | 4 KB | `SYSTEM_I2S0_RST` |
| UART1 | `0x6001_0000` | 4 KB | `SYSTEM_UART1_RST` |
| I2C0 | `0x6001_3000` | 4 KB | `SYSTEM_I2C_EXT0_RST` |
| UHCI0 | `0x6001_4000` | 4 KB | `SYSTEM_UHCI0_RST` |
| RMT (Remote Control) | `0x6001_6000` | 4 KB | `SYSTEM_RMT_RST` |
| PCNT | `0x6001_7000` | 4 KB | `SYSTEM_PCNT_RST` |
| LEDC | `0x6001_9000` | 4 KB | `SYSTEM_LEDC_RST` |
| MCPWM0 | `0x6001_E000` | 4 KB | — |
| Timer Group 0 | `0x6001_F000` | 4 KB | `SYSTEM_TIMERGROUP_RST` |
| Timer Group 1 | `0x6002_0000` | 4 KB | `SYSTEM_TIMERGROUP1_RST` |
| SPI2 | `0x6002_4000`(typical) | 4 KB | `SYSTEM_SPI2_RST` |
| I2S1 | (typical after SPI2) | 4 KB | `SYSTEM_I2S1_RST` |
| TWAI | — | 4 KB | — |
| USB-OTG | — | 4 KB | — |
| USB-Serial/JTAG | — | 4 KB | — |
| GDMA | `0x6003_F000`(typical) | 4 KB | — |
| ADC1/DAC | — | 4 KB | — |

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 4 Table 4.3-3 (Module/Peripheral Address Mapping) — ตารางเต็มอยู่ใน TRM

**⚠️ หมายเหตุ:** ตารางนี้เป็น base address — แต่ละ peripheral มี register offset เป็น multiples ของ 4 bytes

---

## 6. Permission Control (PMS) — ⚠️ สำคัญมาก

### 6.1 ภาพรวม

ESP32-S3 มี **Permission Control (PMS) module** ที่ควบคุม X (Execute), W (Write), R (Read) access แยกตาม:
- **Bus** (IBUS, DBUS)
- **World** (Secure, Non-secure)
- **Core** (CPU0, CPU1)
- **Region** (แบ่งแต่ละ memory type)

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 15 (Permission Control) + [ESP-IDF Security](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/security/security.html)

### 6.2 การตั้งค่าเริ่มต้นของ ESP-IDF

ESP-IDF startup code ตั้งค่า PMS ดังนี้:
- **Instruction memory** (ROM, SRAM via IBUS): R + X (อ่านและรันได้, แต่เขียนไม่ได้)
- **Data memory** (SRAM via DBUS): R + W (อ่านและเขียนได้, แต่รันโค้ดไม่ได้)
- **Peripheral registers**: R + W (อ่านและเขียนได้)

> แหล่งเว็บ: [ESP-IDF Security — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/security/security.html) — "ESP-IDF application startup code configures the permissions attributes like Read/Write access on data memories and Read/Execute access on instruction memories"

### 6.3 ผลของการละเมิด PMS

ถ้าโค้ดพยายาม:
- **เขียนลง instruction memory** (เช่น `0x4037_8000`) → PMS violation → **system panic**
- **รันโค้ดจาก data memory** (เช่น `0x3FC8_8000`) → PMS violation → **system panic**
- **เข้าถึง peripheral โดยไม่ได้รับอนุญาต** → PMS violation → **system panic**

### 6.4 การปิด PMS (ไม่แนะนำ)

```c
// ใน sdkconfig.defaults
// CONFIG_ESP_SYSTEM_MEMPROT_FEATURE=n    // ปิด memory protection
```

> ⚠️ **ห้ามปิด** เว้นแต่จำเป็นจริงๆ (เช่น self-modifying code หรือ dynamic code loading) — ปิดแล้วช่องโหว่ด้านความปลอดภัยเพิ่มขึ้น

> แหล่งเว็บ: [ESP32 Forum — PMS Discussion](https://esp32.com/viewtopic.php?t=29654)

### 6.5 PMS Region Splitting

| Memory Type | จำนวน Region | หน่วยน้อยสุด |
|---|---|---|
| Internal ROM | 2 (ROM0, ROM1) | — |
| Internal SRAM | แบ่งได้หลาย region | ดู Table 15.3-3 |
| External Flash | 4 regions | 64 KB each |
| External RAM | 4 areas | 4 KB units |

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 15 Section 15.5 (External Memory Permission Control)

---

## 7. SRAM Block Breakdown (สำหรับการเข้าถึงแบบละเอียด)

ESP32-S3 มี SRAM แบ่งเป็น block ย่อย แต่ละ block มีขนาดต่างกัน:

| Block | IBUS Address | DBUS Address | ขนาด |
|---|---|---|---|
| SRAM0 Block0 | `0x4037_0000` – `0x4037_3FFF` | — | 16 KB |
| SRAM0 Block1 | `0x4037_4000` – `0x4037_7FFF` | — | 16 KB |
| SRAM1 Block2 | `0x4037_8000` – `0x4037_FFFF` | `0x3FC8_8000` – `0x3FC8_FFFF` | 32 KB |
| SRAM1 Block3 | `0x4038_0000` – `0x4038_FFFF` | `0x3FC9_0000` – `0x3FC9_FFFF` | 64 KB |
| SRAM1 Block4 | `0x4039_0000` – `0x4039_FFFF` | `0x3FCA_0000` – `0x3FCA_FFFF` | 64 KB |
| SRAM1 Block5 | `0x403A_C000` – `0x403A_FFFF` | `0x3FCB_C000` – `0x3FCB_FFFF` | 16 KB |
| SRAM1 Block6 | `0x403B_0000` – `0x403B_FFFF` | `0x3FCC_0000` – `0x3FCC_FFFF` | 64 KB |
| SRAM1 Block7 | `0x403C_0000` – `0x403C_FFFF` | `0x3FCD_4000` – `0x3FCD_FFFF` | 64 KB |
| SRAM2 Block9 | — | — | 32 KB |
| SRAM2 Block10 | — | — | 32 KB |

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 15 Table 15.3-3 (SRAM Address)

---

## 8. Deep Sleep และ RTC Memory Boot

### 8.1 RTC Memory ยัง powered ใน Deep Sleep

```
Deep Sleep Mode:
  ┌──────────────────────────┐
  │ CPU, SRAM, ROM → powered down (สูญเสียข้อมูล)
  │ RTC FAST Memory → ยัง powered (เก็บข้อมูลได้)
  │ RTC SLOW Memory → ยัง powered (เก็บข้อมูลได้)
  └──────────────────────────┘
```

### 8.2 Boot จาก RTC Memory (Wake Stub)

**Method 1: Boot จาก RTC SLOW Memory**
1. เซ็ต `RTC_CNTL_PROCPU_STAT_VECTOR_SEL = 0`
2. เข้า sleep
3. CPU ตื่นแล้ว reset vector เริ่มที่ `0x5000_0000` (ไม่ผ่าน SPI booting)
4. โค้ดใน RTC SLOW Memory รันทันที

**Method 2: Boot จาก RTC FAST Memory**
1. เซ็ต `RTC_CNTL_PROCPU_STAT_VECTOR_SEL = 1`
2. คำนวณ CRC ของ RTC FAST Memory → เก็บใน `RTC_CNTL_RTC_STORE7_REG[31:0]`
3. เซ็ต entry address ใน `RTC_CNTL_RTC_STORE6_REG[31:0]`
4. เข้า sleep
5. ตื่น → ROM unpacking → คำนวณ CRC อีกครั้ง → ถ้าตรง → กระโดดไป entry address

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 10 Section 10.6 (RTC Boot)

### 8.3 Deep Sleep Retention (SRAM)

- สามารถเก็บ CPU register state ใน SRAM ก่อนเข้า sleep และ restore ตอนตื่น
- ใช้ `RTC_CNTL_RETENTION_EN` field ใน `RTC_CNTL_RETENTION_CTRL_REG`
- ต้องจองพื้นที่ SRAM อย่างน้อย 432 words (428 CPU regs + 4 config words)

> แหล่งข้อมูล: ESP32-S3 TRM Chapter 10 (Retention function)

---

## 9. กฎการเข้าถึง Memory อย่างปลอดภัย

### 9.1 กฎพื้นฐาน

1. **ห้ามเขียนลง IBUS address** (เช่น `0x4037_8000`) ถ้า PMS เปิดอยู่ → system panic
2. **ห้าม jump/call ไป DBUS address** (เช่น `0x3FC8_8000`) ถ้า PMS เปิดอยู่ → system panic
3. **DMA descriptor ต้องอยู่ใน internal SRAM** — ห้ามวางใน PSRAM
4. **DMA ไป PSRAM** ต้องใช้ base address ที่ aligned ตามข้อกำหนด (ดู TRM Chapter 3)
5. **RTC memory ใช้สำหรับ deep sleep** — อย่าใช้เป็น general-purpose RAM ในโหมดปกติ

### 9.2 การประกาศตัวแปรใน memory region เฉพาะ

```c
// วางใน RTC SLOW Memory (เก็บข้าม deep sleep)
RTC_NOINIT_ATTR static uint32_t rtc_counter;

// วางใน RTC FAST Memory
RTC_NOINIT_ATTR static uint8_t rtc_fast_buffer[1024];

// วางใน PSRAM (ต้องเปิด CONFIG_SPIRAM)
EXT_RAM_BSS_ATTR static uint8_t psram_buffer[8192];

// วางใน IRAM (instruction RAM, ทำงานเร็ว)
IRAM_ATTR void my_fast_function(void) { /* ... */ }

// วางใน DRAM (data RAM)
DRAM_ATTR static uint8_t dram_buffer[256];
```

---

## 10. Checklist ก่อนเขียนโค้ดที่เข้าถึง Memory

- [ ] ยืนยันว่า address อยู่ใน region ที่ถูกต้อง (IBUS vs DBUS)
- [ ] ถ้าเขียน register-level driver: ใช้ base address จากตาราง Section 5
- [ ] ถ้าใช้ DMA: descriptor ต้องอยู่ใน internal SRAM (ไม่ใช่ PSRAM)
- [ ] ถ้าใช้ RTC memory: ประกาศด้วย `RTC_NOINIT_ATTR`
- [ ] ถ้าใช้ PSRAM: เปิด `CONFIG_SPIRAM` ใน menuconfig
- [ ] อย่าพยายามเขียนหรือ execute memory ใน region ที่ PMS ห้าม
- [ ] ถ้าจำเป็นต้อง execute จาก data memory: ปิด PMS ด้วยความระมัดระวัง
- [ ] ถ้าใช้ deep sleep wake stub: เก็บโค้ดใน RTC FAST หรือ RTC SLOW memory

---

## 11. Common Mistakes ที่ต้องหลีกเลี่ยง

| ผิด | ถูก | เหตุผล |
|---|---|---|
| เขียนลง `0x4037_8000` (IBUS) | เขียนที่ `0x3FC8_8000` (DBUS mirror) | IBUS = R/X only, DBUS = R/W |
| วาง DMA descriptor ใน PSRAM | วางใน internal SRAM | DMA descriptor ต้องอยู่ใน internal memory |
| ใช้ `0x3FC8_8000` เป็น code address | ใช้ `0x4037_8000` (IBUS) | Code ต้องรันจาก IBUS address |
| ปิด PMS โดยไม่จำเป็น | ปล่อย PMS เปิด | ลดช่องโหว่ด้านความปลอดภัย |
| ใช้ RTC memory ในโหมดปกติ | ใช้เฉพาะ deep sleep | RTC memory มีจำกัด (8KB each) |
| ตั้ง PSRAM 120MHz ธรรมดา | ใช้ Octal PSRAM | 120MHz ต้องเป็น Octal เท่านั้น |
| Stack ของ task อยู่ใน PSRAM | Stack อยู่ใน internal SRAM | PSRAM ช้ากว่า + cache issue |

---

## 12. แหล่งข้อมูลอ้างอิง

| หัวข้อ | แหล่งที่มา |
|---|---|
| Internal memory address | ESP32-S3 TRM v1.8 Chapter 4 Table 4.3-1 |
| External memory address | ESP32-S3 TRM v1.8 Chapter 4 Table 4.3-2 |
| SRAM block breakdown | ESP32-S3 TRM v1.8 Chapter 15 Table 15.3-3 |
| Peripheral base address | ESP32-S3 TRM v1.8 Chapter 4 Table 4.3-3 |
| Cache configuration | ESP32-S3 TRM v1.8 Chapter 4 Section 4.3.3.2 |
| Permission Control (PMS) | ESP32-S3 TRM v1.8 Chapter 15 |
| RTC Memory & Deep Sleep | ESP32-S3 TRM v1.8 Chapter 10 Section 10.6 |
| PSRAM configuration | [ESP-IDF External RAM Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/external-ram.html) |
| Flash/PSRAM config | [ESP-IDF Flash/PSRAM Config](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/flash_psram_config.html) |
| PMS overview | [ESP-IDF Security — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/security/security.html) |
| Memory Map visual guide | [ESP32 Memory Map 101](https://developer.espressif.com/blog/esp32-memory-map-101/) |
