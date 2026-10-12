# CHANGELOG — esp32s3-api-reference
> แยกออกจาก SKILL.md ตั้งแต่ v2.7.0 — บันทึกทุกการเปลี่ยนแปลงที่นี่ (ไม่ใส่ใน SKILL.md อีก)


## v2.13.0 — Phase 2: esp_timer.md ตรวจเชิงลึกกับ header v6.1
- esp_timer.md v1.1 → v2.0:
  - ลบ ESP_TIMER_EXT (ไม่มีใน header v6.1 — enum มีแค่ TASK/ISR(conditional)/MAX)
  - esp_timer_get_next_alarm แก้เป็น int64_t (เดิม uint64_t — ผิด) + INT64_MAX semantics
  - start_once_at ลายเซ็นจริง = (timer, alarm_us) 2 พารามิเตอร์ (เดิมเดา 3 — ผิด)
    + ยืนยัน start_periodic_at/restart_at ชื่อพารามิเตอร์ first_alarm_us
  - เพิ่ม API จาก header: stop_blocking (v6.1), is_active, get_period, get_expiry_time,
    get_next_alarm_for_wake_up, early_init, isr_dispatch_need_yield, new_etm_alarm_event
  - ยืนยัน CMakeLists: REQUIRES/PRIV_REQUIRES esp_timer เป็น mandatory (docs ทางการ)
  - ยืนยัน 50 µs periodic / ~20 µs one-shot จาก docs ทางการ (ปิดคิว §10 เดิม)
  - skip_unhandled_events ปรับความหมายตาม header: light-sleep เท่านั้น
  - เพิ่ม: stop() ไม่ preempt callback ที่รันอยู่ (ใช้ stop_blocking/is_active),
    dump พฤติกรรมตาม CONFIG_ESP_TIMER_PROFILING, yield mechanism ต่างจาก gptimer
- ผลตรวจรีวิว DeepSeek: ถูก 9/9 — ต่างจาก gptimer (ผิด allow_pd); จุดผิดของ v1.1
  ทั้ง 3 ถูกกันไว้ที่ §10 ด้วย ⚠️ ตั้งแต่เฟส 1 จึงไม่รั่งสู่ผู้ใช้
- HALLUCINATION-HOT-LIST: เพิ่มหมวด ESP Timer 12 ข้อ (ESPT-01–12, รอ renumber)
- SKILL.md: File Index esp_timer.md → v2.0 (Phase 2) + Version Decision Table + case study
- สถานะ Phase 2: 2/15 (esp_timer, gptimer) — คิวถัดไปตามไฟล์ที่ผู้ใช้ส่ง

## v2.12.0 — Phase 2 เริ่มต้น: gptimer.md ตรวจเชิงลึกกับ header v6.1
- gptimer.md v1.1 → v2.0:
  - แก้ set/get_raw_count เป็น uint64_t (เดิม unsigned long — ผิด)
  - แก้ component v5.3+ เป็น esp_driver_gptimer (เดิมเขียน esp_driver_timer — ผิด)
  - แยก ETM เป็นหัวข้อเฉพาะ: header จริงคือ driver/gptimer_etm.h (มาตั้งแต่ v5.2)
    + เพิ่ม gptimer_new_etm_task + event_type/task_type
  - เพิ่ม API: gptimer_get_resolution (RC_FAST calibration), gptimer_get_captured_count
    (ไม่ trigger software capture), new_timer คืน ESP_ERR_NOT_FOUND เมื่อ timer หมด
  - เพิ่ม Kconfig: CONFIG_GPTIMER_CTRL_FUNC_IN_IRAM (6 ฟังก์ชัน) / CONFIG_GPTIMER_ISR_CACHE_SAFE
    + gotcha alarm_config ต้อง static ไม่ใช่ const เมื่อเปิด IRAM
  - เพิ่ม: แก้ edata->alarm_value ใน callback ได้ทันที / start INVALID_STATE สองกรณี
  - "~5 µs ขั้นต่ำ" ปรับสถานะเป็น heuristic (ไม่พบแหล่งทางการ)
  - TRM ch.12 ยืนยันครบ: 2×2=4 ตัว, 54-bit, prescaler 2–65536 (ตั้ง 1 = หาร 2)
- บทเรียนสำคัญ: รีวิว AI ภายนอก (DeepSeek) อ้างว่า allow_pd ไม่มีใน gptimer_config_t
  — ตรวจกับ header v6.1 พบว่ามีจริง → ยืนยันกฎเหล็ก "ต้องตรวจกับ header จริงเสมอ"
  (ซ้ำเคส adc_cali_delete_scheme_curve_fitting) — ผลการตรวจรีวิว: ถูก 6/7 ข้อ
- HALLUCINATION-HOT-LIST: เพิ่มหมวด GPTimer 9 ข้อ (GPT-01–09, รอ renumber)
- SKILL.md: File Index gptimer.md → v2.0 (Phase 2) + Version Decision Table แถว Timer
- สถานะ Phase 2: 1 ไฟล์เสร็จ (gptimer) — คิวถัดไป: esp_timer.md

## v2.11.0 — Phase 1 เสร็จสมบูรณ์: แปลงโครงสร้างชุดสุดท้าย (15/15)
- esp-dsp.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — จัดระเบียบตาราง 3 ชั้น PIE,
  FFT/vector/FIR API, กับดัก 8 ข้อ; คิวตรวจ §10 เช่น fir_f32_t โครงสร้าง,
  biquad/matrix signature, Kconfig ขนาด FFT สูงสุด
- heap.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — แยก Allocation/Diagnostic API;
  คิวตรวจ §10 เช่น esp_dma_capable_malloc ที่ถูกลบใน v6 (จาก migration guide —
  ยังไม่ verify), GDMA+PSRAM เงื่อนไข, ชุด MALLOC_CAP_* ครบไหม
- ota.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — แยก High-level/Low-level/Rollback
  เป็น §4/§5; คิวตรวจ §10 เช่น partial_http_download (v5.3+ ⚠️),
  ชื่อ error codes, esp_ota_app_desc.h path ใน v6
- wifi.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — ลำดับ init 9 ขั้น + Event System;
  คิวตรวจ §10 เช่น wifi_config_t ฟิลด์เต็ม, event data structs, Kconfig ชื่อจริง
- SKILL.md: อัปเดต File Index — 4 ไฟล์ → "แปลงโครงสร้างแล้ว (Phase 1) รอตรวจเชิงลึก"
- HALLUCINATION-HOT-LIST.md: ยังไม่เพิ่มข้อใหม่ — §7 ของทั้งสี่ไฟล์รอผูกรหัสในรอบถัดไป
- สถานะ Phase 1: เสร็จ 15/15 ไฟล์ — พร้อมเริ่ม Phase 2 (ตรวจเชิงลึกกับ header v6.1)

## v2.10.0 — Phase 1 ต่อเนื่อง: แปลงโครงสร้างไฟล์กลุ่ม Storage/Network
- fatfs_sdmmc.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — จัดระเบียบเนื้อหาเดิมทั้งหมด
  (mount all-in-one, SDMMC/SPI mode, VFS/POSIX, กับดัก 9 ข้อ); คิวตรวจ §10 เช่น
  get_dma_info ที่ถูกลบใน v6 (จาก migration guide — ยังไม่ verify), สถานะ
  esp_vfs_fat_sdmmc_mount ใน v6.1, fsync ผ่าน VFS
- http_client.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — แยก Pattern 1 (perform) /
  Pattern 2 (streaming) เป็น §4/§5; คิวตรวจ §10 เช่น config ฟิลด์เต็ม,
  CONFIG_MBEDTLS_CERTIFICATE_BUNDLE default
- http_server.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — แยก Lifecycle API / In-Handler
  API เป็น §4/§5; คิวตรวจ §10 เช่น httpd_resp_set_status ลายเซ็น, config defaults,
  esp_https_server สถานะ
- mqtt.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — คิวตรวจ §10 เช่น unsubscribe qos
  (⚠️ จากไฟล์เดิม), config ฟิลด์เต็ม v6.1, events ครบไหม
- nvs.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — คิวตรวจ §10 เช่น v5.4 opaque pointer
  ยืนยันกับ header, 2-call pattern, จำกัด 15 ตัวอักษร
- SKILL.md: อัปเดต File Index — 5 ไฟล์ → "แปลงโครงสร้างแล้ว (Phase 1) รอตรวจเชิงลึก"
- HALLUCINATION-HOT-LIST.md: ยังไม่เพิ่มข้อใหม่ — §7 ของทั้งห้าไฟล์รอผูกรหัสในรอบถัดไป
- สถานะ Phase 1: เสร็จ 11/15 ไฟล์ (เหลือ 4)
## v2.9.0 — Phase 1 ต่อเนื่อง: แปลงโครงสร้างไฟล์ MCPWM/PCNT/RMT/TWAI
- mcpwm.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — จัดระเบียบเนื้อหาเดิมทั้งหมด (pipeline, structs,
  action macros, capture/fault, ตัวอย่าง servo); คิวตรวจ §10 เช่น varg set_actions_* ใน v6,
  io_od_mode/pull_up ที่ถูกลบ (จาก migration guide — ยังไม่ verify), dead-time API
- pcnt.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — คิวตรวจ §10 เช่น accum_count พฤติกรรม,
  glitch filter ns vs APB cycles, watch point threshold API
- rmt.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — คิวตรวจ §10 เช่น rmt_rx_channel_config_t ฟิลด์,
  encoder เอง (rmt_encode_t/RMT_ENCODER_FUNC_ATTR), Kconfig ชื่อจริง, io_od_mode (v6)
- twai.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — คิวตรวจ §10 เช่น node-handle API v6.1+,
  Kconfig, filter single/dual mode, twai_status_info_t ฟิลด์
- SKILL.md: อัปเดต File Index — 4 ไฟล์ → "แปลงโครงสร้างแล้ว (Phase 1) รอตรวจเชิงลึก"
- HALLUCINATION-HOT-LIST.md: ยังไม่เพิ่มข้อใหม่ — §7 ของทั้งสี่ไฟล์รอผูกรหัสในรอบถัดไป
- สถานะ Phase 1: เสร็จ 6/15 ไฟล์ (เหลือ 9)

## v2.9.0 — Phase 1 ต่อเนื่อง: แปลงโครงสร้างไฟล์ MCPWM/PCNT/RMT/TWAI
- mcpwm.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — จัดระเบียบเนื้อหาเดิมทั้งหมด (pipeline, structs,
  action macros, capture/fault, ตัวอย่าง servo); คิวตรวจ §10 เช่น varg set_actions_* ใน v6,
  io_od_mode/pull_up ที่ถูกลบ (จาก migration guide — ยังไม่ verify), dead-time API
- pcnt.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — คิวตรวจ §10 เช่น accum_count พฤติกรรม,
  glitch filter ns vs APB cycles, watch point threshold API
- rmt.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — คิวตรวจ §10 เช่น rmt_rx_channel_config_t ฟิลด์,
  encoder เอง (rmt_encode_t/RMT_ENCODER_FUNC_ATTR), Kconfig ชื่อจริง, io_od_mode (v6)
- twai.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — คิวตรวจ §10 เช่น node-handle API v6.1+,
  Kconfig, filter single/dual mode, twai_status_info_t ฟิลด์
- SKILL.md: อัปเดต File Index — 4 ไฟล์ → "แปลงโครงสร้างแล้ว (Phase 1) รอตรวจเชิงลึก"
- HALLUCINATION-HOT-LIST.md: ยังไม่เพิ่มข้อใหม่ — §7 ของทั้งสี่ไฟล์รอผูกรหัสในรอบถัดไป
- สถานะ Phase 1: เสร็จ 6/15 ไฟล์ (เหลือ 9)

## v2.8.0 — Phase 1 เริ่มต้น: แปลงโครงสร้างไฟล์ Timer
- esp_timer.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — จัดระเบียบเนื้อหาเดิมทั้งหมด
  ไม่เพิ่มข้อมูลเทคนิคใหม่; รวมจุดค้างตรวจ 9 รายการไว้ที่ §10 (เช่น ฟังก์ชัน *_at,
  ESP_TIMER_EXT, CMakeLists REQUIRES)
- gptimer.md → v1.1: แปลงโครงสร้างใหม่ §0–§10 — คิวตรวจ §10 เช่น ชนิดพารามิเตอร์
  set/get_raw_count ต่อเวอร์ชัน, component path ของ v5.2, โครงสร้าง ETM config
- SKILL.md: อัปเดต File Index — esp_timer, gptimer → "แปลงโครงสร้างแล้ว (Phase 1) รอตรวจเชิงลึก"
- HALLUCINATION-HOT-LIST.md: ยังไม่เพิ่มข้อใหม่ — §7 ของทั้งสองไฟล์รอผูกรหัสในรอบถัดไป
- สถานะ Phase 1: เสร็จ 2/15 ไฟล์ (เหลือ 13)

## สถานะปัจจุบัน (หลัง v2.7.2)

| ไฟล์                      | เวอร์ชัน | สถานะ                                          |
| ------------------------- | -------- | ---------------------------------------------- |
| SKILL.md                  | v2.7.2   | Router                                         |
| HALLUCINATION-HOT-LIST.md | v3.2     | Checklist Edition — 111 ข้อ / 23 กลุ่ม API     |
| gpio.md                   | v2.2     | reviewed + โครงสร้างใหม่                       |
| uart.md                   | v2.1     | reviewed + โครงสร้างใหม่                       |
| spi.md                    | v2.1     | reviewed + โครงสร้างใหม่                       |
| i2c.md                    | v2.1     | reviewed + โครงสร้างใหม่                       |
| ledc.md                   | v2.1     | reviewed + โครงสร้างใหม่                       |
| i2s.md                    | v2.1     | reviewed + โครงสร้างใหม่                       |
| adc.md                    | v2.0.2   | reviewed + โครงสร้างใหม่ (major review รอบนี้) |
| sleep.md                  | v2.2     | reviewed + โครงสร้างใหม่ (แปลงรอบนี้)          |
| อื่นๆ อีก 15 ไฟล์         | v1.x     | รอรีวิว                                        |
## v2.7.2 — sleep.md แปลงโครงสร้าง + adc.md major review
- **adc.md v1.x→v2.0 (major review):** แก้ hallucination v1.x สองจุด — (1) "calibration สร้างต่อ channel" → จริง field `chan` เป็น reserved ไม่มีผล ผูก unit+atten เท่านั้น (docs + header) (2) "DB_12 วัดได้ 0–3.1V" → จริง **~2450 mV บน S3** (Datasheet — 3.1V คือของ ESP32 DB_11) · เพิ่ม section Continuous (DMA): **DMA = ADC1 เท่านั้นบน S3** (TRM 39.3.7.2 + soc_caps) · component = `esp_adc` (ไม่ใช่ esp_driver_* — แก้ path ที่เคยแนะนำผิด) · Vref จริง 1000–1200 mV · `adc_oneshot_get_calibrated_result` (v6.1) · `.format` deprecated (v6.1) · `adc_continuous_flush_pool` · ตัวอย่างตัด `.chan` + เพิ่มการเฉลี่ย sample · ตรวจกับ header v6.1 จริง 3 ไฟล์ + adc_cali_schemes.h (S3 = curve fitting เท่านั้น)
- **sleep.md v2.1→v2.2:** แปลงโครงสร้าง template ใหม่ (§0–§10) + **แก้ pitfall ค้างจาก v2.1**: "esp_sleep_enable_timer_wakeup คืน uint64_t ห้ามครอบ ESP_ERROR_CHECK" → คืน `esp_err_t` ครอบได้ (ข้อความในไฟล์ขัดกับที่แก้ไว้ใน Hot-List ตั้งแต่ SKILL v2.1) · ย้ายจุดตรวจซ้ำ 2 ข้อจาก SKILL §8 ลง sleep.md §10 · ผูกกับดักกับรหัส SLEEP-01–08
- **HALLUCINATION-HOT-LIST.md v3.1→v3.2:** หมวด ADC ขยาย 2→**8 ข้อ** (ADC-03 การ mapping GPIO↔CH, ADC-04 legacy+component esp_adc, ADC-05 chan=reserved, ADC-06 Vref+cali, ADC-07 DMA=ADC1, ADC-08 2450 mV) → รวม **111 ข้อ** · SLEEP group header เชื่อม sleep.md v1.2
- **SKILL.md v2.7.1→v2.7.2:** File Index อัปเดต adc/sleep + ย้ายรายการ sleep ออกจาก §8
- **ตรวจซ้ำด้วย header จริง `adc_cali_scheme.h` (v6.1): ยืนยันข้อเสนอ DeepSeek ที่เคยปฏิเสธ** — `adc_cali_delete_scheme_curve_fitting()` มีจริงใน header (ผลตรวจรอบแรกที่ปฏิเสธ = ผิด เพราะอ้างจากความจำไม่ใช่ header — ถอนคำ) → adc.md §5 แสดง delete 2 ทางเลือก (เฉพาะ scheme + generic) · แก้เหตุผล field `chan`: header ระบุใช้เฉพาะ chip ที่ define `SOC_ADC_CALIB_CHAN_COMPENS_SUPPORTED` และ S3 ไม่ define (soc_caps v6.1) → บน S3 ไม่มีผล · ยืนยัน line fitting มีใน API แต่ถูก guard ออกบน S3 (`default_vref` = ESP32 only) · adc.md → v2.0.2
- สถานะ: reviewed **8/23** — เหลือ 15 ไฟล์ v1.x รอรีวิว


## สถานะปัจจุบัน (หลัง v2.7.1)

| ไฟล์                          | เวอร์ชัน | สถานะ                                      |
| ----------------------------- | -------- | ------------------------------------------ |
| SKILL.md                      | v2.7.1   | Router (แยก CHANGELOG/SMOKE-TESTS แล้ว)    |
| HALLUCINATION-HOT-LIST.md     | v3.1     | Checklist Edition — 105 ข้อ / 23 กลุ่ม API |
| CHANGELOG.md / SMOKE-TESTS.md | —        | ไฟล์แยก (ย้ายจาก SKILL.md §9–§10)          |
| gpio.md                       | v2.2     | reviewed + โครงสร้างใหม่                   |
| uart.md                       | v2.1     | reviewed + โครงสร้างใหม่                   |
| spi.md                        | v2.1     | reviewed + โครงสร้างใหม่                   |
| i2c.md                        | v2.1     | reviewed + โครงสร้างใหม่                   |
| ledc.md                       | v2.1     | reviewed + โครงสร้างใหม่                   |
| i2s.md                        | v2.1     | reviewed + โครงสร้างใหม่                   |
| sleep.md                      | v2.1     | reviewed (โครงสร้างเก่า — รอแปลง)          |
| อื่นๆ อีก 16 ไฟล์             | v1.x     | รอรีวิว                                    |

## v2.7.1 — Reference โครงสร้างใหม่ครบทุกไฟล์ reviewed
- **แปลง 5 ไฟล์ reviewed ที่เหลือเป็น template ใหม่:** gpio.md v2.1→v2.2 · uart.md v2.0→v2.1 · i2c.md v2.0→v2.1 · spi.md v2.0→v2.1 · ledc.md v2.0→v2.1 — ทุกไฟล์มี §0 TL;DR + §7 กับดักผูกรหัส Hot-List + §10 จุดตรวจซ้ำกับ header จริง
- **ledc.md** ปรับลำดับ section ให้เข้า template: เพิ่ม §5 Duty Workflow + §6 Fade & Callback (กับดักย้าย §5→§7) — ลิงก์ Hot-List ของ LEDC ทุกข้อยังชี้ถูกตำแหน่ง
- **spi.md** กู้ §5 Structs และ §8 ตัวอย่างมินิมัลกลับมาเต็ม (ไฟล์ v2.0 ที่บันทึกไว้เหลือแค่หมายเหตุอ้างอิง — เนื้อหาหาย)
- **gpio/uart/i2c.md** เพิ่ม CMakeLists (`esp_driver_gpio`/`esp_driver_uart`/`esp_driver_i2c`) ที่ขาด + uart/i2c เพิ่ม §9 v6.x Notes
- **HALLUCINATION-HOT-LIST.md v3.0→v3.1:** ใส่เวอร์ชัน reference ใน group header 6 กลุ่ม reviewed · เพิ่ม § links กลุ่ม GPIO/UART/I2C และ SPI-01–04 — ยัง 105 ข้อ ไม่มีข้อเพิ่ม/ลด
- **SKILL.md v2.7.0→v2.7.1:** File Index อัปเดตเวอร์ชัน · ย้ายรายการ §8 ที่เกี่ยวกับ gpio/uart/i2c ลง §10 ของไฟล์แต่ละตัว
- **แก้ตัวเลขสถิติ:** reference มี **23 ไฟล์** (12 peripherals + 4 protocols + 2 storage + 4 system + 1 dsp — ตัวเลข "25" เดิมนับรวมไฟล์ราก) — reviewed 7 / รอรีวิว **16** (ตัวเลข "เหลือ 17–19" ใน entry เก่าผิดทั้งสาย)
- เหลือ: sleep.md (v2.1) ยังเป็นโครงสร้างเก่า — แปลงในรอบถัดไป

## v2.7.0 — โครงสร้างใหม่ (Router + Checklist)
- **SKILL.md → Router:** ตัด §9 Smoke Tests + §10 Changelog ออกเป็นไฟล์แยก · เพิ่ม §2 Dependency Map (งาน→reference→checklist) · แก้กฎข้อ 6 และตารางหมวด Hot-List ที่ค้างเป็น "81 ข้อ" ตั้งแต่ v2.4 · VDT เพิ่มแถว SPI/LEDC/I2S สรุปจาก review ล่าสุด
- **HALLUCINATION-HOT-LIST.md → v3.0 Checklist Edition:** จัดกลุ่มตาม API 23 กลุ่ม (รหัส API-XX) · เพิ่ม tag [COMPILE]/[RUNTIME]/[SILENT] · ยุบข้อซ้ำ: 32+102→I2S-01, 33+98→I2S-02 (+reconfig แยกเป็น I2S-11), 34+105→I2S-03 — 107 → 105 ข้อ ครบถ้วน (มี "(เดิม N)" กำกับทุกข้อ)
- **i2s.md → v2.1:** เพิ่ม §0 TL;DR + §10 จุดตรวจซ้ำกับ header จริง + ผูก §7 กับดักกับรหัส I2S-01–11
- **ไฟล์ใหม่:** CHANGELOG.md (ไฟล์นี้) + SMOKE-TESTS.md

## ประวัติเวอร์ชัน (ย้ายจาก SKILL.md §10 เดิม)

### v2.6.0
- i2s.md major review → v2.0: 24-bit → mclk 384, preload_data ต้อง READY state + bytes_loaded, สูตร DMA 4092 + dma_desc_num/polling, ตารางโหมด per-controller (PDM = I2S0 เท่านั้น), PHILIPS naming (PHILIP = alias), event dma_buf (v5.3+), esp_driver_i2s, S3 16-bit DMA alignment per slot, PDM 2 slots/16-bit/multi-line (v6.1), TDM 16 slots/total_slot, HW v2 slot fields, i2s_channel_tune_rate (v6.x)
- HALLUCINATION-HOT-LIST.md: เพิ่มหมวด L ข้อ 98–107 → รวม 107 ข้อ
- Reviewed files: gpio (v2.1), uart (v2.0), i2c (v2.0), sleep (v2.1), spi (v2.0), ledc (v2.0)

### v2.5.0
- ledc.md major review → v2.0: แก้ hallucination v1.x สองจุด (ledc_cbs_t = callback เดียว .fade_cb / ledc_set_fade_with_time = 4 พารามิเตอร์), hpoint = เฟสเริ่มพัลส์ (TRM), clock source global ทุก timer, duty=2^res พังที่ 14-bit, เพิ่ม ledc_find_suitable_duty_resolution, v6.x removals/deprecations, sleep_mode field (v6.1), CMakeLists esp_driver_ledc, ledc_fade_stop บน S3, f_max = ค่าที่ CLK_DIV=1, ตัวอย่างใช้ LEDC_FADE_WAIT_DONE
- HALLUCINATION-HOT-LIST.md: เพิ่มหมวด K ข้อ 90–97 → รวม 97 ข้อ

### v2.4.0
- spi.md major review → v2.0: clock แยกตาม IOMUX/GPIO Matrix/full-duplex, 64 B = per-transaction + กลไก wrap-around W0, section ใหม่ DMA Buffer Rules + spi_bus_dma_memory_alloc() (v6.x), GP-SPI3 ไม่รองรับ octal, CMakeLists esp_driver_spi, v6.x notes (CONFIG_SPI_MASTER_IN_IRAM, esp_driver_dma), เพิ่ม field ใหม่ของ spi_bus_config_t จาก header v6.1 จริง
- HALLUCINATION-HOT-LIST.md: เพิ่มหมวด J ข้อ 82–89 → รวม 89 ข้อ

### v2.3
- i2c.md v1→v2.0 — ตรวจ header จริง (v5.4.1/v5.5.1/v6.1): ยืนยัน `.flags.enable_internal_pullup` ไม่เคยย้าย (แก้ marker เดิม "v5.2–v5.3 ฟิลด์ตรง" ที่ผิด + ปฏิเสธข้อเสนอ reviewer ที่อ้าง v5.4+ ย้ายขึ้นระดับบนสุด — ที่มาคือ doxygen cross-ref แบน ๆ) · แก้ marker `-1` auto-select (ยืนยัน v5.4.1+ ไม่ใช่ v5.5+) และ `multi_buffer_transmit` (มีตั้งแต่ v5.4.1 ไม่ใช่ v6.1) · เพิ่ม section slave (v5.4 redesign ตาม migration guide) · เติม Hardware Facts จาก TRM 27 (RAM 32×8, clock XTAL/RC_FAST, ~800 kbit/s) · เพิ่ม `execute_defined_operations` (v5.5+), `device_change_address`, `wait_all_done`, `allow_pd`/`intr_priority`/`trans_queue_depth` · probe diagnostics (NOT_FOUND vs TIMEOUT + ห้าม -1) · Hot List 75→81 ข้อ (หมวด I) · Smoke Tests เพิ่ม 26–28 · VDT แถว I2C เขียนใหม่
### v2.2
- uart.md v1→v2.0 — ตรวจ header จริงรายเวอร์ชัน (v5.3.1/v5.4.1/v5.5.1/v6.0/v6.1): ยืนยัน timeout ของ `uart_read_bytes`/`uart_wait_tx_done` เป็น ticks ทุกเวอร์ชัน (v6.0 เปลี่ยนเฉพาะชนิด `TickType_t`→`uint32_t`) — แก้ VDT แถวเดิมที่ระบุ v6 = ms ซึ่งผิด · เพิ่ม v6.0 breaking (UART_FIFO_LEN→UART_HW_FIFO_LEN, soc/uart_channel.h→soc/uart_pins.h, uart_set_pin→_uart_set_pin6) · เติม Hardware Facts จาก TRM บท 26 · เพิ่ม event-queue pattern + API ที่ขาด · Hot List 68→75 ข้อ (หมวด H) · Smoke Tests เพิ่ม 23–25
### v2.1
- gpio.md v2→v2.1 — เพิ่ม wake-cause API (`esp_sleep_get_wakeup_cause` + marker v6 `_causes` bitmap), กับดัก `gpio_hold_dis` หลังตื่น Deep-sleep · Hot List 61→68 ข้อ (หมวด G) · VDT เพิ่มแถว wakeup cause getter · Smoke Tests เพิ่ม 20–22
### v2.0
- Hot List แยกออกเป็น `HALLUCINATION-HOT-LIST.md` (58→61 ข้อ) · VDT เพิ่ม ext1/_io, sleep type rename, UART wake v6, TWAI node-handle v6.1, MCPWM dead-time · เพิ่ม §8 จุดตรวจซ้ำ + §9 Smoke Tests รวม 19 ข้อ
### v1.1
- `sleep.md` v1→v2.1 แก้ตาม external review + verify — Hot List ใส่เพิ่มข้อ 39–41
### v1.0
- รอบ 1 (gpio/uart/spi/i2c/wifi/http_client/nvs) → รอบ 2 (ledc/gptimer/adc/mqtt) → รอบ 3 (esp_timer/rmt/i2s) → รอบ 4 (sleep/heap/esp_dsp/http_server/pcnt) → รอบ 5 (mcpwm/twai/fatfs_sdmmc/ota) = ครบ 25 ไฟล์