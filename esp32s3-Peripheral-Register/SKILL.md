---
name: esp32s3-peripheral-register
version: 1.0.0
description: Peripheral Register skill สำหรับ ESP32-S3 — ครอบคลุม ESP-IDF layer architecture, register access macros, GPIO matrix/IO MUX configuration, clock gating/reset control, และ driver pattern สำหรับ UART, SPI, I2C, I2S, GDMA, Timer, MCPWM, RMT, LEDC, PCNT
triggers:
  - peripheral
  - register
  - GPIO matrix
  - IO MUX
  - REG_WRITE
  - REG_READ
  - SET_PERI_REG_MASK
  - clock gating
  - reset
  - driver
  - UART
  - SPI
  - I2C
  - I2S
  - GDMA
  - DMA
  - Timer
  - MCPWM
  - RMT
  - LEDC
  - PCNT
  - ADC
  - DAC
  - TWAI
  - USB
  - bare metal
---

# Skill: Peripheral Register for ESP32-S3

> **ขอบเขต:** ใช้สำหรับเขียน C code ที่เข้าถึง peripheral ทั้งแบบ high-level ESP-IDF driver API และแบบ bare-metal register-level
>
> **แหล่งข้อมูลหลัก:** ESP32-S3 TRM v1.8 (Chapters 3, 6, 17, 26, 27) + [ESP-IDF Programming Guide — Peripherals](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/index.html)

---

## 1. ESP-IDF Layer Architecture — 4 เลเยอร์

```
┌─────────────────────────────────────┐
│  Layer 1: Driver API (user-facing)  │  ← ใช้ใน application code
│  เช่น uart_driver_install(), i2c_master_write() │
├─────────────────────────────────────┤
│  Layer 2: HAL (Hardware Abstraction) │  ← abstraction ข้าม chip variants
│  เช่น uart_hal_init()                │
├─────────────────────────────────────┤
│  Layer 3: LL (Low-Level)            │  ← เข้าถึง register โดยตรง
│  เช่น uart_ll_write()                │
├─────────────────────────────────────┤
│  Layer 4: Register Macros           │  ← REG_WRITE, SET_PERI_REG_MASK
│  อยู่ใน soc/soc.h, soc/xxx_reg.h     │
└─────────────────────────────────────┘
```

> **แหล่งเว็บ:** [ESP32 Forum — Layer Architecture](https://www.esp32.com/viewtopic.php?t=36775) — "the IDF is structured into layers: the user-facing API layer (driver), which makes use of the underlying hardware abstraction layer (HAL), which in turn uses the low-level (LL) layer, which then sometimes uses specific macros to generate valid accesses to the hardware registers"

**⚠️ คำแนะนำการเลือกเลเยอร์:**
- 95% ของโค้ด: ใช้ **Layer 1 (Driver API)** — portable, thread-safe, มี error handling
- Custom driver หรือ bare-metal: ใช้ **Layer 3–4 (LL + Register Macros)**
- ห้าม skip ขั้นตอน clock enable + reset release ก่อนเข้าถึง register ใดๆ

---

## 2. Register Access Macros — พื้นฐาน

### 2.1 Macros หลัก (จาก `soc/soc.h`)

```c
#include "soc/soc.h"
#include "soc/uart_reg.h"    // สำหรับ UART
#include "soc/gpio_reg.h"    // สำหรับ GPIO
#include "soc/spi_reg.h"     // สำหรับ SPI
// ... แต่ละ peripheral มีไฟล์ _reg.h ของตัวเอง
```

| Macro | หน้าที่ |
|---|---|
| `REG_READ(addr)` | อ่าน 32-bit register |
| `REG_WRITE(addr, val)` | เขียน 32-bit register |
| `SET_PERI_REG_MASK(reg, mask)` | เซ็ต bit ที่ระบุใน mask (OR) |
| `CLEAR_PERI_REG_MASK(reg, mask)` | เคลียร์ bit ที่ระบุใน mask (AND NOT) |
| `SET_PERI_REG_BITS(reg, bitmap, shift, offset)` | เขียน field หลาย bit |
| `GET_PERI_REG_BITS(reg, bitmap, shift)` | อ่าน field หลาย bit |
| `SET_PERI_REG_BIT(reg, bit)` | เซ็ต 1 bit |
| `CLEAR_PERI_REG_BIT(reg, bit)` | เคลียร์ 1 bit |
| `READ_PERI_REG(reg)` | อ่าน register (SMP-safe version) |
| `WRITE_PERI_REG(reg, val)` | เขียน register (SMP-safe version) |

> **แหล่งเว็บ:** [soc/soc.h](https://github.com/espressif/esp-idf/blob/master/components/soc/esp32/include/soc/soc.h)

### 2.2 ตัวอย่างการใช้งาน

```c
// เซ็ต bit 5 ของ UART_INT_ENA_REG
SET_PERI_REG_BIT(UART_INT_ENA_REG(0), UART_RXFIFO_FULL_INT_ENA);

// เคลียร์ bit ของ UART interrupt
CLEAR_PERI_REG_MASK(UART_INT_ENA_REG(0), UART_TXFIFO_EMPTY_INT_ENA);

// เขียน field 3 bit ที่ offset 4
SET_PERI_REG_BITS(UART_CONF0_REG(0), 0x7, 0x5, 4);

// อ่าน field 2 bit ที่ offset 0
uint32_t val = GET_PERI_REG_BITS(UART_STATUS_REG(0), 0x3, 0);
```

**⚠️ กฎสำคัญ (IDF v5.0+):** Register macros ที่เขียนหรือ read-modify-write **ห้ามใช้เป็น expression** ต้องใช้เป็น statement เท่านั้น

```c
// ❌ ผิด (IDF v5.0+)
uint32_t val = REG_WRITE(addr, 0x10) + 1;

// ✅ ถูก
REG_WRITE(addr, 0x10);
uint32_t val = REG_READ(addr) + 1;
```

> **แหล่งเว็บ:** [ESP-IDF v5.0 Migration — Peripherals](https://docs.espressif.com/projects/esp-idf/en/v5.0/esp32s3/migration-guides/release-5.x/peripherals.html) — "register access macros which write or read-modify-write the register can no longer be used as expressions, and can only be used as statements"

---

## 3. Clock Gating และ Reset Control — ⚠️ ทำก่อนเสมอ

### 3.1 ขั้นตอนเริ่มต้นใช้งาน Peripheral ใดๆ

```
1. เปิด clock → SYSTEM_PERIP_CLK_ENx_REG
2. ปล่อยจาก reset → SYSTEM_PERIP_RST_ENx_REG
3. (ถ้าจำเป็น) กำหนด clock source และ divider ของ peripheral
4. กำหนด GPIO ผ่าน GPIO Matrix / IO MUX
5. กำหนด interrupt (ถ้าใช้)
6. เริ่มทำงาน peripheral
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 17 — "Before using any of these peripherals, it is mandatory to enable the clock for the given peripheral and release the peripheral from reset state"

### 3.2 Clock Enable Register — ตาราง Bit Position

#### SYSTEM_PERIP_CLK_EN0_REG (0x0018)

| Bit | Peripheral | Macro |
|---|---|---|
| 1 | SPI01 | `SYSTEM_SPI01_CLK_EN` |
| 2 | UART0 | `SYSTEM_UART_CLK_EN` |
| 4 | I2S0 | `SYSTEM_I2S0_CLK_EN` |
| 5 | UART1 | `SYSTEM_UART1_CLK_EN` |
| 6 | SPI2 | `SYSTEM_SPI2_CLK_EN` |
| 7 | I2C0 | `SYSTEM_I2C_EXT0_CLK_EN` |
| 8 | UHCI0 | `SYSTEM_UHCI0_CLK_EN` |
| 9 | RMT | `SYSTEM_RMT_CLK_EN` |
| 10 | PCNT | `SYSTEM_PCNT_CLK_EN` |
| 11 | LEDC | `SYSTEM_LEDC_CLK_EN` |
| 13 | Timer Group 0 | `SYSTEM_TIMERGROUP_CLK_EN` |
| 15 | Timer Group 1 | `SYSTEM_TIMERGROUP1_CLK_EN` |
| 16 | SPI3 | `SYSTEM_SPI3_CLK_EN` |
| 17 | I2C1 | `SYSTEM_I2C_EXT1_CLK_EN` |
| 18 | MCPWM0 | `SYSTEM_PWM0_CLK_EN` |
| 19 | TWAI (CAN) | `SYSTEM_CAN_CLK_EN` |
| 20 | MCPWM1 | `SYSTEM_PWM1_CLK_EN` |
| 21 | I2S1 | `SYSTEM_I2S1_CLK_EN` |
| 22 | USB | `SYSTEM_USB_CLK_EN` |
| 24 | UART Mem | `SYSTEM_UART_MEM_CLK_EN` |

#### SYSTEM_PERIP_CLK_EN1_REG (0x001C)

| Bit | Peripheral | Macro |
|---|---|---|
| 0 | Peri Backup | `SYSTEM_PERI_BACKUP_CLK_EN` |
| 1 | AES | `SYSTEM_CRYPTO_AES_CLK_EN` |
| 2 | SHA | `SYSTEM_CRYPTO_SHA_CLK_EN` |
| 3 | RSA | `SYSTEM_CRYPTO_RSA_CLK_EN` |
| 4 | DS | `SYSTEM_CRYPTO_DS_CLK_EN` |
| 5 | HMAC | `SYSTEM_CRYPTO_HMAC_CLK_EN` |
| 6 | GDMA | `SYSTEM_DMA_CLK_EN` |
| 7 | SDIO Host | `SYSTEM_SDIO_HOST_CLK_EN` |
| 8 | LCD/CAM | `SYSTEM_LCD_CAM_CLK_EN` |
| 9 | UART2 | `SYSTEM_UART2_CLK_EN` |
| 10 | USB Device | `SYSTEM_USB_DEVICE_CLK_EN` |
| 27 | ADC1 | `SYSTEM_APB_SARADC_CLK_EN` |
| 28 | ADC2 Arb | `SYSTEM_ADC2_ARB_CLK_EN` |
| 29 | System Timer | `SYSTEM_SYSTIMER_CLK_EN` |

### 3.3 Reset Register

`SYSTEM_PERIP_RST_EN0_REG` และ `SYSTEM_PERIP_RST_EN1_REG` มี bit position เหมือน clock enable รหัส:
- **เซ็ต bit = 1** → reset peripheral
- **เคลียร์ bit = 0** → release peripheral from reset

### 3.4 ตัวอย่าง: เปิดใช้งาน UART2

```c
// 1. เปิด clock
SET_PERI_REG_BIT(SYSTEM_PERIP_CLK_EN1_REG, SYSTEM_UART2_CLK_EN);
// 2. Release from reset
CLEAR_PERI_REG_MASK(SYSTEM_PERIP_RST_EN1_REG, SYSTEM_UART2_RST);
// 3. ตอนนี้ register ของ UART2 จึงจะเข้าถึงได้
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 17 Table 17.3-2 (Peripheral Clock Gating and Reset Bits)

---

## 4. GPIO Matrix และ IO MUX — หัวใจของ ESP32-S3

### 4.1 ภาพรวม

```
                    ┌──────────────┐
  External Pin ←──→ │   IO MUX     │ ←── F0 (GPIO func)
                    │              │ ←── F1–F4 (direct peripheral)
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
                    │  GPIO Matrix │ ←── 175 input signals
                    │  (full-switch)│ ──→ 184 output signals
                    └──────────────┘
                           ↑↓
                    ┌──────────────┐
                    │  Peripheral  │ (UART, SPI, I2C, ...)
                    └──────────────┘
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 6 + [ESP-IDF GPIO Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/gpio.html) — "The ESP32-S3 chip features 45 physical GPIO pins (GPIO0 ~ GPIO21 and GPIO26 ~ GPIO48). Each pin can be used as a general-purpose I/O, or be connected to an internal peripheral signal. Through GPIO matrix, IO MUX, and RTC IO MUX, peripheral input signals can be from any GPIO pin, and peripheral output signals can be routed to any GPIO pin."

### 4.2 IO MUX — Direct Connection (เร็วกว่าแต่จำกัด)

- แต่ละ pin มี **F0–F4** (5 functions)
- F0 = GPIO function (ผ่าน GPIO Matrix)
- F1–F4 = peripheral direct (เร็วกว่า, สำหรับสัญญาณความถี่สูง เช่น SPI 80MHz)
- ตัวอย่าง: GPIO12 F1 = FSPIIO6, GPIO12 F2 = SUBSPID

> **แหล่งเว็บ:** [Waveshare ESP32 ESP-IDF Tutorials](https://docs.waveshare.com/ESP32-ESP-IDF-Tutorials/Peripheral) — "Direct signals from high-speed peripherals (such as SPI, JTAG, UART, etc.) for better high-frequency performance, though with less flexibility"

### 4.3 GPIO Matrix — Flexible Routing (ยืดหยุ่นแต่ช้ากว่า)

**Input routing (External Pin → Peripheral):**
```c
// ตั้งค่า peripheral input signal Y ให้รับจาก GPIO pin X
REG_WRITE(GPIO_FUNCy_IN_SEL_CFG_REG, 
    (1 << GPIO_SIGy_IN_SEL_S) |        // เปิดใช้ GPIO matrix routing
    (pin_number << GPIO_FUNCy_IN_SEL_S) // เลือก pin
);
```

**Output routing (Peripheral → External Pin):**
```c
// ตั้งค่า peripheral output signal ให้ส่งไป GPIO pin X
REG_WRITE(GPIO_FUNCn_OUT_SEL_CFG_REG(pin), 
    (signal_index << GPIO_FUNCn_OUT_SEL_S));
// แล้วตั้ง IO MUX ของ pin เป็น GPIO function (F0)
REG_WRITE(IO_MUX_GPIOx_REG(pin), 
    (1 << IO_MUX_FUN_IE_S) |   // input enable
    (1 << IO_MUX_FUN_OE_S) |   // output enable
    (1 << IO_MUX_FUN_WPU_S) | // pull-up (ถ้าต้องการ)
    (0 << IO_MUX_MCU_SEL_S));  // F0 = GPIO function
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 6 Table 6.11-1 (Peripheral Signals via GPIO Matrix) — มีรายการ 175 input signals + 184 output signals

### 4.4 ESP-IDF High-Level API (แนะนำ)

```c
#include "driver/gpio.h"

// กำหนด pin เป็น output
gpio_config_t io_conf = {
    .pin_bit_mask = (1ULL << GPIO_NUM_2),
    .mode = GPIO_MODE_OUTPUT,
    .pull_up_en = GPIO_PULLUP_DISABLE,
    .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type = GPIO_INTR_DISABLE,
};
gpio_config(&io_conf);

// ส่งค่า
gpio_set_level(GPIO_NUM_2, 1);

// กำหนด interrupt
io_conf.intr_type = GPIO_INTR_NEGEDGE;
gpio_config(&io_conf);
gpio_install_isr_service(0);
gpio_isr_handler_add(GPIO_NUM_2, my_isr_handler, NULL);
```

> **แหล่งเว็บ:** [ESP-IDF GPIO Guide — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/gpio.html)

---

## 5. UART Driver Pattern

### 5.1 High-Level ESP-IDF API (แนะนำ)

```c
#include "driver/uart.h"

#define UART_PORT UART_NUM_0
#define BUF_SIZE 1024

void uart_init(void) {
    uart_config_t config = {
        .baud_rate = 115200,
        .data_bits = UART_DATA_8_BITS,
        .parity = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
        .source_clk = UART_SCLK_APB,
    };
    uart_driver_install(UART_PORT, BUF_SIZE * 2, BUF_SIZE * 2, 0, NULL, 0);
    uart_param_config(UART_PORT, &config);
    uart_set_pin(UART_PORT, 1, 3, -1, -1);  // TX=GPIO1, RX=GPIO3
}
```

> **แหล่งเว็บ:** [ESP-IDF UART Guide — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/uart.html) + [controllerstech.com](https://controllerstech.com/how-to-use-uart-in-esp32-esp-idf/)

### 5.2 Register-Level Configuration (bare-metal)

```c
#include "soc/uart_reg.h"
#include "soc/system_reg.h"

#define UART0_BASE 0x6000_0000

void uart_baremetal_init(void) {
    // 1. Enable clock + release reset (อยู่ใน UART0 แล้วเปิดอยู่ตาม default)

    // 2. Configure baud rate — divider = APB_CLK / baud
    //    สำหรับ 115200 bps จาก APB 80MHz: divider ≈ 694
    REG_WRITE(UART_CLKDIV_REG(0), 694);  // integer part
    REG_WRITE(UART_CLKDIV_REG(0), 
        (694 << UART_CLKDIV_S) | (0 << UART_CLKDIV_FRAG_S));

    // 3. Configure frame: 8N1
    uint32_t conf0 = REG_READ(UART_CONF0_REG(0));
    conf0 &= ~(UART_BIT_NUM_M);      // clear data bits
    conf0 |= (0x3 << UART_BIT_NUM_S); // 8-bit (0b11 = 8 bits)
    conf0 &= ~(UART_PARITY_EN);      // no parity
    conf0 &= ~(UART_STOP_BIT_NUM_M);  // 1 stop bit (0b1)
    conf0 |= (1 << UART_STOP_BIT_NUM_S);
    REG_WRITE(UART_CONF0_REG(0), conf0);

    // 4. Route via GPIO Matrix — TX→GPIO1, RX→GPIO3
    REG_WRITE(GPIO_FUNC1_OUT_SEL_CFG_REG, U0TXD_OUT_IDX); // output
    REG_WRITE(GPIO_FUNC3_IN_SEL_CFG_REG,                   // input
        (1 << GPIO_SIG3_IN_SEL_S) | (U0RXD_IN_IDX << GPIO_FUNC3_IN_SEL_S));
    // 5. Configure IO MUX ของ GPIO1 และ GPIO3
    REG_WRITE(IO_MUX_GPIO1_REG, (1 << IO_MUX_FUN_OE_S) | (0 << IO_MUX_MCU_SEL_S));
    REG_WRITE(IO_MUX_GPIO3_REG, (1 << IO_MUX_FUN_IE_S) | (0 << IO_MUX_MCU_SEL_S));
}
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 26 — UART Controller: register layout + Section 26.5 Configuration Steps

---

## 6. I2C Driver Pattern

### 6.1 High-Level ESP-IDF API (v5.x — new master driver)

```c
#include "driver/i2c_master.h"

i2c_master_bus_config_t bus_config = {
    .i2c_port = I2C_NUM_0,
    .sda_io_num = 8,
    .scl_io_num = 9,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .glitch_ignore_cnt = 7,
    .flags.enable_internal_pullup = true,
};
i2c_master_bus_handle_t bus_handle;
i2c_new_master_bus(&bus_config, &bus_handle);

i2c_device_config_t dev_config = {
    .dev_addr_length = I2C_ADDR_BIT_LEN_7,
    .device_address = 0x68,  // MPU6050
    .scl_speed_hz = 400000,
};
i2c_master_dev_handle_t dev_handle;
i2c_master_bus_add_device(bus_handle, &dev_config, &dev_handle);

// Write
uint8_t buf[] = {0x75, 0x00};
i2c_master_write_read_device(dev_handle, 0x75, buf, 1, buf+1, 1, -1);
```

> **แหล่งเว็บ:** [ESP-IDF I2C Guide — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/i2c.html)

**⚠️ ข้อสำคัญ:** ESP32-S3 I2C รองรับ Standard (100 kHz), Fast (400 kHz), และ up to 800 kHz

> **แหล่งข้อมูล:** POM — "Standard mode (100 kbit/s), Fast mode (400 kbit/s), Up to 800"

---

## 7. SPI Driver Pattern

### 7.1 High-Level ESP-IDF API

```c
#include "driver/spi_master.h"

spi_device_handle_t spi;
spi_device_interface_config_t devcfg = {
    .clock_speed_hz = 10 * 1000 * 1000,  // 10 MHz
    .mode = 0,                            // CPOL=0, CPHA=0
    .spics_io_num = 5,
    .queue_size = 7,
    .flags = SPI_DEVICE_HALFDUPLEX,
};
spi_bus_add_device(SPI2_HOST, &devcfg, &spi);

// Transaction
spi_transaction_t t = {
    .length = 8,
    .tx_buffer = &data,
};
spi_device_polling_transmit(spi, &t);
```

> **แหล่งเว็บ:** [ESP-IDF SPI Master — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/spi_master.html)

**⚠️ SPI frequency limits:**
- ผ่าน GPIO Matrix: สูงสุด ~40 MHz
- ผ่าน IO MUX (direct): สูงสุด 80 MHz

> **แหล่งเว็บ:** [Waveshare ESP32 ESP-IDF Tutorials](https://docs.waveshare.com/ESP32-ESP-IDF-Tutorials/Peripheral) — "Introduces some signal delay and has frequency limitations for high-speed signals (e.g., up to 40MHz for SPI, and up to 80MHz for IO_MUX)"

---

## 8. GDMA (General DMA) Pattern

### 8.1 Overview

GDMA มี 5 channels (0–4) แต่ละ channel มี TX (OUT) และ RX (IN):
- **TX (OUT):** Memory → Peripheral
- **RX (IN):** Peripheral → Memory
- แต่ละ channel รองรับ peripheral ที่กำหนดใน `GDMA_IN/OUT_PERI_SEL_CHn_REG`

| Peripheral ID | Peripheral |
|---|---|
| 0 | SPI2 |
| 1 | SPI3 |
| 2 | UHCI0 |
| 3 | I2S0 |
| 4 | I2S1 |
| 5 | LCD/CAM |
| 6 | AES |
| 7 | SHA |
| 8 | ADC |
| 9 | RMT |

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 3 Section 3.4.3

### 8.2 Linked List Descriptor Structure (3 words each)

```c
struct gdma_descriptor_t {
    uint32_t dw0;  // [31:24] size, [30] suc_eof, [29:21] (reserved), [20:0] length
    uint32_t dw1;  // buffer address (must be in internal RAM!)
    uint32_t dw2;  // next descriptor address (0 if last)
};
```

**⚠️ กฎสำคัญ:**
- Descriptor **ต้องอยู่ใน internal SRAM** — ห้ามวางใน PSRAM
- Buffer address อยู่ได้ใน internal SRAM หรือ PSRAM
- ถ้าใช้ PSRAM: ต้องใช้ base address ที่ aligned ตามข้อกำหนด DMA

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 3 — "Linked lists should be in internal RAM for the GDMA engine to be able to use them"

### 8.3 Configuration Steps

```
1. เปิด GDMA clock: SYSTEM_DMA_CLK_EN
2. เลือก peripheral ที่จะใช้ DMA: GDMA_OUT_PERI_SEL_CHn / GDMA_IN_PERI_SEL_CHn
3. เตรียม linked list descriptor ใน internal SRAM
4. กำหนด address ของ descriptor ตัวแรก: GDMA_OUTLINK_ADDR_CHn / GDMA_INLINK_ADDR_CHn
5. เปิดใช้งาน: GDMA_OUTLINK_START_CHn / GDMA_INLINK_START_CHn
6. (ถ้าต้องการ) ตั้งค่า owner bit check
7. รอ interrupt GDMA_OUT_EOF_CHn_INT หรือ GDMA_IN_SUC_EOF_CHn_INT
```

> **แหล่งข้อมูล:** ESP32-S3 TRM Chapter 3 Section 3.4.2 (Peripheral-to-Memory and Memory-to-Peripheral)

---

## 9. Timer Group Pattern

ESP32-S3 มี 2 Timer Groups แต่ละ group มี 2 timers (Timer00, Timer01, Timer10, Timer11) แต่ละตัว 54-bit:

```c
#include "driver/gptimer.h"

gptimer_config_t timer_config = {
    .clk_src = GPTIMER_CLK_SRC_APB,
    .direction = GPTIMER_COUNT_UP,
    .resolution_hz = 1000000,  // 1 MHz → 1 tick = 1 us
};
gptimer_handle_t timer;
gptimer_new_timer(&timer_config, &timer);

gptimer_event_callbacks_t cbs = {
    .on_alarm = my_timer_isr,
};
gptimer_register_event_callbacks(timer, &cbs, NULL);

gptimer_alarm_config_t alarm_config = {
    .alarm_count = 1000000,  // 1 second
    .flags.auto_reload_on_alarm = true,
};
gptimer_set_alarm_action(timer, &alarm_config);

gptimer_enable(timer);
gptimer_start(timer);
```

---

## 10. Other Peripherals — Quick Reference

### 10.1 I2S

```c
#include "driver/i2s_std.h"
// ESP32-S3 มี I2S0 + I2S1
// รองรับ standard, TDM, PDM modes
```

### 10.2 MCPWM (Motor Control PWM)

```c
#include "driver/mcpwm.h"
// MCPWM0 + MCPWM1 — แต่ละตัวมี 6 PWM outputs, fault detect, capture
```

### 10.3 RMT (Remote Control)

```c
#include "driver/rmt.h"
// ใช้สำหรับ IR, WS2812 LED, ฯลฯ — มี 4 TX + 4 RX channels
```

### 10.4 LEDC (LED PWM)

```c
#include "driver/ledc.h"
// 6 channels, 14-bit resolution, 8 timers (4 high-speed + 4 low-speed)
```

### 10.5 PCNT (Pulse Counter)

```c
#include "driver/pulse_cnt.h"
// 4 units, แต่ละ unit มี 2 channels, 16-bit counter
```

### 10.6 ADC (SAR ADC)

```c
#include "driver/adc.h"
// ADC1: 7 channels, ADC2: 3 channels (ขัดแย้งกับ WiFi)
// 12-bit resolution
```

### 10.7 TWAI (CAN)

```c
#include "driver/twai.h"
// ⚠️ ใช้ twai.h ไม่ใช่ can.h (CAN deprecated ใน IDF v5.0)
```

> **แหล่งเว็บ:** [ESP-IDF v5.0 Migration — Peripherals](https://docs.espressif.com/projects/esp-idf/en/v5.0/esp32s3/migration-guides/release-5.x/peripherals.html) — "The deprecated CAN peripheral driver is removed. Please use TWAI driver instead"

---

## 11. Interrupt Configuration Pattern

### 11.1 สำหรับ ESP-IDF High-Level API

```c
#include "esp_intr_alloc.h"

// กำหนด interrupt handler
esp_err_t ret = esp_intr_alloc(
    ETS_UART0_INTR_SOURCE,    // interrupt source
    ESP_INTR_FLAG_IRAM,       // flag
    uart_isr_handler,         // handler
    arg,                      // argument
    &handle                   // returned handle
);
```

### 11.2 Common Interrupt Sources

| Source | Macro |
|---|---|
| UART0 | `ETS_UART0_INTR_SOURCE` |
| UART1 | `ETS_UART1_INTR_SOURCE` |
| UART2 | `ETS_UART2_INTR_SOURCE` |
| SPI2 | `ETS_SPI2_INTR_SOURCE` |
| I2C0 | `ETS_I2C_EXT0_INTR_SOURCE` |
| Timer00 | `ETS_TG0_T0_LEVEL_INTR_SOURCE` |
| GPIO | `ETS_GPIO_INTR_SOURCE` |

---

## 12. Initialization Sequence Template — ทุก Peripheral

```c
// แม่แบบมาตรฐานสำหรับเริ่มต้น peripheral ใดๆ

void peripheral_init_template(void) {
    // 1. Enable peripheral clock
    SET_PERI_REG_BIT(SYSTEM_PERIP_CLK_ENx_REG, SYSTEM_xxx_CLK_EN);

    // 2. Release from reset
    CLEAR_PERI_REG_MASK(SYSTEM_PERIP_RST_ENx_REG, SYSTEM_xxx_RST);

    // 3. (ถ้าจำเป็น) เลือก clock source และ divider
    //    ใน xxx_CLK_CONF_REG ของ peripheral

    // 4. กำหนด GPIO ผ่าน GPIO Matrix / IO MUX
    //    (ตาม Section 4.3 หรือใช้ driver/gpio.h)

    // 5. กำหนด peripheral-specific configuration registers
    //    (ดู chapter ของ peripheral นั้นใน TRM)

    // 6. (ถ้าใช้ interrupt) เปิด interrupt ใน peripheral register
    //    SET_PERI_REG_BIT(xxx_INT_ENA_REG, xxx_INT_ENA);

    // 7. (ถ้าใช้ DMA) เตรียม descriptor + ตั้งค่า GDMA channel

    // 8. เปิดใช้งาน peripheral
    //    SET_PERI_REG_BIT(xxx_CMD_REG, xxx_EN);
}
```

---

## 13. Checklist ก่อนเขียน Peripheral Driver

- [ ] ยืนยันว่า peripheral clock เปิดแล้ว (`SYSTEM_PERIP_CLK_ENx`)
- [ ] ยืนยันว่า peripheral ถูก release from reset แล้ว (`SYSTEM_PERIP_RST_ENx`)
- [ ] ใช้ register macros จากไฟล์ `soc/xxx_reg.h` ที่ถูกต้อง
- [ ] Register macros ที่ write ใช้เป็น statement ไม่ใช่ expression (IDF v5.0+)
- [ ] GPIO routing: ตรวจสอบว่า signal index ถูกต้องตาม TRM Table 6.11-1
- [ ] ถ้าใช้ high-speed peripheral (SPI >40MHz): ใช้ IO MUX direct แทน GPIO Matrix
- [ ] ถ้าใช้ DMA: descriptor ต้องอยู่ใน internal SRAM
- [ ] ถ้าใช้ DMA ไป PSRAM: ตรวจสอบ address alignment
- [ ] ISR มี `IRAM_ATTR` และใช้ `...FromISR()` API (ดู FreeRTOS Skill)
- [ ] ใช้ TWAI แทน CAN (CAN deprecated ใน IDF v5.0+)

---

## 14. Common Mistakes ที่ต้องหลีกเลี่ยง

| ผิด | ถูก | เหตุผล |
|---|---|---|
| อ่าน/เขียน register โดยไม่เปิด clock | เปิด clock + release reset ก่อน | Register ไม่ตอบสนอง |
| ใช้ GPIO Matrix สำหรับ SPI 80MHz | ใช้ IO MUX direct | GPIO Matrix ช้าเกินไป |
| วาง DMA descriptor ใน PSRAM | วางใน internal SRAM | GDMA อ่าน descriptor จาก internal RAM เท่านั้น |
| ใช้ `can.h` | ใช้ `driver/twai.h` | CAN deprecated ใน IDF v5.0+ |
| Register macro เป็น expression | ใช้เป็น statement | Compile error ใน IDF v5.0+ |
| ใช้ `REG_WRITE` กับ DPORT register | ใช้ `DPORT_WRITE_PERI_REG` | DPORT ต้องใช้ SMP-safe variant |
| ลืม release reset หลังเปิด clock | ทำทั้งสองขั้นตอน | Peripheral ยังอยู่ใน reset state |
| ใช้ ADC2 ขณะ WiFi ทำงาน | ใช้ ADC1 หรือหยุด WiFi ชั่วคราว | ADC2 ขัดแย้งกับ WiFi |

---

## 15. แหล่งข้อมูลอ้างอิง

| หัวข้อ | แหล่งที่มา |
|---|---|
| GPIO Matrix + IO MUX | ESP32-S3 TRM v1.8 Chapter 6 |
| System registers (clock + reset) | ESP32-S3 TRM v1.8 Chapter 17 |
| GDMA Controller | ESP32-S3 TRM v1.8 Chapter 3 |
| UART Controller | ESP32-S3 TRM v1.8 Chapter 26 |
| I2C Controller | ESP32-S3 TRM v1.8 Chapter 27 |
| Peripheral base addresses | ESP32-S3 TRM v1.8 Chapter 4 Table 4.3-3 |
| Register access macros | [soc/soc.h](https://github.com/espressif/esp-idf/blob/master/components/soc/esp32/include/soc/soc.h) |
| IDF v5.0 register migration | [ESP-IDF v5.0 Migration Guide](https://docs.espressif.com/projects/esp-idf/en/v5.0/esp32s3/migration-guides/release-5.x/peripherals.html) |
| GPIO High-Level API | [ESP-IDF GPIO Guide — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/gpio.html) |
| UART High-Level API | [ESP-IDF UART Guide — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/uart.html) |
| I2C High-Level API | [ESP-IDF I2C Guide — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/i2c.html) |
| SPI High-Level API | [ESP-IDF SPI Master — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/spi_master.html) |
| Layer architecture | [ESP32 Forum — Layer Discussion](https://www.esp32.com/viewtopic.php?t=36775) |
| IO MUX vs GPIO Matrix | [Waveshare ESP32 ESP-IDF Tutorials](https://docs.waveshare.com/ESP32-ESP-IDF-Tutorials/Peripheral) |