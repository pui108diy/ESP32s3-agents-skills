# I2C — New Master Driver (ESP-IDF v5.2+) — ESP32-S3

## ⚠️ กับดักใหญ่สุดของทั้ง skill นี้
มี **2 driver คนละยุค** ห้ามปนกันเด็ดขาด:

|         | Legacy                    | New (ใช้อันนี้)                |
| ------- | ------------------------- | ------------------------------ |
| Header  | `driver/i2c.h`            | `driver/i2c_master.h`          |
| โมเดล   | อ้าง port number          | **bus handle + device handle** |
| Timeout | `TickType_t` (ticks)      | **`int xfer_timeout_ms` (ms)** |
| สถานะ   | deprecated ใน v5          | มาตั้งแต่ v5.2                 |
| v6      | **End-of-Life (ลบใน v7)** | มาตรฐานปัจจุบัน                |

## Include
```c
#include "driver/i2c_master.h"
```

## Hardware Facts (จาก TRM)
- 2 controllers: `I2C_NUM_0`, `I2C_NUM_1`; SDA/SCL ผ่าน GPIO Matrix
- ต้องมี pull-up ที่สาย (ใช้ internal ได้กรณีระยะสั้น/ช้า; production แนะนำ external 4.7kΩ)

## Structs
```c
i2c_master_bus_config_t bus_cfg = {
    .i2c_port = 0,                        // v5.5+ ใช้ -1 เพื่อ auto-select ⚠️
    .sda_io_num = 8,
    .scl_io_num = 9,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .glitch_ignore_cnt = 7,               // ค่าที่นิยม
    .flags.enable_internal_pullup = true, // ⚠️ v5.2–v5.3 เป็น field ตรง: .enable_internal_pullup
};

i2c_device_config_t dev_cfg = {
    .dev_addr_length = I2C_ADDR_BIT_LEN_7,
    .device_address  = 0x68,
    .scl_speed_hz    = 400000,
};
```

## Function Signatures (exact — ยืนยันจาก header จริงใน esp-idf)
```c
esp_err_t i2c_new_master_bus(const i2c_master_bus_config_t *bus_config, i2c_master_bus_handle_t *ret_bus_handle);
esp_err_t i2c_del_master_bus(i2c_master_bus_handle_t bus_handle);
esp_err_t i2c_master_bus_add_device(const i2c_master_bus_handle_t bus_handle,
                                    const i2c_device_config_t *dev_config, i2c_master_dev_handle_t *ret_handle);
esp_err_t i2c_master_bus_rm_device(const i2c_master_dev_handle_t handle);
esp_err_t i2c_master_probe(const i2c_master_bus_handle_t bus_handle, uint16_t address, int xfer_timeout_ms);

esp_err_t i2c_master_transmit(const i2c_master_dev_handle_t i2c_dev,
                              const uint8_t *write_buffer, size_t write_size, int xfer_timeout_ms);
esp_err_t i2c_master_receive(const i2c_master_dev_handle_t i2c_dev,
                              uint8_t *read_buffer, size_t read_size, int xfer_timeout_ms);
esp_err_t i2c_master_transmit_receive(const i2c_master_dev_handle_t i2c_dev,
                              const uint8_t *write_buffer, size_t write_size,
                              uint8_t *read_buffer, size_t read_size, int xfer_timeout_ms);
esp_err_t i2c_master_register_event_callbacks(const i2c_master_dev_handle_t i2c_dev,
                              const i2c_master_event_callbacks_t *cbs, void *user_data); // async: ต้องตั้ง trans_queue_depth > 0
// v6.1 เพิ่ม: i2c_master_multi_buffer_transmit(...)
```

## Legacy (เจอในโค้ดเก่า — อ่านให้ออก แต่ห้ามเขียนใหม่)
```c
#include "driver/i2c.h"   // v4.x style
i2c_driver_install(I2C_NUM_0, I2C_MODE_MASTER, 0, 0, 0);
i2c_param_config(I2C_NUM_0, &i2c_config_t{...});
i2c_master_write_to_device(I2C_NUM_0, addr, buf, len, ticks);   // ← AI มัก hallucinate ตัวนี้บน v5.2+
```

## กับดัก Hallucination
1. ผสม legacy กับ new (เช่น `i2c_param_config` + `i2c_master_transmit`) → ไม่มีทาง compile ผ่าน
2. ส่ง ticks เข้า `xfer_timeout_ms` → timeout ผิด 100 เท่า
3. ใช้ address 8-bit (0xD0) แทน 7-bit (0x68) — new driver ใช้ 7-bit ตรง ๆ
4. ลืมว่า 1 bus ต่อหลาย device ได้ (add_device หลายครั้ง) ไม่ต้อง new bus ใหม่ต่อ sensor

## ตัวอย่างมินิมัล (อ่าน WHO_AM_I จาก IMU 0x68)
```c
#include "driver/i2c_master.h"

void app_main(void)
{
    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t imu;

    i2c_master_bus_config_t bus_cfg = {
        .i2c_port = 0, .sda_io_num = 8, .scl_io_num = 9,
        .clk_source = I2C_CLK_SRC_DEFAULT, .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,
    };
    ESP_ERROR_CHECK(i2c_new_master_bus(&bus_cfg, &bus));

    i2c_device_config_t dev_cfg = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address = 0x68, .scl_speed_hz = 400000,
    };
    ESP_ERROR_CHECK(i2c_master_bus_add_device(bus, &dev_cfg, &imu));

    uint8_t reg = 0x75, id = 0;
    ESP_ERROR_CHECK(i2c_master_transmit_receive(imu, &reg, 1, &id, 1, 100)); // 100 ms
}
```