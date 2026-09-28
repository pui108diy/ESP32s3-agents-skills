# Hallucination Hot List — ESP32-S3 API (ฉบับรวม 61 ข้อ)

> ไฟล์รวมจากทุก reference — ตรวจ 61 ข้อนี้ก่อนส่งโค้ดทุกครั้ง
> สถานะเวอร์ชันอ้างอิง: IDF v5.2–v6.x (stable = v6.1), TRM ESP32-S3 v1.8

## กฎเหล็ก (สรุป)
1. ห้ามเขียน API จากความจำลอย ๆ — เปิด reference ของ peripheral นั้นก่อน
2. API ที่ไม่มีใน reference = verify กับ header จริงก่อน (แล้วเขียนเพิ่มกลับ)
3. เช็ค IDF เวอร์ชันของโปรเจกต์ → จับคู่ Version Decision Table ใน SKILL.md
4. ส่งมอบ = #include ครบ + จัดการ esp_err_t ทุกจุด

---

## A. GPIO / UART / SPI / I2C
| #   | สิ่งที่มักเขียนผิด                                 | ความจริง                                      |
| --- | -------------------------------------------------- | --------------------------------------------- |
| 1   | `void isr(int gpio_num)`                           | `gpio_isr_t` = `void (*)(void *arg)` ตัวเดียว |
| 2   | `i2c_master_write_to_device()` บน v5.2+            | ใช้ `i2c_master_transmit()` (handle + ms)     |
| 3   | `uart_read_bytes(..., pdMS_TO_TICKS(100))`         | ถูกบน v5.0–v5.4; เวอร์ชันใหม่รับ ms ตรง ๆ ⚠️  |
| 4   | `spi_device_transmit(h,&t,timeout)`                | มี 2 พารามิเตอร์ (blocking อยู่แล้ว)          |
| 5   | `.length = sizeof(buf)`                            | SPI นับเป็น **bit** → `sizeof(buf)*8`         |
| 10  | `gpio_pad_select_gpio()`                           | deprecated→removed → `gpio_reset_pin()`       |
| 11  | ส่ง ticks ให้ `i2c_master_transmit`                | `xfer_timeout_ms` เป็น ms                     |
| 13  | ใช้ GPIO 26–32 / 33–37 บน S3                       | flash ภายใน / Octal PSRAM — ห้าม              |
| 14  | `uart_set_pin(..., 0, 0, 0, 0)` เมื่อไม่มี RTS/CTS | ต้อง `UART_PIN_NO_CHANGE` (-1)                |
| 16  | `printf`/`ESP_LOGx` ใน ISR                         | ห้าม — flag/queue แล้วทำใน task               |

## B. WiFi / HTTP / NVS
| #   | สิ่งที่มักเขียนผิด                                     | ความจริง                                            |
| --- | ------------------------------------------------------ | --------------------------------------------------- |
| 6   | `ESP_ERROR_CHECK(esp_netif_create_default_wifi_sta())` | คืน **pointer** ห้ามครอบ                            |
| 7   | `esp_wifi_init` ก่อน netif                             | ลำดับ: NVS→netif→event loop→default netif→wifi init |
| 8   | ลืม `nvs_flash_init()`                                 | esp_wifi_init พัง (WiFi ต้องใช้ NVS)                |
| 9   | `nvs_set_*` แล้ว close เลย                             | ต้อง `nvs_commit()` ก่อน                            |
| 12  | `ESP_ERROR_CHECK(esp_http_client_init())`              | คืน handle — เช็ค NULL                              |
| 15  | ลืม `esp_http_client_cleanup()`                        | memory leak                                         |

## C. LEDC / GPTimer / ADC / MQTT
| # | สิ่งที่มักเขียนผิด | ความจริง |
|---|---|---|
| 17 | `LEDC_HIGH_SPEED_MODE` บน S3 | S3 มี **low-speed** เท่านั้น |
| 18 | `ledc_set_duty()` เฉย ๆ | ต้อง `ledc_update_duty()` / `set_duty_and_update()` |
| 19 | `timer_group_init`/`timer_isr_register` | legacy `driver/timer.h` **ลบใน v6** → gptimer |
| 20 | `gptimer_start()` ไม่ `gptimer_enable()` | `ESP_ERR_INVALID_STATE` |
| 21 | อ่าน ADC2 ขณะ WiFi ทำงาน | พัง — ใช้ ADC1 (GPIO1–10) |
| 22 | `ADC_ATTEN_DB_11` บน S3 | ระดับ 4 = 12 dB → `ADC_ATTEN_DB_12` |
| 23 | mqtt `.host`/`.port` | สไตล์ v4 — v5+ ใช้ `.broker.address.uri` |
| 24 | `printf("%s", evt->data)` ใน MQTT | ไม่มี `'\0'` → `%.*s` + `data_len` |

## D. esp_timer / RMT / I2S
| # | สิ่งที่มักเขียนผิด | ความจริง |
|---|---|---|
| 25 | `esp_timer_start_periodic(t,1000)` คิดว่า 1 วิ | หน่วย **µs** → 1 s = 1000000 |
| 26 | esp_timer callback ยาว/หลายตัวพร้อมกัน | ทุกตัวแชร์ task เดียว — ตัวหนึ่งหน่วง ทั้งระบบเลื่อน |
| 27 | เก็บ `esp_timer_get_time()` ใน `int` | `int64_t` — `int32_t` ล้นใน ~35 นาที |
| 28 | `rmt_config()` + `rmt_item32_t` | legacy **ลบใน v6** → `rmt_new_tx_channel` + encoder |
| 29 | RMT duration ใส่ ns | หน่วย **tick ตาม resolution_hz** |
| 30 | `rmt_transmit()` แล้วแตะ payload | async — ต้อง `rmt_tx_wait_all_done()` ก่อน |
| 31 | เขียน NeoPixel ด้วย RMT เอง | ใช้ component `led_strip` |
| 32 | `i2s_driver_install()` + `i2s_config_t` | legacy **ลบใน v6** → `i2s_new_channel` + `init_std_mode` |
| 33 | PDM บน I2S1 / reconfig ระหว่างรัน | PDM เฉพาะ **I2S0**; reconfig ต้อง disable ก่อน |
| 34 | `i2s_channel_write(..., ticks)` | timeout เป็น **ms** (`uint32_t`) |

## E. Sleep / Heap / DSP / HTTPD / PCNT
| # | สิ่งที่มักเขียนผิด | ความจริง |
|---|---|---|
| 35 | `ESP_ERROR_CHECK(esp_sleep_enable_timer_wakeup())` | คืน **uint64_t** ห้ามครอบ |
| 36 | ปลุก deep sleep ด้วย GPIO22–48 | ได้เฉพาะ **RTC GPIO 0–21** (ext0/ext1) |
| 37 | `gpio_wakeup_enable()` กับ deep sleep | เป็น API ของ **light sleep** |
| 38 | WiFi/BT/UART ปลุก deep sleep | **light sleep เท่านั้น** (TRM 10.4-3) |
| 39 | `esp_sleep_enable_ext1_wakeup()` บน v6 | **deprecated ตั้งแต่ v6.0** → `_io` variant; status คืน bitmask → `__builtin_ctzll()` หาขา |
| 40 | UART wake คิดว่าได้ข้อมูล | byte ตอน sleep **ไม่เข้า driver** — ส่ง wakeup byte ก่อน; threshold = rising edges (S3 ขั้นต่ำ 3 — ASCII '0' ปลุกไม่ตื่น) |
| 41 | `esp_sleep_enable_timer_wakeup(3600 * 1000000)` | int overflow (3.6e9 > INT32_MAX) → `3600ULL * 1000000ULL` เสมอ |
| 42 | `malloc()` แล้วหวัง PSRAM | ต้อง CONFIG / `MALLOC_CAP_SPIRAM`; DMA = `MALLOC_CAP_DMA` |
| 43 | `dsps_fft2r_fc32()` เลย | ต้อง init + `bit_rev` (+`cplx2reC`); N = 2^n |
| 44 | httpd handler ทำงานนาน | handler ทั้ง server **serialized ใน task เดียว** |
| 45 | `pcnt_unit_start()` ไม่ `pcnt_unit_enable()` | `ESP_ERR_INVALID_STATE`; `low_limit` ต้องติดลบ |
| 46 | `dac_output_enable()` บน S3 | **S3 ไม่มี DAC** — ใช้ LEDC + วงจรภายนอก |

## F. MCPWM / TWAI / SD / OTA (รอบ production)
| #   | สิ่งที่มักเขียนผิด                                      | ความจริง                                                                                      |
| --- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| 47  | `mcpwm_init()` + `mcpwm_set_duty()`                     | legacy **ลบใน v6** → timer→operator→comparator→generator                                      |
| 48  | สร้างครบแต่สัญญาณไม่ออก                                 | ลืม `mcpwm_operator_connect_timer()` หรือ `mcpwm_timer_enable()`                              |
| 49  | `mcpwm_timer_start_stop(timer, true)`                   | พารามิเตอร์เป็น **enum** (`MCPWM_TIMER_START_NO_STOP` ฯลฯ)                                    |
| 50  | MCPWM `group_id` ปนกัน (0/1)                            | ต้องตรงกันทุก submodule; S3 มี 2 groups × 3 timers/operators                                  |
| 51  | Dead time แบบ legacy                                    | ตรวจ API ของเวอร์ชันนั้น ⚠️ / classic = comparator 2 ตัวต่างกันเท่า dead time                 |
| 52  | TWAI DLC > 8 / คิดว่ามี CAN-FD                          | S3 = **CAN 2.0 classic** เท่านั้น                                                             |
| 53  | `twai_transmit(..., 100)` คิดว่า ms                     | timeout เป็น **ticks** → `pdMS_TO_TICKS()`                                                    |
| 54  | install แล้วส่งไม่ได้                                   | ลืม `twai_start()` → state STOPPED                                                            |
| 55  | bus-off แล้วรอเดี๋ยวเอง                                 | ต้อง `twai_initiate_recovery()` เอง                                                           |
| 56  | `twai_driver_install` บน v6.1+                          | ⚠️ v6.1 เพิ่ม driver แบบ node-handle (`twai_new_node`/`twai_node_transmit`) — เทียบ docs ก่อน |
| 57  | `fopen("/sdcard/…")` ไม่ mount ก่อน / `max_files` น้อย  | ENOENT / fopen fail — mount ก่อน + ตั้ง max_files พอ                                          |
| 58  | SD โหมด SPI ใช้ `sdmmc_slot_config_t`                   | ต้อง `sdspi_device_config_t` + `SDSPI_HOST_DEFAULT()`                                         |
| 59  | ลืม `host.flags = SDMMC_HOST_FLAG_4BIT`                 | วิ่งแค่ 1-bit เงียบ ๆ (ช้า 4 เท่า ไม่มี error)                                                |
| 60  | OTA แล้ว reboot วนกลับ app เก่า                         | เปิด rollback แล้วลืม `esp_ota_mark_app_valid_cancel_rollback()`                              |
| 61  | `esp_ota_set_boot_partition` โดย image ไม่ผ่าน validate | ต้อง `esp_ota_end()` ก่อน; ต้องมี ota_0/ota_1+otadata; HTTPS ต้องใส่ cert                     |
