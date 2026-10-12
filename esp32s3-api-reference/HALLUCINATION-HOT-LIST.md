# HALLUCINATION HOT LIST — ESP32-S3 API (Checklist Edition)

> v3.2 | ใช้คู่กับ SKILL.md v2.7.2 | แปลงจาก v2.6 (107 ข้อ) → จัดกลุ่มตาม API + tag ความรุนแรง | v3.1: § links + เวอร์ชัน reference | v3.2: หมวด ADC ขยาย 2→8 ข้อ (major review adc.md) + SLEEP group เชื่อม sleep.md v1.2 → รวม 111 ข้อ

## วิธีใช้ (อ่านก่อน)
1. เลือกกลุ่ม API ที่โค้ดใช้จาก **Dependency Map (SKILL.md §2)** — ไม่ต้องอ่านทั้งไฟล์
2. ตรวจทีละข้อ — แต่ละข้อมีลิงก์ไป reference ที่อธิบายละเอียดพร้อมที่มา
3. **Tag ความรุนแรง:**
   - `[COMPILE]` — ผิดแล้ว compile ไม่ผ่าน จับได้ง่ายสุด
   - `[RUNTIME]` — compile ผ่าน แต่พัง/ทำงานผิดตอนรัน
   - `[SILENT]` — ไม่มี error เลย แต่ผลลัพธ์ผิด (อันตรายสุด — ตรวจก่อนเสมอ)

> **การยุบรวมจาก v2.6 (107 → 105 ข้อ):** ข้อ 32+102 → I2S-01 · ข้อ 33+98 → I2S-02 (ส่วน "reconfig ต้อง disable ก่อน" แยกเป็น I2S-11) · ข้อ 34+105 → I2S-03 — ข้อเดิมทุกข้อปรากฏครบ (ตรวจย้อนได้จาก "(เดิม N)" ท้ายบรรทัด)

---

## GPIO — gpio.md v2.2 (8 ข้อ · จากข้อเดิม 1, 10, 16, 62–65, 67)
- [COMPILE] GPIO-01: `gpio_isr_t` ต้องเป็น `void (*)(void *arg)` เท่านั้น — ส่งเลข pin ผ่าน `args` → gpio.md §5 (เดิม 1)
- [COMPILE] GPIO-02: `gpio_pad_select_gpio()` deprecated v5.0 + **removed v6** → `gpio_reset_pin()` → gpio.md §4 (เดิม 10)
- [RUNTIME] GPIO-03: ห้าม `printf`/`ESP_LOGx` ใน ISR (interrupt stack) — set flag หรือ `xQueueSendFromISR()` → gpio.md §5 (เดิม 16)
- [COMPILE] GPIO-04: **S3 ไม่มี GPIO 22–25** — เลขวิ่ง 0–21 แล้วกระโดดไป 26–48 → gpio.md §2 (เดิม 62)
- [RUNTIME] GPIO-05: คงค่า digital GPIO ข้าม deep sleep ต้อง `gpio_hold_en()` + `gpio_deep_sleep_hold_en()`; RTC GPIO 0–21 ใช้ `rtc_gpio_hold_en()` → gpio.md §6 (เดิม 63)
- [SILENT] GPIO-06: GPIO3 **ไม่มี internal pull — ห้ามลอย**; default (EFUSE_STRAP_JTAG_SEL=0) ค่าถูก ignore — JTAG วิ่งผ่าน USB_SERIAL_JTAG เสมอ → gpio.md §2 (เดิม 64)
- [RUNTIME] GPIO-07: เปิด `CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP` + `gpio_wakeup_enable()` → ต้อง `rtc_gpio_init()` + ตั้ง direction เป็น input ก่อน → gpio.md §6 (เดิม 65)
- [RUNTIME] GPIO-08: `gpio_hold_dis()` ทันทีหลังตื่น deep sleep → ระดับกระโดด — ตั้ง direction+level ให้ได้ระดับเดิม**ก่อน**เรียก → gpio.md §6 (เดิม 67)

## UART — uart.md v2.1 (9 ข้อ · จากข้อเดิม 3, 14, 69–75)
- [RUNTIME] UART-01: `uart_read_bytes()`/`uart_wait_tx_done()` รับ **ticks** ทุกเวอร์ชัน (v6 เปลี่ยนแค่ชนิดพารามิเตอร์เป็น `uint32_t`) → `pdMS_TO_TICKS()` → uart.md §4 (เดิม 3)
- [COMPILE] UART-02: ขา RTS/CTS ที่ไม่ใช้ใน `uart_set_pin()` ต้องส่ง `UART_PIN_NO_CHANGE` (-1) ห้ามส่ง 0 → uart.md §3 (เดิม 14)
- [COMPILE] UART-03: v6.0 ลบ `UART_FIFO_LEN` → `UART_HW_FIFO_LEN` และ `soc/uart_channel.h` → `soc/uart_pins.h` (macro เช่น `UART_NUM_0_TXD_DIRECT_GPIO_NUM` → `U0TXD_GPIO_NUM`) → uart.md §9 (เดิม 69)
- [COMPILE] UART-04: v6.0+ header ของ driver เลิก include FreeRTOS — ต้อง include `freertos/queue.h` เอง → uart.md §1 (เดิม 70)
- [COMPILE] UART-05: `esp_vfs_dev_uart_use_driver()` deprecated ตั้งแต่ v5.3 → `uart_vfs_dev_use_driver()` → uart.md §6 (เดิม 71)
- [RUNTIME] UART-06: UART2 ปลุก light sleep ไม่ได้ — wake source เฉพาะ UART0/1; ไบต์แรก (wakeup byte) ไม่เข้า driver ต้องส่งซ้ำหลังตื่น → uart.md §2 (เดิม 72)
- [COMPILE] UART-07: `uart_set_pin()` 5 args ยังใช้ได้บน v6 (wrapper); ต้องการ DTR/DSR → `_uart_set_pin6()` 7 args → uart.md §9 (เดิม 73)
- [RUNTIME] UART-08: `uart_write_bytes()` เป็น async เฉพาะเมื่อ `tx_buffer_size > 0` ตอน install — ตั้ง 0 = block จนส่งครบ → uart.md §4 (เดิม 74)
- [SILENT] UART-09: FIFO RAM 1024×8-bit **แชร์ทั้ง 3 port** — ขยาย port หนึ่ง = ยืมพื้นที่อีก port → uart.md §2 (เดิม 75)


## I2C — i2c.md v2.1 (8 ข้อ · จากข้อเดิม 2, 11, 76–81)
- [COMPILE] I2C-01: `i2c_master_write_to_device()` บน v5.2+ → ใช้ `i2c_master_transmit()` (new driver) → i2c.md §1 (เดิม 2)
- [RUNTIME] I2C-02: `i2c_master_transmit()` timeout พารามิเตอร์ `xfer_timeout_ms` รับ **ms** ไม่ใช่ ticks → i2c.md §4 (เดิม 11)
- [COMPILE] I2C-03: pull-up อยู่ใน `.flags.enable_internal_pullup` **ทุกเวอร์ชัน v5.2–v6.1 ไม่เคยย้าย** — ห้ามใช้ doxygen cross-ref แบน ๆ เป็นหลักฐานตำแหน่ง field → i2c.md §3 (เดิม 76)
- [COMPILE] I2C-04: `i2c_master_multi_buffer_transmit()` มีอย่างน้อยตั้งแต่ v5.4.1 — ไม่ใช่ API ใหม่ของ v6.1 → i2c.md §4 (เดิม 77)
- [RUNTIME] I2C-05: NACK บน v6.0+ → `ESP_ERR_INVALID_RESPONSE` (ไม่ใช่ `ESP_ERR_INVALID_STATE`) → i2c.md §4 (เดิม 78)
- [RUNTIME] I2C-06: probe: ไม่มี ACK = `ESP_ERR_NOT_FOUND`; `ESP_ERR_TIMEOUT` = บัสผิดปกติ มักเป็น pull-up; ห้ามส่ง timeout = -1 → i2c.md §4 (เดิม 79)
- [COMPILE] I2C-07: slave API เก่า (`i2c_slave_receive`/`i2c_slave_transmit`/`*_ram`) ลบใน v5.4 — รับผ่าน callbacks + ส่งด้วย `i2c_slave_write()` → i2c.md §6 (เดิม 80)
- [SILENT] I2C-08: S3 I2C สูงสุด **~800 kbit/s** (จำกัดโดย pull-up); clock บัส = XTAL_CLK/RC_FAST_CLK เท่านั้น → i2c.md §2 (เดิม 81)

## SPI — spi.md v2.1 (12 ข้อ · จากข้อเดิม 4, 5, 13, 66, 82–89)
- [COMPILE] SPI-01: `spi_device_transmit(h, &t)` มี **2 พารามิเตอร์** — ไม่มี timeout → spi.md §6 (เดิม 4)
- [SILENT] SPI-02: `.length`/`.rxlength` ของ transaction นับเป็น **bit** — ส่ง 4 bytes = `.length = 32` → spi.md §5 (เดิม 5)
- [RUNTIME] SPI-03: **GPIO26–32 ห้ามใช้เสมอ** (SPI0/1 flash/PSRAM); 33–37 ห้ามเฉพาะเมื่อใช้ Octal flash/PSRAM — โมดูล Quad-only (N8, N8R2) ใช้ 33–37 ได้ → spi.md §2 (เดิม 13)
- [SILENT] SPI-04: `FSPID`/`FSPICLK` (GPIO35/36) เป็น alternate function ของ GPSPI2 — **ไม่ใช่**ขา flash จริง (อยู่ GPIO32/GPIO30) → spi.md §2 (เดิม 66)
- [SILENT] SPI-05: 64 B = ขีดจำกัด **ต่อ transaction** (ไม่ใช่ต่อ bus); ยัดเกินใน transaction เดียวแบบ no-DMA → HW wrap กลับอ่าน `SPI_W0` ซ้ำ ข้อมูลเสียเงียบ ๆ → spi.md §3 (เดิม 82)
- [RUNTIME] SPI-06: clock สูงสุดแยกตามเส้นทาง: **IOMUX 80 MHz / GPIO matrix half-duplex ≤40 / full-duplex ~26 MHz**; ใช้ `input_delay_ns` ชดเชยได้ → spi.md §2 (เดิม 83)
- [RUNTIME] SPI-07: DMA buffer ต้อง DMA-capable (`heap_caps_malloc(..., MALLOC_CAP_DMA)`) + **32-bit aligned + ความยาว %4 == 0**; rodata ใช้ไม่ได้ → spi.md §4 (เดิม 84)
- [COMPILE] SPI-08: v6.x มี `spi_bus_dma_memory_alloc()` — เรียก**หลัง** `spi_bus_initialize()` จัด cache+alignment ให้ → spi.md §4 (เดิม 85)
- [RUNTIME] SPI-09: default `max_transfer_sz` = **4092 B (DMA) / 64 B (no DMA)** — ตั้งค่าเฉพาะเมื่อต้องการอื่น → spi.md §3 (เดิม 86)
- [COMPILE] SPI-10: `SPI3_HOST` ไม่รองรับ Octal/OPI (8-line) — ต้องใช้ `SPI2_HOST` → spi.md §2 (เดิม 87)
- [COMPILE] SPI-11: CMakeLists v5.3+ → `REQUIRES esp_driver_spi` (v6.x legacy `driver` ไม่ pull dependency แล้ว) → spi.md §1 (เดิม 88)
- [COMPILE] SPI-12: `CONFIG_SPI_MASTER_IN_IRAM` บน v6.x กลายเป็น invisible + depends on `CONFIG_FREERTOS_IN_IRAM` → spi.md §9 (เดิม 89)



## LEDC — ledc.md v2.1 (11 ข้อ · จากข้อเดิม 17, 18, 46, 90–97)
- [COMPILE] LEDC-01: S3 มี `LEDC_LOW_SPEED_MODE` เท่านั้น — `LEDC_HIGH_SPEED_MODE` compile error → ledc.md §2 (เดิม 17)
- [RUNTIME] LEDC-02: `ledc_set_duty()` เฉย ๆ ไม่มีผล — ต้อง `ledc_update_duty()` หรือใช้ `ledc_set_duty_and_update()` → ledc.md §4 (เดิม 18)
- [COMPILE] LEDC-03: **S3 ไม่มี DAC** — `dac_output_enable()` ไม่มี ใช้ LEDC แทน → ledc.md §2 (เดิม 46)
- [SILENT] LEDC-04: `hpoint` = จุดที่สัญญาณ PWM **ขึ้น high** (เฟสเริ่มพัลส์; Lpoint = Hpoint + duty) ไม่ใช่แค่ "จุดเริ่มนับ duty" → ledc.md §2 (เดิม 90)
- [COMPILE] LEDC-05: clock source เป็น **global ของทุก timer** บน S3 — ห้ามใช้ APB กับ XTAL ปนกันข้าม timer → ledc.md §2 (เดิม 91)
- [SILENT] LEDC-06: ที่ duty_resolution สูงสุด (14-bit บน S3) **ห้ามตั้ง duty = 2^res** (16384) — ใช้สูงสุด 2^res − 1 → ledc.md §5 (เดิม 92)
- [COMPILE] LEDC-07: มี helper `ledc_find_suitable_duty_resolution(src_clk_freq, timer_freq)` — ไม่ต้องคำนวณเอง → ledc.md §4 (เดิม 93)
- [COMPILE] LEDC-08: `ledc_set_fade_with_time()` มี **4 พารามิเตอร์** — fade_mode เป็นของ `ledc_fade_start()` เท่านั้น → ledc.md §4 (เดิม 94)
- [COMPILE] LEDC-09: `ledc_cbs_t` มี callback **เดียว `.fade_cb`** (return bool; เช็ค `param->event == LEDC_FADE_END_EVT`) → ledc.md §3 (เดิม 95)
- [COMPILE] LEDC-10: v6.x ลบ/deprecate: `ledc_timer_set` (ลบ) · `ledc_isr_register` + `intr_type` (deprecated → `ledc_cb_register`) · `LEDC_APB_CLK_HZ`/`LEDC_USE_RTC8M_CLK` (ลบ → `LEDC_USE_RC_FAST_CLK`) → ledc.md §9 (เดิม 96)
- [RUNTIME] LEDC-11: S3 หยุด fade กลางคันได้ (`ledc_fade_stop()`); ตัวเลข f_max ในตาราง = ค่าที่ **CLK_DIV = 1** (ขีดจำกัดบนทางทฤษฎี) → ledc.md §2 (เดิม 97)

## GPTIMER — gptimer.md (2 ข้อ · จากข้อเดิม 19, 20)
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

## ADC — adc.md v2.0 (8 ข้อ · จากข้อเดิม 21, 22 + เพิ่มจาก review adc.md v2.0)
- [RUNTIME] ADC-01: ใช้ ADC2 ขณะ WiFi ทำงาน → fail/ค่าเพี้ยน — SAR ADC2 มี arbiter แย่งระหว่าง RTC controller กับ PWDET (WiFi) → ใช้ ADC1 (GPIO1–10) → adc.md §2 (เดิม 21)
- [COMPILE] ADC-02: `ADC_ATTEN_DB_11` บน S3 → ไม่มี enum นี้ — S3 ใช้ `ADC_ATTEN_DB_12` (DB_11 คือของ ESP32 รุ่นแรก ซึ่ง max atten ต่างกัน — เป็นเรื่อง chip ไม่ใช่ IDF เวอร์ชัน) → adc.md §2 (เดิม 22)
- [SILENT] ADC-03: ส่งเลข GPIO แทน channel → **GPIO4 = ADC1_CH3** — ADC1: GPIO1–10 = CH0–9, ADC2: GPIO11–20 = CH0–9 (channel = เลข GPIO − 1 ในช่วง unit) → adc.md §2
- [COMPILE] ADC-04: ใช้ `driver/adc.h` / `esp_adc_cal.h` / include `driver/adc_oneshot.h` → deprecated v5.0 **ลบใน v6.0** — ใช้ `esp_adc/adc_oneshot.h` + `esp_adc/adc_cali.h` + `esp_adc/adc_cali_scheme.h` และ `REQUIRES esp_adc` (**ชื่อ component พิเศษ ไม่ใช่ esp_driver_adc**) → adc.md §1
- [SILENT] ADC-05: คิดว่า calibration ต้องสร้าง handle ต่อ channel (ตั้ง `.chan`) → field `chan` เป็น **reserved ไม่มีผล** — curve fitting ผูกกับ unit_id + atten (+bitwidth) เท่านั้น สร้างหนึ่ง handle ใช้ได้ทั้ง unit → adc.md §5
- [RUNTIME] ADC-06: เอา raw 0–4095 คูณเป็นแรงดันเอง → ต้อง `adc_cali_raw_to_voltage()` — Vref ออกแบบ 1100 mV แต่**ค่าจริง 1000–1200 mV ต่างกันทุก chip** → adc.md §5
- [COMPILE] ADC-07: ใช้ continuous/DMA mode กับ ADC2 → บน S3 **DMA ต่อกับ ADC1 เท่านั้น** (TRM 39.3.7.2 + `SOC_ADC_DIG_SUPPORTED_UNIT`) — ADC2 อ่านได้แบบ oneshot เท่านั้น → adc.md §2/§4
- [SILENT] ADC-08: คิดว่า DB_12 วัดได้ถึง 3.1V (ตัวเลขของ ESP32 DB_11) → บน S3 ที่ 12 dB วัดได้ **~2450 mV** — เกินนั้นอิ่มตัว (ค่าติด 4095) → adc.md §2


## RMT — rmt.md (4 ข้อ · จากข้อเดิม 28–31)
- [COMPILE] RMT-01: `rmt_config()` + `rmt_item32_t` legacy **ลบใน v6** → `rmt_new_tx_channel()` + encoder (เดิม 28)
- [RUNTIME] RMT-02: RMT duration หน่วยจริงคือ **tick ตาม `resolution_hz`** ไม่ใช่ ns (เดิม 29)
- [RUNTIME] RMT-03: แตะ payload หลัง `rmt_transmit()` → เป็น async — ต้อง `rmt_tx_wait_all_done()` ก่อน (เดิม 30)
- [COMPILE] RMT-04: เขียน NeoPixel เองด้วย RMT → ใช้ component `led_strip` (เดิม 31)

##  I2S — i2s.md v2.1 (11 ข้อ · จากข้อเดิม 32–34, 98–107)
- [COMPILE] I2S-01: `driver/i2s.h` + `i2s_driver_install()`/`i2s_config_t`/`i2s_set_adc_mode()` **ลบใน v6.0** → channel API (`i2s_new_channel` + `i2s_channel_init_std/pdm/tdm_mode`) จาก `esp_driver_i2s` → i2s.md §1 (เดิม 32 + 102)
- [COMPILE] I2S-02: **PDM TX และ RX อยู่ I2S0 เท่านั้น** (Standard/TDM ใช้ได้ทั้ง I2S0/I2S1); PCM↔PDM converters เป็นของ I2S0 → i2s.md §2 (เดิม 33 + 98)
- [RUNTIME] I2S-03: `timeout_ms` ของ `i2s_channel_read/write` เป็น **ms** (`portMAX_DELAY` = UINT32_MAX ใช้ได้); `size` นับเป็น bytes ไม่ใช่ samples; เช็ค bytes_written/read จริงเสมอ → i2s.md §3 (เดิม 34 + 105)
- [RUNTIME] I2S-04: data 24-bit ต้องตั้ง `mclk_multiple = I2S_MCLK_MULTIPLE_384` — ไม่งั้น sample rate ไม่แม่น → i2s.md §4 (เดิม 99)
- [RUNTIME] I2S-05: `i2s_channel_preload_data()` ใช้ได้**เฉพาะ state READY + TX channel** — DMA เต็มแล้ว preload ไม่ได้ เช็ค `bytes_loaded` → i2s.md §3 (เดิม 100)
- [RUNTIME] I2S-06: buffer DMA เดี่ยว ≤ **4092 bytes**: `dma_frame_num × slot_num × data_bit_width / 8 ≤ 4092`; กันหลุด: `dma_desc_num > polling_cycle / (dma_frame_num / sample_rate)` → i2s.md §5 (เดิม 101)
- [COMPILE] I2S-07: macro ทางการคือ **PHILIPS** (มี S) — `I2S_STD_PHILIP_*` เป็น alias เก่า → i2s.md §4 (เดิม 103)
- [COMPILE] I2S-08: อ่าน DMA buffer ใน callback ผ่าน `i2s_event_data_t::data` → **deprecated v5.3** — ใช้ `dma_buf` (+ `size`) → i2s.md §6 (เดิม 104)
- [RUNTIME] I2S-09: PDM fixed **2 slots + 16-bit เท่านั้น**; TDM ≤ 16 slots; 192 kHz ไม่รองรับใน 32-bit slave → i2s.md §2 (เดิม 106)
- [SILENT] I2S-10: S3 เก็บข้อมูลต่อ slot แบบ **16-bit aligned** (slot 8-bit กิน 2 bytes จริง → buffer ใหญ่กว่าสูตรอย่างง่าย); S3 = HW v2: ใช้ `left_align`/`big_endian`/`bit_order_lsb` — **ไม่มี `msb_right`** → i2s.md §5 (เดิม 107)
- [RUNTIME] I2S-11: ต้อง `i2s_channel_enable()` ก่อน read/write; ต้อง `i2s_channel_disable()` ก่อน reconfig/delete → i2s.md §3 (เดิม 33 ส่วน reconfig)

## PCNT — pcnt.md (1 ข้อ · จากข้อเดิม 45)
- [RUNTIME] PCNT-01: `pcnt_unit_enable()` ก่อน `pcnt_unit_start()` — ลำดับเดียวกับ gptimer (เดิม 45)

## MCPWM — mcpwm.md (5 ข้อ · จากข้อเดิม 47–51)
- [COMPILE] MCPWM-01: legacy `mcpwm_init()`/`mcpwm_set_duty()` **ลบใน v6** → timer→operator→comparator→generator (เดิม 47)
- [RUNTIME] MCPWM-02: สร้างครบแต่สัญญาณไม่ออก → ลืม `mcpwm_operator_connect_timer()` หรือ `mcpwm_timer_enable()` (เดิม 48)
- [COMPILE] MCPWM-03: `mcpwm_timer_start_stop()` รับ **enum** (`MCPWM_TIMER_START_NO_STOP` ฯลฯ) ไม่ใช่ bool (เดิม 49)
- [RUNTIME] MCPWM-04: `group_id` ต้องตรงกันทุก submodule; S3 = 2 groups × 3 timers/operators (เดิม 50)
- [COMPILE] MCPWM-05: dead-time API ต่างตามเวอร์ชัน ⚠️ — ตรวจ `mcpwm_gen.h` ของเวอร์ชันนั้น (เดิม 51)

## TWAI — twai.md (5 ข้อ · จากข้อเดิม 52–56)
- [COMPILE] TWAI-01: S3 = **CAN 2.0 classic เท่านั้น** — ไม่มี CAN-FD; DLC ≤ 8 (เดิม 52)
- [RUNTIME] TWAI-02: `twai_transmit()` timeout เป็น **ticks** → `pdMS_TO_TICKS()` (เดิม 53)
- [RUNTIME] TWAI-03: install แล้วส่งไม่ได้ → ลืม `twai_start()` (state STOPPED) (เดิม 54)
- [RUNTIME] TWAI-04: bus-off ไม่ฟื้นเอง → ต้อง `twai_initiate_recovery()` (เดิม 55)
- [COMPILE] TWAI-05: v6.1 เพิ่ม node-handle driver (`twai_new_node`/`twai_node_transmit`) ⚠️ — เทียบ docs ก่อนเลือกชุด (เดิม 56)

## WIFI — wifi.md (3 ข้อ · จากข้อเดิม 6–8)
- [RUNTIME] WIFI-01: `esp_netif_create_default_wifi_sta()` คืน **pointer** ห้ามครอบ `ESP_ERROR_CHECK` — เช็ค NULL เอง (เดิม 6)
- [RUNTIME] WIFI-02: ลำดับ init: **NVS → netif → event loop → default netif → `esp_wifi_init()`** (เดิม 7)
- [RUNTIME] WIFI-03: ลืม `nvs_flash_init()` → WiFi พัง (WiFi ต้องใช้ NVS) (เดิม 8)

## HTTP — http_client.md (2 ข้อ · จากข้อเดิม 12, 15)
- [RUNTIME] HTTP-01: `esp_http_client_init()` คืน **handle** — เช็ค NULL ไม่ใช่ esp_err_t (เดิม 12)
- [RUNTIME] HTTP-02: ลืม `esp_http_client_cleanup()` → memory leak (เดิม 15)

## MQTT — mqtt.md (2 ข้อ · จากข้อเดิม 23, 24)
- [COMPILE] MQTT-01: config v5+ เป็น nested — `.broker.address.uri` ไม่ใช่ `.host`/`.port` (เดิม 23)
- [RUNTIME] MQTT-02: `evt->data` ไม่มี `'\0'` — ใช้ `%.*s` + `evt->data_len` (เดิม 24)

## HTTPD — http_server.md (1 ข้อ · จากข้อเดิม 44)
- [RUNTIME] HTTPD-01: handlers ทำงานแบบ **serialized** (task เดียว) — handler หนึ่งบล็อก = ทั้ง server ค้าง (เดิม 44)

## NVS — nvs.md (1 ข้อ · จากข้อเดิม 9)
- [RUNTIME] NVS-01: `nvs_set_*()` แล้วจบ → ต้อง `nvs_commit()` ไม่งั้นค่าไม่ลง flash (เดิม 9)

## SD — fatfs_sdmmc.md (3 ข้อ · จากข้อเดิม 57–59)
- [RUNTIME] SD-01: `fopen("/sdcard/…")` ต้อง mount ก่อน + ตั้ง `max_files` พอ — ไม่งั้น ENOENT/fopen fail (เดิม 57)
- [COMPILE] SD-02: SD โหมด SPI ต้อง `sdspi_device_config_t` + `SDSPI_HOST_DEFAULT()` ไม่ใช่ `sdmmc_slot_config_t` (เดิม 58)
- [SILENT] SD-03: ลืม `host.flags = SDMMC_HOST_FLAG_4BIT` → วิ่ง 1-bit เงียบ ๆ (ช้า 4 เท่า ไม่มี error) (เดิม 59)

## ESP_TIMER — esp_timer.md (3 ข้อ · จากข้อเดิม 25–27)
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

## SLEEP — sleep.md v1.2 (8 ข้อ · จากข้อเดิม 35–41, 68)
- [COMPILE] SLEEP-01: `esp_sleep_enable_timer_wakeup()` คืน `esp_err_t` ครอบ `ESP_ERROR_CHECK` ได้; `uint64_t` คือชนิด**พารามิเตอร์** (เดิม 35)
- [RUNTIME] SLEEP-02: deep sleep ปลุกด้วย digital GPIO ไม่ได้ — เฉพาะ **RTC GPIO 0–21** ผ่าน ext0/ext1 (เดิม 36)
- [RUNTIME] SLEEP-03: `gpio_wakeup_enable()` ใช้กับ **light sleep** เท่านั้น (+ `esp_sleep_enable_gpio_wakeup()`) (เดิม 37)
- [RUNTIME] SLEEP-04: WiFi/BT/UART เป็น wake source ของ **light sleep** เท่านั้น (เดิม 38)
- [COMPILE] SLEEP-05: `esp_sleep_enable_ext1_wakeup()` บน v6 → deprecated — ใช้ variant `_io`; ตรวจสถานะด้วย bitmask ผ่าน `__builtin_ctzll()` (เดิม 39)
- [RUNTIME] SLEEP-06: UART wakeup: ไบต์แรกไม่เข้า driver ต้องส่งซ้ำหลังตื่น; threshold นับ rising edges (S3 ขั้นต่ำ 3) (เดิม 40)
- [SILENT] SLEEP-07: `3600 * 1000000` ใน int → overflow — ต้อง `3600ULL * 1000000ULL` (เดิม 41)
- [COMPILE] SLEEP-08: v6.0+ `esp_sleep_get_wakeup_cause()` deprecated → `esp_sleep_get_wakeup_causes()` คืน bitmap เช็คด้วย `& ESP_SLEEP_WAKEUP_*` (เดิม 68)

## HEAP — heap.md (1 ข้อ · จากข้อเดิม 42)
- [RUNTIME] HEAP-01: `malloc()` แล้วหวังได้ PSRAM → ต้อง `heap_caps_malloc(size, MALLOC_CAP_SPIRAM)` (เดิม 42)

## OTA — ota.md (2 ข้อ · จากข้อเดิม 60, 61)
- [RUNTIME] OTA-01: เปิด rollback แล้วต้อง `esp_ota_mark_app_valid_cancel_rollback()` — ไม่งั้น reboot วนกลับ app เก่า (เดิม 60)
- [COMPILE] OTA-02: ต้อง `esp_ota_end()` ก่อน `esp_ota_set_boot_partition()`; ต้องมี ota_0/ota_1+otadata; HTTPS ต้องใส่ cert (เดิม 61)

## DSP — esp_dsp.md (1 ข้อ · จากข้อเดิม 43)
- [RUNTIME] DSP-01: FFT ต้อง `dsps_fft2r_init_fc32()` + `dsps_bit_rev_fc32()` ก่อน `dsps_cplx2reC_fc32()` (เดิม 43)

---

**ตรวจสอบการย้าย (v2.6 → v3.0):** ข้อเดิม 1–107 ปรากฏครบทุกข้อ (ดู "(เดิม N)" ท้ายบรรทัด) — ยกเว้น 3 คู่ที่ยุบรวม: 32+102 → I2S-01 · 33+98 → I2S-02 (+I2S-11) · 34+105 → I2S-03 → รวม 105 ข้อ ใน 23 กลุ่ม API