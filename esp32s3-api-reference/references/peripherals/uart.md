# 3-2 UART — v2.1
> Target: ESP32-S3 | ESP-IDF v5.2 – v6.1
> อ้างอิง: TRM v1.8 ch.26 (UART) + ch.10 (Low-power) | Datasheet v2.2 | driver/uart.h (ตรวจจริง v5.3.1/v5.4.1/v5.5.1/v6.0/v6.1) | soc_caps.h v6.1
> เปลี่ยนแปลง v2.1: แปลงโครงสร้างตาม skill v2.7 (§0 TL;DR + §10 จุดตรวจซ้ำ + ผูกรหัส Hot-List) · เพิ่ม CMakeLists esp_driver_uart · รวม signatures เข้า §3–§6

## 0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. **3 ports** (UART0/1/2) ผ่าน GPIO Matrix; UART0 = console (GPIO43/44 default)
2. **timeout ของ `uart_read_bytes`/`uart_wait_tx_done` เป็น ticks ทุกเวอร์ชัน** — v6 เปลี่ยนแค่ชนิดพารามิเตอร์ (`TickType_t` → `uint32_t`) → ใช้ `pdMS_TO_TICKS()`
3. v6.0 ลบ: `UART_FIFO_LEN` → `UART_HW_FIFO_LEN` · `soc/uart_channel.h` → `soc/uart_pins.h`
4. v6.0+ header driver เลิก include FreeRTOS → include `freertos/queue.h` เอง
5. ขาไม่ใช้ใน `uart_set_pin()` → `UART_PIN_NO_CHANGE` (-1) ห้ามส่ง 0
6. **FIFO RAM 1024×8-bit แชร์ทั้ง 3 port** — ขยาย port หนึ่ง = ยืมพื้นที่อีก port
7. ปลุก Light-sleep ได้เฉพาะ **UART0/1** — UART2 ไม่ได้; ไบต์แรก (wakeup byte) ไม่เข้า driver ต้องส่งซ้ำ
8. `uart_write_bytes()` async เฉพาะเมื่อ `tx_buffer_size > 0` ตอน install
9. VFS: `uart_vfs_dev_use_driver()` (v5.3+ — ชื่อเก่า `esp_vfs_dev_uart_*` deprecated)
10. CMakeLists v5.3+: `REQUIRES esp_driver_uart`

## 1. Include & CMakeLists

```c
#include "driver/uart.h"          // API หลัก
#include "driver/uart_vfs.h"      // เฉพาะเมื่อผูก stdio: uart_vfs_dev_use_driver (v5.3+)
#include "freertos/FreeRTOS.h"    // ⚠️ v6.0+ ต้อง include เอง — header ของ driver เลิก include หัว FreeRTOS
#include "freertos/queue.h"       // event queue / xQueueReceive / pdMS_TO_TICKS
```

```cmake
# v5.2:        REQUIRES driver
# v5.3+ / v6.x: แยก component — ระบุตรง:
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_driver_uart)
```

## 2. Hardware Facts (TRM ch.26 + soc_caps v6.1)
- **3 ports**: `UART_NUM_0` (console/log), `UART_NUM_1`, `UART_NUM_2` — เลือกขาได้เกือบทุกตัวผ่าน GPIO Matrix
- UART0 default: **GPIO43 (TX) / GPIO44 (RX)**
- **FIFO แชร์กัน**: RAM 1024×8-bit แชร์ TX/RX FIFO ทั้ง 3 port (default ข้างละ 128 B) — ขยาย UART0 Tx แล้ว UART1 Tx อาจใช้ไม่ได้
- สูงสุด **5 Mbps** (`SOC_UART_BITRATE_MAX`) · data bits 5–8 · stop bits 1/1.5/2 · parity ได้
- โหมดพิเศษ: **RS485 half-duplex** (`uart_set_mode` + DE/RE ผ่าน RTS หรือ DTR), IrDA, AT_CMD char detect, auto-baud
- Clock 3 แหล่งเลือกผ่าน `source_clk` · hw flow ctrl (RTS/CTS) + sw flow ctrl (XON/XOFF) · GDMA สำหรับความเร็วสูง
- **ปลุก Light-sleep เฉพาะ UART0/UART1** (TRM 10.4-3) — Wakeup_Ctrl นับ rising edges ของ rxd_in ครบ threshold+3 จึง wake; ไบต์แรกไม่เข้า driver → protocol ต้องออกแบบให้ส่งซ้ำได้

## 3. Structs & Driver Setup
```c
typedef struct {
    int baud_rate;
    uart_word_length_t data_bits;    // UART_DATA_8_BITS ฯลฯ
    uart_parity_t parity;            // UART_PARITY_DISABLE ฯลฯ
    uart_stop_bits_t stop_bits;      // UART_STOP_BITS_1 ฯลฯ
    uart_hw_flowcontrol_t flow_ctrl; // UART_HW_FLOWCTRL_DISABLE / CTS_RTS
    uint8_t rx_flow_ctrl_thresh;
    union { uart_sclk_t source_clk; uint32_t clk_source; };  // UART_SCLK_DEFAULT
    // หมายเหตุ: มี field เสริม rx_glitch_filt_thresh บน S3 (ตรวจ header เวอร์ชันที่ใช้)
} uart_config_t;

esp_err_t uart_driver_install(uart_port_t uart_num, int rx_buffer_size, int tx_buffer_size,
                              int queue_size, QueueHandle_t *uart_queue, int intr_alloc_flags);
esp_err_t uart_param_config(uart_port_t uart_num, const uart_config_t *uart_config);
esp_err_t uart_set_pin(uart_port_t uart_num, int tx_io_num, int rx_io_num,
                       int rts_io_num, int cts_io_num);   // ขาไม่ใช้ = UART_PIN_NO_CHANGE (-1)
esp_err_t uart_set_mode(uart_port_t uart_num, uart_mode_t uart_mode);  // UART_MODE_UART / RS485_HALF_DUPLEX
bool uart_is_driver_installed(uart_port_t uart_num);
esp_err_t uart_driver_delete(uart_port_t uart_num);
```

## 4. I/O Patterns (⚠️ timeout = ticks ทุกเวอร์ชัน)
```c
int      uart_write_bytes(uart_port_t uart_num, const void *src, size_t size);  // คืนจำนวน byte (int ไม่ใช่ esp_err_t)
int      uart_read_bytes(uart_port_t uart_num, void *buf, uint32_t length, TickType_t ticks_to_wait);
         // ⚠️ v6.0: ชนิด ticks_to_wait เปลี่ยน TickType_t → uint32_t — หน่วยยังเป็น ticks
esp_err_t uart_wait_tx_done(uart_port_t uart_num, TickType_t ticks_to_wait);    // v6.0: uint32_t เช่นกัน
esp_err_t uart_get_buffered_data_len(uart_port_t uart_num, size_t *size);
esp_err_t uart_flush_input(uart_port_t uart_num);
esp_err_t uart_set_wakeup_threshold(uart_port_t uart_num, int wakeup_threshold); // นับ rising edges (S3 ขั้นต่ำ 3)
```
- `uart_write_bytes` เป็น async **เฉพาะเมื่อ** `tx_buffer_size > 0` ตอน install — ตั้ง 0 = block จนส่งครบ
- อ่านแบบมี timeout: `uart_read_bytes(UART_NUM_1, buf, len, pdMS_TO_TICKS(100))`

## 5. Event Queue Pattern
```c
// ตอน install ส่ง queue เข้าไป: uart_driver_install(port, 2048, 0, 20, &uart_queue, 0)
uart_event_t evt;
if (xQueueReceive(uart_queue, &evt, portMAX_DELAY)) {
    switch (evt.type) {
    case UART_DATA:        /* evt.size ไบต์ถึงแล้ว — uart_read_bytes */ break;
    case UART_FIFO_OVF:    /* overflow — uart_flush_input */             break;
    case UART_BUFFER_FULL: /* ring buffer เต็ม */                        break;
    default: break;
    }
}
```

## 6. VFS (ผูก stdio)
```c
void uart_vfs_dev_use_driver(int uart_num);        // v5.3+ (แทน esp_vfs_dev_uart_use_driver — deprecated)
void uart_vfs_dev_use_nonblocking(int uart_num);
```

## 7. กับดักที่พบบ่อย (ผูก HALLUCINATION-HOT-LIST v3.1)
1. คิดว่า v6 รับ timeout เป็น ms ตรง ๆ → ผิดทุกเวอร์ชัน — เป็น ticks → **UART-01**
2. v6.0 ใช้ `UART_FIFO_LEN` / `soc/uart_channel.h` → ถูกลบ → **UART-03**
3. v6.0+ ลืม include `freertos/*.h` เอง → compile error → **UART-04**
4. ใช้ `esp_vfs_dev_uart_use_driver()` → deprecated v5.3 → **UART-05**
5. `uart_set_pin` ส่ง 0 ให้ RTS/CTS → ต้อง `-1` (`UART_PIN_NO_CHANGE`) → **UART-02**
6. `uart_write_bytes` คืน `int` ไม่ใช่ `esp_err_t` — ห้ามครอบ `ESP_ERROR_CHECK` ตรง ๆ
7. ลืม `uart_driver_install` ก่อนใช้งาน → error
8. ใช้ UART0 ทำงานอื่นแล้ว log หาย (console จม)
9. ปลุก Light-sleep ด้วย UART2 → ไม่ได้ → **UART-06**
10. คาดว่า write เป็น async → เฉพาะเมื่อ `tx_buffer_size > 0` → **UART-08**
11. คิดว่า FIFO 128 B ของแต่ละ port อิสระ → แชร์ RAM เดียวกัน → **UART-09**
12. `uart_set_pin` 5 args บน v6 → ยังใช้ได้ (wrapper ไป `_uart_set_pin6`) → **UART-07**

## 8. ตัวอย่างมินิมัล
```c
#include "driver/uart.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define UART_PORT UART_NUM_1
#define TXD 17
#define RXD 18

void app_main(void)
{
    const uart_config_t cfg = {
        .baud_rate = 115200,
        .data_bits = UART_DATA_8_BITS,
        .parity = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
        .source_clk = UART_SCLK_DEFAULT,
    };
    ESP_ERROR_CHECK(uart_driver_install(UART_PORT, 2048, 0, 0, NULL, 0));
    ESP_ERROR_CHECK(uart_param_config(UART_PORT, &cfg));
    ESP_ERROR_CHECK(uart_set_pin(UART_PORT, TXD, RXD,
                                 UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE));

    const char *msg = "hello\r\n";
    int sent = uart_write_bytes(UART_PORT, msg, strlen(msg));   // คืน int

    uint8_t buf[64];
    int n = uart_read_bytes(UART_PORT, buf, sizeof(buf), pdMS_TO_TICKS(100));  // ticks!
    if (n > 0) { /* ประมวลผล buf[0..n-1] */ }
}
```

## 9. v6.x Notes
- `ticks_to_wait` ของ `uart_read_bytes`/`uart_wait_tx_done` เปลี่ยนชนิด `TickType_t` → `uint32_t` (หน่วยยัง ticks)
- ลบ: `UART_FIFO_LEN` → `UART_HW_FIFO_LEN`; `soc/uart_channel.h` → `soc/uart_pins.h` (macro เช่น `UART_NUM_0_TXD_DIRECT_GPIO_NUM` → `U0TXD_GPIO_NUM`)
- header driver เลิก include FreeRTOS → include เอง
- + `uart_wakeup_setup(uart_port_t, const uart_wakeup_cfg_t*)` (ชุดใหม่ คู่ `uart_set_wakeup_threshold` เดิม)
- `uart_set_pin` 5 args = wrapper ของ `_uart_set_pin6` (7 args รองรับ DTR/DSR)

## 10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
- `_uart_set_pin6()` — ยืนยันมีใน v6.1 แล้ว ยังไม่เช็ค v5.2–v5.5 → เปิด uart.h ของเวอร์ชันเป้าหมายก่อนใช้ DTR/DSR
- `uart_wakeup_cfg_t` — โครงสร้าง field ตามเวอร์ชัน (v6 ชุดใหม่)
- `rx_glitch_filt_thresh` ใน `uart_config_t` บน S3 — มีตั้งแต่เวอร์ชันไหน
- `uart_vfs.h` — ชื่อไฟล์/ชุดฟังก์ชัน VFS ใน v5.2 (ก่อน rename ใน v5.3)