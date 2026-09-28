# TWAI (CAN 2.0) — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "driver/twai.h"     // v5.3+/v6: component esp_driver_twai
```

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 31)
- **CAN Specification 2.0 / ISO 11898-1** — 11-bit standard + 29-bit extended ID, **ไม่มี CAN-FD**
- Bit rate **1 Kbit/s – 1 Mbit/s**
- **RX FIFO 64 ไบต์** (ฮาร์ดแวร์) + TX buffer; driver เพิ่ม software queue (`rx_queue_len`/`tx_queue_len`)
- โหมด: Normal / Listen-only (ไม่ออก influence บนบัส) / Self-test (ไม่ต้องมี ACK)
- ส่งพิเศษ: Single-shot (ไม่ retry เมื่อ error), Self-reception
- Acceptance filter: single/dual mode; Error: counters + configurable warning limit + bus-off **พร้อม recovery**, error code capture, arbitration lost capture
- **S3 มี TWAI controller 1 ตัว** (`controller_id` = 0)
- ต้องใช้ **external transceiver** (เช่น SN65HVD230/TJA1050) เสมอ — TWAI ของ chip ไม่ต่อบัส CAN ตรง ๆ

## Structs + Macro มาตรฐาน
```c
// --- General ---
twai_general_config_t g_config = TWAI_GENERAL_CONFIG_DEFAULT(TX_GPIO_NUM, RX_GPIO_NUM, TWAI_MODE_NORMAL);
// ฟิลด์หลัก: .mode .tx_io .rx_io .clkout_io (TWAI_IO_UNUSED) .bus_off_io
//           .tx_queue_len .rx_queue_len .alerts_enabled .clkout_divider .intr_flags
// mode: TWAI_MODE_NORMAL / TWAI_MODE_NO_ACK / TWAI_MODE_LISTEN_ONLY

// --- Timing (macro สำเร็จรูป) ---
twai_timing_config_t t_config = TWAI_TIMING_CONFIG_500KBITS();
// มีให้เลือก: 100KBITS / 125KBITS / 250KBITS / 500KBITS / 800KBITS / 1MBITS

// --- Filter ---
twai_filter_config_t f_config = TWAI_FILTER_CONFIG_ACCEPT_ALL();

// --- Message ---
typedef struct {
    union {
        struct { uint32_t extd:1; uint32_t rtr:1; uint32_t ss:1; uint32_t self:1; uint32_t dlc_non_comp:1; };
        uint32_t flags;
    };
    uint32_t identifier;          // 11/29 บิตตาม extd
    uint8_t data_length_code;     // 0–8 เท่านั้น!
    uint8_t data[8];
} twai_message_t;
// flags: TWAI_MSG_FLAG_NONE / _EXTD / _RTR / _SS / _SELF
```

## Function Signatures (exact)
```c
esp_err_t twai_driver_install(const twai_general_config_t *g_config,
                              const twai_timing_config_t *t_config,
                              const twai_filter_config_t *f_config);
esp_err_t twai_driver_uninstall(void);
esp_err_t twai_start(void);                       // ← ลืม = state STOPPED ส่งไม่ได้!
esp_err_t twai_stop(void);

esp_err_t twai_transmit(const twai_message_t *message, TickType_t ticks_to_wait);   // timeout = TICKS
esp_err_t twai_receive(twai_message_t *message, TickType_t ticks_to_wait);          // timeout = TICKS

esp_err_t twai_read_status(twai_status_info_t *status_info);   // v4.x
esp_err_t twai_get_status_info(twai_status_info_t *status_info); // ชื่อปัจจุบัน (v5+)
// state: TWAI_STATE_STOPPED / RUNNING / BUS_OFF; นับ error/arb_lost/tx_failed/...

esp_err_t twai_initiate_recovery(void);           // ออกจาก bus-off
esp_err_t twai_clear_transmit_queue(void);
esp_err_t twai_clear_receive_queue(void);
esp_err_t twai_reconfigure_alerts(uint32_t alerts_enabled, uint32_t *current_alerts);
esp_err_t twai_read_alerts(uint32_t *alerts, TickType_t ticks_to_wait);
// alerts: TWAI_ALERT_RX_DATA / TX_IDLE / TX_FAILED / BUS_OFF / ERROR_PASSIVE / ABOVE_ERR_WARN ...
```

## Version Marker
- **v6.1+**: เพิ่ม TWAI driver แบบใหม่ **node-handle** (`twai_new_node*`, `twai_node_transmit`) พร้อม Kconfig `CONFIG_TWAI_ISR_IN_IRAM` / `CONFIG_TWAI_IO_FUNC_IN_IRAM` ⚠️ — ชุด `twai_driver_install` (บนนี้) ยังใช้ได้ แต่โปรเจกต์ v6.1+ ควรเทียบ docs ก่อนว่าจะใช้ชุดไหน
- v4.x `twai_read_status()` → เปลี่ยนชื่อเป็น `twai_get_status_info()` ใน v5+

## กับดัก Hallucination
1. **DLC > 8 หรือคิดว่ามี CAN-FD** → S3 = CAN 2.0 classic เท่านั้น
2. `twai_transmit/receive` timeout เป็น **ticks** → `pdMS_TO_TICKS(100)` (คนละเรื่องกับ new I2C ที่เป็น ms!)
3. `twai_driver_install()` แล้วไม่ `twai_start()` → state = STOPPED, transmit ได้ `ESP_ERR_INVALID_STATE`
4. เจอ **bus-off** แล้วรอเดี๋ยวเอง → ต้องเรียก `twai_initiate_recovery()` เอง (ระหว่าง recovery ห้าม transmit)
5. ต่อบัส CAN ตรง ๆ ไม่มี transceiver → ไม่ทำงาน + เสี่ยงพังขา
6. ลืม termination 120 Ω สองปลายบัส → สื่อสารเพี้ยน/ส่งไม่ได้
7. Listen-only mode ใช้ transmit → error (โหมดนี้ฟังอย่างเดียว)
8. `TWAI_TIMING_CONFIG_*KBITS()` คำนวณจาก APB 80 MHz — ปรับ timing เองต้องเข้าใจ brp/seg1/seg2/sjw
9. อ่านบัสรถยนต์: ใช้ `TWAI_MODE_LISTEN_ONLY` + filter เฉพาะ ID ที่ต้องการ (ลด CPU)

## ตัวอย่างมินิมัล (ส่ง/รับ 500 kbit/s)
```c
#include "driver/twai.h"
#define TX_GPIO 4
#define RX_GPIO 5

void can_init(void)
{
    twai_general_config_t g = TWAI_GENERAL_CONFIG_DEFAULT(TX_GPIO, RX_GPIO, TWAI_MODE_NORMAL);
    twai_timing_config_t  t = TWAI_TIMING_CONFIG_500KBITS();
    twai_filter_config_t  f = TWAI_FILTER_CONFIG_ACCEPT_ALL();
    ESP_ERROR_CHECK(twai_driver_install(&g, &t, &f));
    ESP_ERROR_CHECK(twai_start());                                  // ← ห้ามลืม

    twai_message_t msg = { .identifier = 0x123, .data_length_code = 2,
                           .data = {0x11, 0x22} };
    ESP_ERROR_CHECK(twai_transmit(&msg, pdMS_TO_TICKS(100)));       // ticks!

    twai_message_t rx;
    if (twai_receive(&rx, pdMS_TO_TICKS(100)) == ESP_OK) {
        ESP_LOGI("can", "id=0x%lx dlc=%d", (unsigned long)rx.identifier, rx.data_length_code);
    }
}
```