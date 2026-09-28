---
name: esp32s3-api-reference
version: 2.0.0
target: esp32s3
idf_versions: v5.2 – v6.x (stable ปัจจุบัน = v6.1)
description: >-
  Exact ESP-IDF API signatures สำหรับ ESP32-S3 — ครบ 25 ไฟล์: peripheral 12 (gpio uart spi i2c
  ledc gptimer adc rmt i2s pcnt mcpwm twai) + protocol 4 (wifi http_client mqtt http_server)
  + storage 2 (nvs fatfs_sdmmc) + system 4 (esp_timer sleep heap ota) + dsp 1 (esp_dsp)
  พร้อม version markers และ required includes — Anti-Hallucination layer
  โหลดก่อนเขียน/ตรวจโค้ด C ที่เรียก ESP-IDF API ทุกครั้ง
---

# ESP32-S3 API Reference — Anti-Hallucination Layer

## 1. เมื่อไรต้องใช้ skill นี้
- กำลังเขียน/แก้/review โค้ด C ที่เรียก driver, WiFi/BT, HTTP (client/server), MQTT, NVS/SD, OTA, sleep หรือ DSP บน ESP32-S3
- ต้องยืนยัน function signature, struct field, หรือ header ที่ต้อง include สำหรับ IDF เวอร์ชันไหน

## 2. กฎเหล็ก 7 ข้อ
1. **เปิดไฟล์ reference ของ peripheral นั้นก่อนเขียนเสมอ** — ห้ามเขียน API จากความจำลอย ๆ
2. **คัดลอก signature ตรงตาม reference** — ห้ามเปลี่ยนชนิดพารามิเตอร์ ห้ามเติมพารามิเตอร์ที่ไม่มีจริง
3. **API ที่ไม่มีใน reference = ต้อง verify ก่อนใช้** กับ header จริง (`components/esp_driver_*/include/`) หรือ docs.espressif.com — แล้วเขียนผลเพิ่มกลับเข้า reference
4. **เช็ค ESP-IDF เวอร์ชันของโปรเจกต์ก่อน** (`idf.py --version`) แล้วจับคู่กับ §3 Version Decision Table
5. **โค้ดที่ส่งมอบต้องมี `#include` ครบ** และจัดการ error ด้วย `esp_err_t` / `ESP_ERROR_CHECK` ทุกจุด
6. **ก่อนส่งโค้ดทุกครั้ง → รันผ่าน `HALLUCINATION-HOT-LIST.md` (61 ข้อ — ไฟล์แยกที่รากของ skill)** ให้ครบทุกหมวดที่เกี่ยวข้อง
7. **เจอป้าย ⚠️ = จุดที่ API เปลี่ยนระหว่างเวอร์ชัน** — ต้องเปิด header ของเวอร์ชันที่ใช้ยืนยันก่อน (รายการรวมอยู่ที่ §8)

## 3. Version Decision Table

| หัวข้อ | v4.x | v5.0–v5.1 | v5.2–v5.3 | v5.4–v5.5 | v6.x (stable = v6.1) |
|---|---|---|---|---|---|
| I2C driver | legacy `driver/i2c.h` | legacy (เริ่ม deprecate) | **new `driver/i2c_master.h`** (pullup = field ตรง `.enable_internal_pullup`) | pullup ย้ายเข้า `.flags.enable_internal_pullup` ⚠️; v5.5: port = -1 auto | legacy = **EOL** (ลบใน v7) ใช้ new เท่านั้น |
| โครงสร้าง component | `components/driver` | เดิม | v5.3 เริ่มแยก `esp_driver_*` | แยกแล้ว | `esp_driver_uart/i2c/rmt/...` (header `driver/*.h` ยังชื่อเดิม) |
| `gpio_pad_select_gpio()` | ใช้ได้ | deprecated | deprecated | deprecated | **removed** → `gpio_reset_pin()` |
| `uart_read_bytes()` timeout | ticks | ticks | ticks | ช่วงปลาย v5 เปลี่ยนเป็น ms ⚠️ | **ms (`int`)** |
| NVS `nvs_handle_t` | `uint32_t` | `uint32_t` | `uint32_t` | **opaque pointer** | opaque pointer |
| ADC/Timer/RMT/I2S/MCPWM/PCNT | legacy | **new drivers** มาแล้ว | new | new | legacy ถูกลบ |
| ADC atten (บน S3) | — | `ADC_ATTEN_DB_12` | เดิม | เดิม | เดิม (ระวัง `DB_11` จากโค้ด ESP32 รุ่นแรก) |
| MQTT config struct | flat (`.host`/`.port`) | **nested** (`.broker.address.uri`) | nested | nested | nested |
| ext1 wakeup | — | `esp_sleep_enable_ext1_wakeup` | เดิม | + variant ใหม่ `_io` | ตัวเดิม **deprecated ตั้งแต่ v6.0** → `esp_sleep_enable_ext1_wakeup_io()` |
| sleep cause type | — | `esp_sleep_wakeup_cause_t` | เดิม | เดิม | rename → **`esp_sleep_source_t`** (ชื่อ enum เดิม) |
| UART wake threshold | — | `uart_set_wakeup_threshold()` | เดิม | เดิม | + ชุดใหม่ `uart_wakeup_setup()` + `uart_wakeup_cfg_t` ⚠️ |
| TWAI | classic driver | classic | classic | classic | v6.1: + **node-handle driver** (`twai_new_node`/`twai_node_transmit`) ⚠️ เลือกชุดให้ถูก |
| MCPWM dead-time | — | new driver (pipeline) | + API dead-time เฉพาะ ⚠️ | เดิม | เดิม — ตรวจ `mcpwm_gen.h` ของเวอร์ชันนั้น |

## 4. Hallucination Hot List (ไฟล์แยก)

**ดูที่ `HALLUCINATION-HOT-LIST.md` (รากของ skill) — 61 ข้อ แบ่ง 6 หมวด:**

| หมวด | ข้อ | ครอบเรื่อง |
|---|---|---|
| A | 1–16 | GPIO / UART / SPI / I2C |
| B | 17–24 และ 6–15 (WiFi/HTTP/NVS) | WiFi init, pointer returns, NVS commit |
| C | 17–24 | LEDC / gptimer / ADC / MQTT |
| D | 25–34 | esp_timer / RMT / I2S |
| E | 35–49 | Sleep / Heap / DSP / HTTPD / PCNT |
| F | 50–61 | MCPWM / TWAI / SD / OTA |

## 5. File Index (25 ไฟล์)
```
esp32s3-api-reference/
├── SKILL.md                    ← ไฟล์นี้ (v2.0)
├── HALLUCINATION-HOT-LIST.md   ← 61 ข้อ (ไฟล์แยก)
└── references/
    ├── peripherals/            (12 ไฟล์)
    │   ├── gpio.md             ดิจิทัล IO, ISR, strapping, GPIO map เต็ม
    │   ├── uart.md             3 ports, ticks→ms marker, VFS
    │   ├── spi.md              SPI master + GDMA, 64-byte no-DMA limit
    │   ├── i2c.md              new master driver (+ legacy สำหรับอ่านโค้ดเก่า)
    │   ├── ledc.md             8ch/4timer, fade, low-speed only บน S3
    │   ├── gptimer.md          4× 54-bit TIMG + state machine enable→start
    │   ├── adc.md              2×12-bit, ADC1 กับ WiFi, curve fitting
    │   ├── rmt.md              8ch TX0-3/RX4-7, encoder, 48 words/ch, +led_strip
    │   ├── i2s.md              new channel driver, std/pdm/tdm, PDM=I2S0
    │   ├── pcnt.md             4 units×2ch, glitch filter, encoder 4x
    │   ├── mcpwm.md            2 groups×3 timers/operators, pipeline, capture
    │   └── twai.md             CAN 2.0, ticks timeout, bus-off recovery
    ├── protocols/              (4 ไฟล์)
    │   ├── wifi.md             STA + event + ลำดับ init 9 ขั้น
    │   ├── http_client.md      perform/streaming 2 patterns
    │   ├── mqtt.md             nested config, %.*s, chunked data
    │   └── http_server.md      httpd, serialized handlers, wildcard
    ├── storage/                (2 ไฟล์)
    │   ├── nvs.md              key-value, two-call get, commit
    │   └── fatfs_sdmmc.md      SDMMC/SDSPI + VFS + FATFS
    ├── system/                 (4 ไฟล์)
    │   ├── esp_timer.md        µs timer + ตารางเทียบ vs gptimer vs FreeRTOS
    │   ├── sleep.md            (v1.1 — แก้ตาม review) wakeup table TRM, ext1 _io, UART edges
    │   ├── heap.md             heap_caps + MALLOC_CAP_DMA/SPIRAM
    │   └── ota.md              esp_https_ota + rollback state machine
    └── dsp/
        └── esp_dsp.md          dsps_* (PIE-optimized), FFT pipeline
```

## 6. การใช้ร่วมกับ skill อื่น
- `skills/esp-idf/` → โครงสร้างโปรเจกต์, build, CMake
- `skills/esp-idf-v6-migration/` → แปลงโค้ด v5 → v6
- `skills/esp32s3-freertos/` → API ฝั่ง RTOS (task, queue, semaphore)
- `skills/esp32s3-memory-map/` → วาง buffer/DMA ให้ถูก memory region (ใช้คู่กับ heap.md)
- `skills/esp32s3-peripheral-register/` → ลงระดับ register ข้าม driver
- `skills/esp32s3-pie-simd/` → instruction ระดับ asm (esp_dsp.md คือชั้น library ที่ใช้ PIE)

## 7. วิธีขยาย skill
1. สร้าง `references/<หมวด>/<ชื่อ>.md` ตาม template เดียวกัน: **Include → Hardware Facts → Structs → Signatures → กับดัก → ตัวอย่างมินิมัล**
2. คัดลอก signature จาก header จริงของเวอร์ชันที่รองรับ + ใส่ version marker ทุกจุดที่ต่างกัน
3. เพิ่มข้อใหม่เข้า `HALLUCINATION-HOT-LIST.md` (ต่อเลขข้อล่าสุด) — อย่าเก็บใน SKILL.md
4. อัปเดต File Index (§5) + Changelog (§10) ของไฟล์นี้
5. Roadmap ที่เหลือ: tsens, usb-otg, esp-tls, bt/nimble, websocket, esp_eth (W5500), spiffs, esp_partition, esp_pm, ULP-RISC-V, esp-dl inference, https_server, ETM

## 8. ⚠️ จุดที่ต้องตรวจซ้ำกับ header จริง (ป้ายใน reference)
- `uart_read_bytes` timeout (ticks vs ms) ตาม IDF เวอร์ชันที่ใช้
- MCPWM dead-time API + enum ชุด `start_stop`; ชื่อ field ย่อยบางตัวของ i2s/rmt structs
- `ESP_TIMER_EXT` / `esp_timer_*_at` (IDF รุ่นใหม่)
- esp-dsp: biquad/matrix signatures + ชื่อ Kconfig ขนาด FFT สูงสุด
- TWAI node-handle driver ของ v6.1 (`twai_new_node`/`twai_node_transmit`)
- PCNT พฤติกรรมชน limit (accum_count)
- `esp_https_ota` fields เพิ่มของ v5.3+ (partial_http_download)
- SDHOST Slot 0 บน S3 (`sdmmc_host.h`)

## 9. Smoke Tests (ตรวจว่า skill ทำงานจริง — ใช้เป็น prompt ทดสอบ)
| #   | Prompt ทดสอบ                            | ผลที่ต้องได้                                           |
| --- | --------------------------------------- | ------------------------------------------------------ |
| 1   | อ่าน WHO_AM_I reg 0x68 ผ่าน I2C บน v5.4 | `i2c_master.h` + handle + timeout `100` ms             |
| 2   | ปุ่ม GPIO4 falling edge + ISR           | `void isr(void*)` + `install_isr_service` + `1ULL<<4`  |
| 3   | เซฟ string ลง NVS แล้วอ่านกลับ          | `nvs_commit` + two-call `nvs_get_str` + key ≤ 15       |
| 4   | หายใจ LED ด้วย fade                     | `fade_func_install` + `LEDC_LOW_SPEED_MODE`            |
| 5   | timer 1 วินาที periodic                 | gptimer: `enable`→`start` + `auto_reload`              |
| 6   | อ่านแรงดัน GPIO4                        | `ADC_CHANNEL_3` + curve fitting (ไม่ใช่ ch 4)          |
| 7   | MQTT subscribe topic                    | subscribe ใน `MQTT_EVENT_CONNECTED` + `%.*s`           |
| 8   | callback ทุก 1 วินาที esp_timer         | `1000000` µs + `void cb(void*)`                        |
| 9   | ส่ง NEC IR code                         | `rmt_enable` + `wait_all_done` + duration = tick       |
| 10  | เล่นเสียง 44.1 kHz                      | std mode + `enable` ก่อน `write` + timeout ms          |
| 11  | deep sleep ปลุกทุก 5 นาที + ปุ่ม        | ext1 + `RTC_DATA_ATTR` + `300ULL*1000000ULL`           |
| 12  | จอง SPI DMA buffer 4KB                  | `heap_caps_malloc(4096, MALLOC_CAP_DMA)`               |
| 13  | FFT 1024 หา magnitude                   | init + `bit_rev` + `cplx2reC` + `float[2048]`          |
| 14  | web server รับ POST ตอบ JSON            | recv loop จน content_len + `HTTPD_TYPE_JSON`           |
| 15  | encoder 4x                              | channel เดียว edge+level + glitch filter               |
| 16  | servo 50 Hz                             | `connect_timer` + enum `MCPWM_TIMER_START_NO_STOP`     |
| 17  | ส่ง/รับ CAN 500k                        | `twai_start` + `pdMS_TO_TICKS` (ticks!)                |
| 18  | บันทึกไฟล์ลง SD                         | mount ก่อน `fopen` + `max_files` + 4BIT flag           |
| 19  | OTA + rollback                          | `mark_app_valid_cancel_rollback` + partition table OTA |

## 10. Changelog
- **v2.0 (ปัจจุบัน):** Hot List แยกออกเป็น `HALLUCINATION-HOT-LIST.md` (58→61 ข้อ) · Version Decision Table เพิ่ม ext1/_io, sleep type rename, UART wake v6, TWAI node-handle v6.1, MCPWM dead-time · เพิ่ม §8 จุดตรวจซ้ำ + §9 Smoke Tests รวม 19 ข้อ
- **v1.1:** `sleep.md` แก้ตาม external review + verify (`esp_deep_sleep_start` = void, PM 3 ขั้น, ext1 bitmask, UART edges, ULL overflow) — Hot List ใส่เพิ่มข้อ 39-41
- **v1.0:** รอบ 1 (gpio/uart/spi/i2c/wifi/http_client/nvs) → รอบ 2 (ledc/gptimer/adc/mqtt) → รอบ 3 (esp_timer/rmt/i2s) → รอบ 4 (sleep/heap/esp_dsp/http_server/pcnt) → รอบ 5 (mcpwm/twai/fatfs_sdmmc/ota) = ครบ 25 ไฟล์




