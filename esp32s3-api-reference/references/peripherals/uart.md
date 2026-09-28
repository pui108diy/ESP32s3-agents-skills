# UART Driver — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "driver/uart.h"
#include "driver/uart_vfs.h"   // เฉพาะเมื่อผูก stdio (uart_vfs_dev_use_driver)
```
> v5.3+/v6: code อยู่ใน component `esp_driver_uart` (header ชื่อเดิม)

## Hardware Facts (จาก TRM)
- S3 มี **3 ports**: `UART_NUM_0` (console/log ปกติ), `UART_NUM_1`, `UART_NUM_2`
- TX/RX ผ่าน GPIO Matrix → เลือก pin ได้เกือบทุกตัว
- UART0 default: **GPIO43 (TX) / GPIO44 (RX)**

## Struct
```c
typedef struct {
    int baud_rate;                    // เช่น 115200
    uart_word_length_t data_bits;     // UART_DATA_8_BITS
    uart_parity_t parity;             // UART_PARITY_DISABLE
    uart_stop_bits_t stop_bits;       // UART_STOP_BITS_1
    uart_hw_flowcontrol_t flow_ctrl;  // UART_HW_FLOWCTRL_DISABLE
    uart_sclk_t source_clk;           // UART_SCLK_DEFAULT (มาตั้งแต่ v5.0)
} uart_config_t;
```

## Function Signatures (exact)
```c
esp_err_t uart_driver_install(uart_port_t uart_num, int rx_buffer_size, int tx_buffer_size,
                              int queue_size, QueueHandle_t *uart_queue, int intr_alloc_flags);
esp_err_t uart_param_config(uart_port_t uart_num, const uart_config_t *uart_config);
esp_err_t uart_set_pin(uart_port_t uart_num, int tx_io_num, int rx_io_num,
                       int rts_io_num, int cts_io_num);   // pin ไม่ใช้ = UART_PIN_NO_CHANGE (-1)
esp_err_t uart_set_mode(uart_port_t uart_num, uart_mode_t mode); // UART_MODE_UART / RS485_HALF_DUPLEX ...
esp_err_t uart_wait_tx_done(uart_port_t uart_num, TickType_t ticks_to_wait);
esp_err_t uart_flush_input(uart_port_t uart_num);
esp_err_t uart_driver_delete(uart_port_t uart_num);

int uart_write_bytes(uart_port_t uart_num, const void *src, size_t size);   // คืนจำนวน byte ที่เขียน

// ⚠️ VERSION MARKER — ตรวจ driver/uart.h ของเวอร์ชันที่ใช้:
// v5.0–v5.4:  int uart_read_bytes(uart_port_t uart_num, void *buf, uint32_t length, TickType_t ticks_to_wait);
// ช่วงปลาย v5 ขึ้นไป: timeout เปลี่ยนเป็น int timeout_ms (หน่วยมิลลิวินาที ส่งค่าตรง ๆ ไม่ต้องแปลง)
```

## กับดัก Hallucination
1. ใส่ `pdMS_TO_TICKS()` ทั้งที่เวอร์ชันใหม่รับ ms ตรง ๆ (หรือกลับกัน) — เช็คก่อนทุกครั้ง
2. `uart_set_pin` ส่ง 0 ให้ RTS/CTS ที่ไม่ใช้ → ผิด ต้องใช้ `UART_PIN_NO_CHANGE`
3. `uart_write_bytes` คืน `int` (จำนวน byte) ไม่ใช่ `esp_err_t` — ห้าม ESP_ERROR_CHECK
4. ลืม `uart_driver_install` ก่อน `uart_read_bytes` → driver ยังไม่ถูกผูกกับ ring buffer
5. ใช้ UART0 สำหรับงานอื่นแล้วหา log หาย (log console ออก UART0)

## ตัวอย่างมินิมัล
```c
#include "driver/uart.h"
#define PORT UART_NUM_2

void app_main(void)
{
    const uart_config_t cfg = {
        .baud_rate = 115200,
        .data_bits = UART_DATA_8_BITS,
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
        .source_clk = UART_SCLK_DEFAULT,
    };
    ESP_ERROR_CHECK(uart_driver_install(PORT, 2048, 0, 0, NULL, 0));
    ESP_ERROR_CHECK(uart_param_config(PORT, &cfg));
    ESP_ERROR_CHECK(uart_set_pin(PORT, 17, 18, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE));

    uint8_t buf[64];
    int n = uart_read_bytes(PORT, buf, sizeof(buf), pdMS_TO_TICKS(100)); // v5.0–v5.4 (ticks)
    uart_write_bytes(PORT, "hello\r\n", 7);
}
```