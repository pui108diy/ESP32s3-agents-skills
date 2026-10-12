---
name: esp32s3-api-reference
version: 2.7.2
description: |
  Anti-hallucination API reference สำหรับ ESP32-S3 + ESP-IDF v5.2–v6.1 — ใช้เมื่อ: (1) เขียน/แก้/review โค้ด C ที่เรียก driver, WiFi/BT, HTTP, MQTT, NVS/SD, OTA, sleep หรือ DSP บน ESP32-S3 (2) ต้องยืนยัน function signature, struct field, หรือ header ที่ต้อง include สำหรับ IDF เวอร์ชันใดเวอร์ชันหนึ่ง (3) ตรวจโค้ดก่อนส่งมอบว่าไม่โดน กับดัก hallucination ที่พบบ่อยบนชิป/เวอร์ชันนี้
---

# ESP32-S3 API Reference — Router (v2.7.2)

> ไฟล์นี้คือ **Router** — อ่านไฟล์นี้ทุกครั้งก่อนทำงาน แล้ว**โหลดเฉพาะ** reference ที่งานต้องใช้ตาม §2
> ประวัติการแก้ไข → `CHANGELOG.md` · prompt ทดสอบ skill → `SMOKE-TESTS.md` · checklist กับดัก → `HALLUCINATION-HOT-LIST.md`

## 1. เมื่อไรต้องใช้ skill นี้
- กำลังเขียน/แก้/review โค้ด C ที่เรียก driver, WiFi/BT, HTTP (client/server), MQTT, NVS/SD, OTA, sleep หรือ DSP บน ESP32-S3
- ต้องยืนยัน function signature, struct field, หรือ header ที่ต้อง include สำหรับ IDF เวอร์ชันไหน

## 2. Dependency Map — งานนี้ต้องโหลดอะไรบ้าง
**กติกา:** โหลดเฉพาะ reference ที่ตรงกับงาน (ห้ามโหลดทั้งหมด) แล้วรัน checklist เฉพาะกลุ่ม API นั้นจาก `HALLUCINATION-HOT-LIST.md`

| ประเภทงาน | Reference ที่โหลด | Checklist ที่รัน |
|---|---|---|
| ดิจิทัล IO / ปุ่ม / ISR / คงค่าขาข้าม sleep | `peripherals/gpio.md` | GPIO |
| Serial / โมเด็ม AT / VFS | `peripherals/uart.md` | UART |
| Bus I2C (sensor, EEPROM, จอ) | `peripherals/i2c.md` | I2C |
| Bus SPI / จอ / flash ภายนอก | `peripherals/spi.md` | SPI |
| PWM / LED / servo พื้นฐาน / แทน DAC | `peripherals/ledc.md` | LEDC |
| Timer ฮาร์ดแวร์ / periodic แม่นยำ | `peripherals/gptimer.md` | GPTIMER |
| อ่านแรงดัน / sensor อนาล็อก | `peripherals/adc.md` | ADC |
| IR / NeoPixel / สัญญาณพัลส์ละเอียด | `peripherals/rmt.md` | RMT |
| เสียง / ไมค์ / ลำโพง / PDM | `peripherals/i2s.md` | I2S |
| Encoder / นับพัลส์ | `peripherals/pcnt.md` | PCNT |
| มอเตอร์ / H-bridge / dead-time | `peripherals/mcpwm.md` | MCPWM |
| CAN bus | `peripherals/twai.md` | TWAI |
| WiFi เชื่อมต่อ | `protocols/wifi.md` | WIFI (+ NVS) |
| เรียก REST API / ดาวน์โหลด | `protocols/http_client.md` | HTTP |
| MQTT | `protocols/mqtt.md` | MQTT |
| เปิด web server | `protocols/http_server.md` | HTTPD |
| เก็บค่า/settings ถาวร | `storage/nvs.md` | NVS |
| SD card / ไฟล์ | `storage/fatfs_sdmmc.md` | SD |
| callback เป็นรอบเวลา / one-shot | `system/esp_timer.md` | ESP_TIMER |
| ประหยัดพลังงาน / deep-light sleep | `system/sleep.md` | SLEEP (+ GPIO ถ้าเกี่ยว wakeup) |
| จอง memory / DMA buffer / PSRAM | `system/heap.md` | HEAP (+ `spi.md` §4 ถ้างาน DMA ของ SPI) |
| OTA / firmware update | `system/ota.md` | OTA |
| FFT / ตัวกรอง / คณิต | `dsp/esp_dsp.md` | DSP |

**คอมโบที่พบบ่อย:** audio → `i2s.md` + `heap.md` (DMA) · WiFi+MQTT → `wifi.md` + `mqtt.md` + `nvs.md` · ปลุกด้วยปุ่มจาก deep sleep → `sleep.md` + `gpio.md`

## 3. กฎเหล็ก 7 ข้อ
1. **เปิดไฟล์ reference ของ peripheral นั้นก่อนเขียนเสมอ** — ห้ามเขียน API จากความจำลอย ๆ
2. **คัดลอก signature ตรงตาม reference** — ห้ามเปลี่ยนชนิดพารามิเตอร์ ห้ามเติมพารามิเตอร์ที่ไม่มีจริง
3. **API ที่ไม่มีใน reference = ต้อง verify ก่อนใช้** กับ header จริง (`components/esp_driver_*/include/`) หรือ docs.espressif.com — แล้วเขียนผลเพิ่มกลับเข้า reference
4. **เช็ค ESP-IDF เวอร์ชันของโปรเจกต์ก่อน** (`idf.py --version`) แล้วจับคู่กับ §4 Version Decision Table
5. **โค้ดที่ส่งมอบต้องมี `#include` ครบ** และจัดการ error ด้วย `esp_err_t` / `ESP_ERROR_CHECK` ทุกจุด
6. **ก่อนส่งโค้ดทุกครั้ง → รัน `HALLUCINATION-HOT-LIST.md` เฉพาะกลุ่ม API ที่โค้ดใช้** (เลือกกลุ่มจาก §2) — เริ่มตรวจข้อ `[SILENT]` ก่อนเสมอเพราะตรวจจากอาการไม่ได้
7. **เจอป้าย ⚠️ = จุดที่ API เปลี่ยนระหว่างเวอร์ชัน** — ต้องเปิด header ของเวอร์ชันที่ใช้ยืนยันก่อน (คิวรวมอยู่ที่ §8 และท้ายไฟล์ reference แต่ละไฟล์)

## 4. Version Decision Table

| หัวข้อ | v4.x | v5.0–v5.1 | v5.2–v5.3 | v5.4–v5.5 | v6.x (stable = v6.1) |
|---|---|---|---|---|---|
| I2C driver | legacy `driver/i2c.h` | legacy (เริ่ม deprecate) | **new `driver/i2c_master.h`** — pullup อยู่ใน `.flags` **ตั้งแต่ v5.2** | **slave redesign ⚠️** (callbacks + `i2c_slave_write`, ลบ `i2c_slave_receive`/`*_ram`); `i2c_port = -1` auto ยืนยัน v5.4.1+; `multi_buffer_transmit` มีแล้ว | + `execute_defined_operations` (v5.5+); legacy = **EOL** (ลบใน v7); NACK → **`ESP_ERR_INVALID_RESPONSE`** ⚠️ |
| โครงสร้าง component | `components/driver` | เดิม | v5.3 เริ่มแยก `esp_driver_*` | แยกแล้ว | `esp_driver_uart/i2c/rmt/...` (header `driver/*.h` ยังชื่อเดิม) |
| `gpio_pad_select_gpio()` | ใช้ได้ | deprecated | deprecated | deprecated | **removed** → `gpio_reset_pin()` |
| `uart_read_bytes()` timeout | ticks (`TickType_t`) | ticks | ticks | ticks (ถึง v5.5 ยัง `TickType_t`) | **ยังเป็น ticks** — เปลี่ยนชนิดเป็น `uint32_t` เท่านั้น (ชื่อพารามิเตอร์ยัง `ticks_to_wait`) |
| UART macros / headers | ใช้ได้ | ใช้ได้ | ใช้ได้ | ใช้ได้ | **removed v6.0**: `UART_FIFO_LEN` → `UART_HW_FIFO_LEN`; `soc/uart_channel.h` → `soc/uart_pins.h` |
| `uart_set_pin()` | 5 args ตรง ๆ | เดิม | เดิม | เดิม | 5 args ยังใช้ได้ (wrapper) + `_uart_set_pin6` ฟอร์ม 7 args (DTR/DSR) |
| NVS `nvs_handle_t` | `uint32_t` | `uint32_t` | `uint32_t` | **opaque pointer** | opaque pointer |
| ADC/RMT/I2S/MCPWM/PCNT | legacy | **new drivers** มาแล้ว | new | new | legacy ถูกลบ |
| GPTimer (TIMG) | legacy `driver/timer.h` | new `driver/gptimer.h` | new — v5.3+ ต้อง `REQUIRES esp_driver_gptimer` | เดิม | legacy **ลบใน v6**; `set/get_raw_count` = `uint64_t` ทุกเวอร์ชัน; ETM = header แยก `gptimer_etm.h` |
| esp_timer (SYSTIMER) | ใช้ได้ | ใช้ได้ | ใช้ได้ — ต้อง `REQUIRES esp_timer` | เดิม | เดิม; **v6.1+** เพิ่มกลุ่ม `_at` + `stop_blocking`/`is_active`/`get_period`/`get_expiry_time`/`get_next_alarm_for_wake_up`; **ไม่มี `ESP_TIMER_EXT`**; `ESP_TIMER_ISR` ต้องเปิด `CONFIG_ESP_TIMER_SUPPORTS_ISR_DISPATCH_METHOD` |
| ADC atten (บน S3) | — | `ADC_ATTEN_DB_12` | เดิม | เดิม | เดิม (ระวัง `DB_11` จากโค้ด ESP32 รุ่นแรก) |
| MQTT config struct | flat (`.host`/`.port`) | **nested** (`.broker.address.uri`) | nested | nested | nested |
| ext1 wakeup | — | `esp_sleep_enable_ext1_wakeup` | เดิม | + variant ใหม่ `_io` | ตัวเดิม **deprecated ตั้งแต่ v6.0** → `esp_sleep_enable_ext1_wakeup_io()` |
| sleep cause type | — | `esp_sleep_wakeup_cause_t` | เดิม | เดิม | rename → **`esp_sleep_source_t`** (ชื่อ enum เดิม) |
| wakeup cause getter | `esp_sleep_get_wakeup_cause()` | เดิม | เดิม | เดิม | **deprecated ตั้งแต่ v6.0** → `esp_sleep_get_wakeup_causes()` (bitmap หลาย source พร้อมกัน) ⚠️ |
| UART wake threshold | — | `uart_set_wakeup_threshold()` | เดิม | เดิม | + ชุดใหม่ `uart_wakeup_setup()` + `uart_wakeup_cfg_t` ⚠️ |
| TWAI | classic driver | classic | classic | classic | v6.1: + **node-handle driver** (`twai_new_node`/`twai_node_transmit`) ⚠️ เลือกชุดให้ถูก |
| MCPWM dead-time | — | new driver (pipeline) | + API dead-time เฉพาะ ⚠️ | เดิม | เดิม — ตรวจ `mcpwm_gen.h` ของเวอร์ชันนั้น |
| SPI / LEDC / I2S CMakeLists | `driver` | `driver` | v5.3+ แยก `esp_driver_spi/ledc/i2s` | แยกแล้ว | legacy `driver` deprecated — ระบุ `esp_driver_*` ตรง (ดู reference แต่ละตัว) |
| I2S driver | legacy `driver/i2s.h` | legacy (deprecate v5.0) | + channel API ใหม่ | channel API | **legacy ลบใน v6.0** → `i2s_std/pdm/tdm.h` |
| LEDC | ใช้ได้ | ใช้ได้ | ใช้ได้ | ใช้ได้ | `ledc_timer_set` ลบ / `ledc_isr_register`+`intr_type` deprecated / macro clock เปลี่ยน (ดู ledc.md §9) |
| SPI master | ใช้ได้ | ใช้ได้ | ใช้ได้ | ใช้ได้ | signature คงเดิม; `CONFIG_SPI_MASTER_IN_IRAM` ซ่อน + ต้อง `CONFIG_FREERTOS_IN_IRAM` (ดู spi.md §9) |

## 5. File Index
```
esp32s3-api-reference/
├── SKILL.md                    ← ไฟล์นี้ — Router (v2.7.1)
├── HALLUCINATION-HOT-LIST.md   ← Checklist Edition v3.1 (105 ข้อ · 23 กลุ่ม API)
├── CHANGELOG.md                ← ประวัติการแก้ไขรายไฟล์ (แยกจาก SKILL.md ตั้งแต่ v2.7.0)
├── SMOKE-TESTS.md              ← prompt ทดสอบ skill (ย้ายจาก SKILL.md §9 เดิม)
└── references/                 (23 ไฟล์)
    ├── peripherals/            (12 ไฟล์)
    │   ├── gpio.md             (v2.2) ดิจิทัล IO, ISR, strapping, GPIO map, hold/wakeup, wake-cause
    │   ├── uart.md             (v2.1) 3 ports, ticks ทุกเวอร์ชัน, v6.0 breaking, event queue, VFS, wakeup
    │   ├── spi.md              (v2.1) SPI master + GDMA, 64-byte per-transaction, clock per path, DMA rules
    │   ├── i2c.md              (v2.1) new master + slave redesign, `.flags` ทุกเวอร์ชัน, probe, NACK v6
    │   ├── ledc.md             (v2.1) 8ch/4timer, fade, low-speed only, hpoint, clock global, duty/fade workflow
    │   ├── gptimer.md          (v2.0) 4× 54-bit TIMG + state machine enable→start
    │   ├── adc.md              (v2.0.1) 2×12-bit, GPIO↔CH mapping , ADC1 กับ WiFi,curve fitting (chan=reserved), DMA=ADC1 only
    │   ├── rmt.md              (v1.1) แปลงโครงสร้างแล้วรอreview 8ch TX0-3/RX4-7, encoder, 48 words/ch, +led_strip
    │   ├── i2s.md              (v2.1) channel driver, std/pdm/tdm, PDM=I2S0, DMA sizing
    │   ├── pcnt.md             (v1.1) แปลงโครงสร้างแล้วรอreview 4 units×2ch, glitch filter, encoder 4x
    │   ├── mcpwm.md            (v1.1) แปลงโครงสร้างแล้วรอreview 2 groups×3 timers/operators, pipeline, capture
    │   └── twai.md             (v1.1) แปลงโครงสร้างแล้วรอreview CAN 2.0, ticks timeout, bus-off recovery
    ├── protocols/              (4 ไฟล์)
    │   ├── wifi.md             (v1.1) แปลงโครงสร้างแล้วรอreview STA + event + ลำดับ init 9 ขั้น
    │   ├── http_client.md      (v1.xperform/streaming 2 patterns
    │   ├── mqtt.md             (v1.1) แปลงโครงสร้างแล้วรอreview nested config, %.*s, chunked data
    │   └── http_server.md      (v1.1) แปลงโครงสร้างแล้วรอreview httpd, serialized handlers, wildcard
    ├── storage/                (2 ไฟล์)
    │   ├── nvs.md              (v1.1) แปลงโครงสร้างแล้วรอreview key-value, two-call get, commit
    │   └── fatfs_sdmmc.md      (v1.1) แปลงโครงสร้างแล้วรอreview SDMMC/SDSPI + VFS + FATFS
    ├── system/                 (4 ไฟล์)
    │   ├── esp_timer.md        (v2.0) µs timer + ตารางเทียบ vs gptimer vs FreeRTOS
    │   ├── sleep.md            (v2.2) — wakeup table TRM, ext1 _io, UART edges, auto light sleep
    │   ├── heap.md             (v1.1) แปลงโครงสร้างแล้วรอreview heap_caps + MALLOC_CAP_DMA/SPIRAM
    │   └── ota.md              (v1.1) แปลงโครงสร้างแล้วรอreview esp_https_ota + rollback state machine
    └── dsp/
        └── esp_dsp.md          (v1.1) แปลงโครงสร้างแล้วรอreview dsps_* (PIE-optimized), FFT pipeline

```

## 6. การใช้ร่วมกับ skill อื่น
- `skills/esp-idf/` → โครงสร้างโปรเจกต์, build, CMake
- `skills/esp-idf-v6-migration/` → แปลงโค้ด v5 → v6
- `skills/esp32s3-freertos/` → API ฝั่ง RTOS (task, queue, semaphore)
- `skills/esp32s3-memory-map/` → วาง buffer/DMA ให้ถูก memory region (ใช้คู่กับ heap.md)
- `skills/esp32s3-peripheral-register/` → ลงระดับ register ข้าม driver
- `skills/esp32s3-pie-simd/` → instruction ระดับ asm (esp_dsp.md คือชั้น library ที่ใช้ PIE)

## 7. วิธีขยาย skill
1. สร้าง `references/<หมวด>/<ชื่อ>.md` ตาม template: **§0 TL;DR → §1 Include & CMakeLists → §2 Hardware Facts → §3–§6 API/Structs/Signatures → §7 กับดัก (ผูกรหัส Hot-List ท้ายบรรทัด) → §8 ตัวอย่างมินิมัล → §9 v6.x Notes → §10 จุดที่ต้องตรวจซ้ำกับ header จริง**
2. คัดลอก signature จาก header จริงของเวอร์ชันที่รองรับ + ใส่ version marker ทุกจุดที่ต่างกัน
3. เพิ่มข้อใหม่เข้า `HALLUCINATION-HOT-LIST.md` **ใต้กลุ่ม API ที่ตรง** (รหัสต่อจากล่าสุด เช่น SPI-13) พร้อม tag `[COMPILE]`/`[RUNTIME]`/`[SILENT]` — อย่าเก็บใน SKILL.md
4. อัปเดต File Index (§5) ของไฟล์นี้ + เพิ่ม entry ใน `CHANGELOG.md`
5. Roadmap ที่เหลือ: tsens, usb-otg, esp-tls, bt/nimble, websocket, esp_eth (W5500), spiffs, esp_partition, esp_pm, ULP-RISC-V, esp-dl inference, https_server, ETM

## ## 8. ⚠️ คิวตรวจซ้ำกับ header จริง (ของไฟล์ที่ยังไม่รีวิว — ไฟล์ reviewed แล้วดู §9/§10 ท้ายไฟล์แต่ละตัว)
- MCPWM dead-time API + enum ชุด `start_stop`; ชื่อ field ย่อยบางตัวของ rmt structs
- `ESP_TIMER_EXT` / `esp_timer_*_at` (IDF รุ่นใหม่)
- esp-dsp: biquad/matrix signatures + ชื่อ Kconfig ขนาด FFT สูงสุด
- TWAI node-handle driver ของ v6.1 (`twai_new_node`/`twai_node_transmit`)
- PCNT พฤติกรรมชน limit (accum_count)
- `esp_https_ota` fields เพิ่มของ v5.3+ (partial_http_download)
- SDHOST Slot 0 บน S3 (`sdmmc_host.h`)