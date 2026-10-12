# SMOKE TESTS — esp32s3-api-reference
> ย้ายจาก SKILL.md §9 เดิม (v2.7.0) — ใช้เป็น prompt ทดสอบว่า skill ช่วย agent ตอบถูกจริง
> วิธีใช้: ส่ง prompt ให้ agent ที่โหลด skill แล้ว แล้วเทียบกับ "ผลที่ต้องได้"

| #   | Prompt ทดสอบ                                           | ผลที่ต้องได้                                                                                                                       |
| --- | ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| 1   | อ่าน WHO_AM_I reg 0x68 ผ่าน I2C บน v5.4                | `i2c_master.h` + handle + timeout `100` ms                                                                                         |
| 2   | ปุ่ม GPIO4 falling edge + ISR                          | `void isr(void*)` + `install_isr_service` + `1ULL<<4`                                                                              |
| 3   | เซฟ string ลง NVS แล้วอ่านกลับ                         | `nvs_commit` + two-call `nvs_get_str` + key ≤ 15                                                                                   |
| 4   | หายใจ LED ด้วย fade                                    | `fade_func_install` + `LEDC_LOW_SPEED_MODE`                                                                                        |
| 5   | timer 1 วินาที periodic                                | gptimer: `enable`→`start` + `auto_reload`                                                                                          |
| 6   | อ่านแรงดัน GPIO4                                       | `ADC_CHANNEL_3` + curve fitting (ไม่ใช่ ch 4)                                                                                      |
| 7   | MQTT subscribe topic                                   | subscribe ใน `MQTT_EVENT_CONNECTED` + `%.*s`                                                                                       |
| 8   | callback ทุก 1 วินาที esp_timer                        | `1000000` µs + `void cb(void*)`                                                                                                    |
| 9   | ส่ง NEC IR code                                        | `rmt_enable` + `wait_all_done` + duration = tick                                                                                   |
| 10  | เล่นเสียง 44.1 kHz                                     | std mode + `enable` ก่อน `write` + timeout ms                                                                                      |
| 11  | deep sleep ปลุกทุก 5 นาที + ปุ่ม                       | ext1 + `RTC_DATA_ATTR` + `300ULL*1000000ULL`                                                                                       |
| 12  | จอง SPI DMA buffer 4KB                                 | `heap_caps_malloc(4096, MALLOC_CAP_DMA)`                                                                                           |
| 13  | FFT 1024 หา magnitude                                  | init + `bit_rev` + `cplx2reC` + `float[2048]`                                                                                      |
| 14  | web server รับ POST ตอบ JSON                           | recv loop จน content_len + `HTTPD_TYPE_JSON`                                                                                       |
| 15  | encoder 4x                                             | channel เดียว edge+level + glitch filter                                                                                           |
| 16  | servo 50 Hz                                            | `connect_timer` + enum `MCPWM_TIMER_START_NO_STOP`                                                                                 |
| 17  | ส่ง/รับ CAN 500k                                       | `twai_start` + `pdMS_TO_TICKS` (ticks!)                                                                                            |
| 18  | บันทึกไฟล์ลง SD                                        | mount ก่อน `fopen` + `max_files` + 4BIT flag                                                                                       |
| 19  | OTA + rollback                                         | `mark_app_valid_cancel_rollback` + partition table OTA                                                                             |
| 20  | ปลุก light sleep ด้วยปุ่ม GPIO4                        | `gpio_wakeup_enable` + `esp_sleep_enable_gpio_wakeup` (ไม่ใช่ ext0)                                                                |
| 21  | คง LED GPIO38 ติดระหว่าง deep sleep แล้วปลดหลังตื่น    | `gpio_hold_en` + `gpio_deep_sleep_hold_en` + ตั้ง output/level ก่อน `gpio_hold_dis`                                                |
| 22  | ตรวจว่าตื่นด้วยอะไร (บน v6)                            | `esp_sleep_get_wakeup_causes()` bitmap + `& ESP_SLEEP_WAKEUP_*` (ไม่ใช่ตัวเดี่ยว)                                                  |
| 23  | อ่าน UART2 บน v6.1 timeout 100 ms                      | `pdMS_TO_TICKS(100)` — ยังเป็น ticks (v6 เปลี่ยนแค่ชนิดพารามิเตอร์ ไม่ใช่ 100 ตรง ๆ)                                               |
| 24  | ผูก stdio เข้ากับ UART1 ผ่าน driver                    | `uart_vfs_dev_use_driver` (ชื่อใหม่ v5.3+ — ไม่ใช่ esp_vfs_dev_*)                                                                  |
| 25  | ปลุก light sleep ด้วยข้อมูลเข้า UART2                  | ต้อง flag ว่าใช้ไม่ได้ — wake source เฉพาะ UART0/1 + นับ rising edges                                                              |
| 26  | probe IC ที่ address 0x68 บน v6.1                      | `i2c_master_probe(bus, 0x68, 100)` + แปลผล: `NOT_FOUND` = ไม่มี ACK, `TIMEOUT` = เช็ค pull-up + ห้าม -1                            |
| 27  | เปิด internal pull-up บัส I2C (โค้ดรองรับ v5.2 ถึง v6) | `.flags.enable_internal_pullup = true` — รูปแบบเดียวทุกเวอร์ชัน                                                                    |
| 28  | รัน I2C ที่ 1 MHz บน S3                                | ต้องเตือน: เกิน spec (~800 kbit/s สูงสุดตาม Datasheet — จำกัดโดย pull-up)                                                          |
| 29  | ส่ง 100 bytes ผ่าน SPI โดยไม่เปิด DMA                  | ต้องแบ่งเป็นหลาย transaction (limit 64 B/transaction) หรือเปิด `SPI_DMA_CH_AUTO` — พร้อมเตือน wrap-around                          |
| 30  | เล่นเสียง 24-bit ผ่าน I2S                              | ต้องตั้ง `mclk_multiple = I2S_MCLK_MULTIPLE_384`                                                                                   |
| 31  | อ่าน ADC ต่อเนื่องด้วย DMA จาก ADC2                    | ต้อง flag ว่าใช้ไม่ได้ — บน S3 DMA/continuous = ADC1 เท่านั้น                                                                      |
| 32  | อ่านแรงดันจริง mV จาก GPIO4 โดยไม่ตั้ง calibration     | ต้องใช้ adc_cali_create_scheme_curve_fitting + adc_cali_raw_to_voltage (ห้ามคูณ Vref เอง — Vref จริง 1000–1200 mV ต่างกันทุก chip) |