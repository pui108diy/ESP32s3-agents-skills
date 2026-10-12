# I2C — v2.1
> Target: ESP32-S3 | ESP-IDF v5.2 – v6.1
> อ้างอิง: TRM v1.8 ch.27 (I2C) | Datasheet v2.2 §4.2.1.2 | driver/i2c_master.h + i2c_slave.h (ตรวจจริง v5.4.1/v5.5.1/v6.1) | soc_caps.h v6.1
> เปลี่ยนแปลง v2.1: แปลงโครงสร้างตาม skill v2.7 (§0 TL;DR + §10 จุดตรวจซ้ำ + ผูกรหัส Hot-List) · เพิ่ม CMakeLists esp_driver_i2c · เพิ่ม §9 v6.x Notes

## 0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. ใช้ **new driver** (`driver/i2c_master.h`) ตั้งแต่ v5.2 — legacy `driver/i2c.h` EOL ใน v6.0 ลบใน v7.0
2. **2 controllers**; ความเร็วสูงสุด **~800 kbit/s** (จำกัดโดย pull-up) — ไม่มี fast-plus 1 MHz บน S3
3. Internal pull-up อยู่ใน **`.flags.enable_internal_pullup`** — รูปแบบเดียวทุกเวอร์ชัน v5.2–v6.1
4. 7-bit address ใส่ตรง ๆ (0x68) — **ห้าม shift** เป็น 0xD0
5. Transfer ทุกตัว sync ใช้ `xfer_timeout_ms` หน่วย **ms**
6. probe: `ESP_ERR_NOT_FOUND` = ไม่มี ACK · `ESP_ERR_TIMEOUT` = บัสผิดปกติ (มัก pull-up) · ห้ามส่ง -1
7. Device หลายตัว = `i2c_master_bus_add_device()` หลายครั้งบน bus handle เดิม
8. Slave driver **redesign ใน v5.4** — รับผ่าน callbacks + ส่งด้วย `i2c_slave_write()`
9. v6.0+: NACK → **`ESP_ERR_INVALID_RESPONSE`** (ไม่ใช่ `ESP_ERR_INVALID_STATE`)
10. Async (callback) ต้องตั้ง `trans_queue_depth > 0` ตอนสร้าง bus

## 1. Include & CMakeLists

```c
#include "driver/i2c_master.h"   // new master driver (v5.2+)
#include "driver/i2c_slave.h"    // new slave driver (redesign v5.4) — เฉพาะงาน slave
// legacy "driver/i2c.h" — EOL ตั้งแต่ v6.0 ลบใน v7.0 (ไว้อ่านโค้ดเก่าเท่านั้น)
```

```cmake
# v5.2:        REQUIRES driver
# v5.3+ / v6.x: แยก component — ระบุตรง:
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_driver_i2c)
```

## 2. Hardware Facts (TRM ch.27 + Datasheet v2.2 + soc_caps v6.1)
- **2 controllers**: `I2C_NUM_0`, `I2C_NUM_1` (`SOC_I2C_NUM = 2`) — SDA/SCL ผ่าน GPIO Matrix เลือกขาได้เกือบทุกตัว
- ความเร็ว: Standard **100 kbit/s** / Fast **400 kbit/s** / สูงสุด **~800 kbit/s** (จำกัดโดยกำลัง pull-up)
- Addressing: 7-bit และ 10-bit (`SOC_I2C_SUPPORT_10BIT_ADDR`) + double addressing (slave + register address)
- รองรับ multi-master, clock stretching (slave อ่าน stretch cause ได้), slave broadcast
- **TX RAM / RX RAM: 32×8 bit อย่างละด้าน** ต่อ controller — โหมด FIFO wrap ทำให้ส่ง/รับเกิน 32 ไบต์ได้ (driver จัดการ — **ไม่ใช่ลิมิตขนาด transaction**)
- **Clock source ของ I2C_SCLK: XTAL_CLK หรือ RC_FAST_CLK เท่านั้น** (register/RAM อยู่โดเมน APB แต่บัสไม่กิน clock จาก APB)
- Programmable noise filter ทั้ง SCL/SDA (register คู่กับ `glitch_ignore_cnt` ของ driver)
- Master ส่ง SCL pulses ตอน idle เพื่อปลดล็อก slave ที่ค้าง (SCL_RST_SLV)
- Internal pull-up อ่อน (สายสั้น/ทดลอง) — production ใช้ external **4.7 kΩ**

## 3. Structs (ยืนยัน header จริง v5.4.1/v5.5.1/v6.1 — โครงเหมือนกันทั้งสาม)
```c
typedef struct {
    i2c_port_num_t i2c_port;          // -1 = auto select (ยืนยัน v5.4.1+; ก่อนหน้านั้นยังไม่เช็ค)
    gpio_num_t sda_io_num;
    gpio_num_t scl_io_num;
    i2c_clock_source_t clk_source;    // I2C_CLK_SRC_DEFAULT (อยู่ใน anonymous union — S3 ใช้ตัวนี้)
    uint8_t glitch_ignore_cnt;        // กรอง glitch สั้นกว่าค่านี้ (I2C module clock cycle) — แนะนำ 7
    int intr_priority;                // 0 = driver เลือก default (1,2,3)
    size_t trans_queue_depth;         // เฉพาะโหมด async (callback) — แนะนำ max_device_num × per_transaction
    struct {
        uint32_t enable_internal_pullup: 1;  // ⚠️ อยู่ใน .flags ตั้งแต่ v5.2 — ไม่เคยย้ายไประดับบนสุด
        uint32_t allow_pd:               1;  // ให้ driver backup/restore registers รอบ sleep → ปิด power domain ได้ (แลกกับ RAM)
    } flags;
} i2c_master_bus_config_t;

typedef struct {
    i2c_addr_bit_len_t dev_addr_length;  // I2C_ADDR_BIT_LEN_7 / I2C_ADDR_BIT_LEN_10
    uint16_t device_address;             // 7-bit ใส่ตรง ๆ เช่น 0x68 (ห้าม shift เป็น 0xD0)
    uint32_t scl_speed_hz;               // เช่น 100000 / 400000
    uint32_t scl_wait_us;                // timeout ต่อ SCL cycle (µs)
} i2c_device_config_t;
```

## 4. Master API (timeout ทุกตัวหน่วย ms)
```c
// ===== Bus / device =====
esp_err_t i2c_new_master_bus(const i2c_master_bus_config_t *bus_config, i2c_master_bus_handle_t *ret_bus_handle);
//   คืน ESP_ERR_NOT_FOUND = ไม่มีพอร์ตว่างเหลือ
esp_err_t i2c_del_master_bus(i2c_master_bus_handle_t bus_handle);
esp_err_t i2c_master_bus_add_device(i2c_master_bus_handle_t bus_handle, const i2c_device_config_t *dev_config, i2c_master_dev_handle_t *ret_handle);
esp_err_t i2c_master_bus_rm_device(i2c_master_dev_handle_t handle);
esp_err_t i2c_master_device_change_address(i2c_master_dev_handle_t i2c_dev, uint16_t new_device_address, int timeout_ms);  // ยืนยันมีใน v6.1

// ===== Probe =====
esp_err_t i2c_master_probe(i2c_master_bus_handle_t bus_handle, uint16_t address, int xfer_timeout_ms);
//   -1 (รอตลอดไป) ไม่แนะนำกับฟังก์ชันนี้ — doc ระบุตรง ๆ
//   ESP_ERR_NOT_FOUND = ไม่มี device ACK | ESP_ERR_TIMEOUT = บัสผิดปกติ มัก pull-up ไม่ต่อ
//   @attention: ต้องมี pull-up (external หรือ .flags.enable_internal_pullup)

// ===== Transfer (sync) =====
esp_err_t i2c_master_transmit(i2c_master_dev_handle_t i2c_dev, const uint8_t *write_buffer, size_t write_size, int xfer_timeout_ms);
esp_err_t i2c_master_receive(i2c_master_dev_handle_t i2c_dev, uint8_t *read_buffer, size_t read_size, int xfer_timeout_ms);
esp_err_t i2c_master_transmit_receive(i2c_master_dev_handle_t i2c_dev, const uint8_t *write_buffer, size_t write_size, uint8_t *read_buffer, size_t read_size, int xfer_timeout_ms);
esp_err_t i2c_master_multi_buffer_transmit(i2c_master_dev_handle_t i2c_dev, i2c_master_transmit_multi_buffer_info_t *buffer_info_array, size_t array_size, int xfer_timeout_ms);
//   ↑ มีอย่างน้อยตั้งแต่ v5.4.1 (ไม่ใช่ API ใหม่ของ v6.1)
esp_err_t i2c_master_execute_defined_operations(i2c_master_dev_handle_t i2c_dev, i2c_operation_job_t *i2c_operation, size_t operation_list_num, int xfer_timeout_ms);
//   ↑ v5.5+ (ยืนยัน v5.5.1; v5.4.1 ยังไม่มี)
```

## 5. Async (Callback)
```c
esp_err_t i2c_master_register_event_callbacks(i2c_master_dev_handle_t i2c_dev, const i2c_master_event_callbacks_t *cbs, void *user_data);
//   ⚠️ ต้องตั้ง trans_queue_depth > 0 ตอน i2c_new_master_bus ก่อน
esp_err_t i2c_master_bus_wait_all_done(i2c_master_bus_handle_t bus_handle, int timeout_ms);
//   รอ transaction ค้างทั้งหมดเสร็จ; -1 = รอตลอดไป
```

## 6. Slave Driver (v5.4 redesign — slave เป็นฝ่ายถูกกระทำ ทำงานผ่าน callback ตามจังหวะ master)
```c
// ⚠️ signature ย่อจาก migration guide — ก่อนใช้จริงเปิด driver/i2c_slave.h ของเวอร์ชันที่ใช้ (ดู §10)
esp_err_t i2c_new_slave_bus(const i2c_slave_bus_config_t *slave_bus_config, i2c_slave_bus_handle_t *ret_bus_handle);
esp_err_t i2c_slave_register_event_callbacks(i2c_slave_bus_handle_t slave_bus, const i2c_slave_event_callbacks_t *cbs, void *user_data);
esp_err_t i2c_slave_write(i2c_slave_bus_handle_t slave_bus, const uint8_t *data, size_t size, int xfer_timeout_ms);
//   ↑ แทน i2c_slave_transmit เดิม
// ลบแล้วใน v5.4: i2c_slave_receive (รับผ่าน callbacks แทน), i2c_slave_write_ram, i2c_slave_read_ram
```

## 7. กับดักที่พบบ่อย (ผูก HALLUCINATION-HOT-LIST v3.1)
1. `.enable_internal_pullup` เป็นฟิลด์ระดับบนสุด / "ย้ายใน v5.4" → ผิดทุกเวอร์ชัน — อยู่ใน `.flags` ตั้งแต่ v5.2 → **I2C-03**
2. 7-bit address ต้อง shift เป็น 0xD0 → ผิด — ใส่ตรง ๆ 0x68
3. `i2c_master_write_to_device()` / `read_from_device()` → ของ legacy — v5.2+ ใช้ `transmit/receive/transmit_receive` → **I2C-01**
4. ผสม legacy + new driver ในโปรเจกต์เดียว → เสี่ยง conflict การยึดพอร์ต — ใช้ new อย่างเดียว
5. NACK เช็คด้วย `ESP_ERR_INVALID_STATE` บน v6.0+ → ผิด — `ESP_ERR_INVALID_RESPONSE` → **I2C-05**
6. probe ได้ TIMEOUT แปลว่า "device ไม่อยู่" → ผิด — NOT_FOUND = ไม่มี ACK; TIMEOUT = บัสผิดปกติ → **I2C-06**
7. probe ส่ง `-1` รอตลอดไป → ไม่แนะนำ (doc ระบุตรง ๆ) → **I2C-06**
8. `scl_speed_hz = 1000000` (fast-plus) → S3 สูงสุด ~800 kbit/s → **I2C-08**
9. `i2c_port = -1` ในโค้ดที่รองรับ v5.2–v5.3 → ยังไม่ยืนยันว่ามีก่อน v5.4 — ระบุพอร์ตตรง ๆ ปลอดภัยกว่า
10. TX/RX RAM 32 ไบต์ = ลิมิต transaction → ไม่ใช่ — FIFO wrap + driver จัดการ
11. `register_event_callbacks` โดยไม่ตั้ง `trans_queue_depth > 0` → async ใช้ไม่ได้
12. Device ที่สองต้องสร้าง bus ใหม่ → ไม่ต้อง — `add_device` หลายครั้งบน bus เดิม
13. Timeout ของ transfer เป็น ticks → ผิด — `xfer_timeout_ms` หน่วย ms → **I2C-02**
14. `multi_buffer_transmit` คิดว่าใหม่ใน v6.1 → มีตั้งแต่ v5.4.1 → **I2C-04**
15. เขียน slave แบบเก่า (`i2c_slave_receive` ฯลฯ) → ลบใน v5.4 → **I2C-07**

## 8. ตัวอย่างมินิมัล
```c
#include "driver/i2c_master.h"

void app_main(void)
{
    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t imu;

    i2c_master_bus_config_t bus_cfg = {
        .i2c_port = 0, .sda_io_num = 8, .scl_io_num = 9,
        .clk_source = I2C_CLK_SRC_DEFAULT, .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,   // รูปแบบเดียว ใช้ได้ทุกเวอร์ชัน v5.2–v6.1
    };
    ESP_ERROR_CHECK(i2c_new_master_bus(&bus_cfg, &bus));

    esp_err_t r = i2c_master_probe(bus, 0x68, 100);   // 100 ms — ห้าม -1
    // ESP_OK = มี device · ESP_ERR_NOT_FOUND = ไม่มี ACK · ESP_ERR_TIMEOUT = เช็ค pull-up/สาย

    i2c_device_config_t dev_cfg = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address = 0x68, .scl_speed_hz = 400000,
    };
    ESP_ERROR_CHECK(i2c_master_bus_add_device(bus, &dev_cfg, &imu));

    uint8_t reg = 0x75, id = 0;
    ESP_ERROR_CHECK(i2c_master_transmit_receive(imu, &reg, 1, &id, 1, 100)); // 100 ms
}
```

## 9. v6.x Notes
- Legacy `driver/i2c.h` = **EOL ตั้งแต่ v6.0** (ลบใน v7.0) — ไม่มี bug fix/security patch
- NACK เปลี่ยนเป็น `ESP_ERR_INVALID_RESPONSE` (transmit / multi_buffer_transmit / transmit_receive / execute_defined_operations)
- `esp_driver_i2c` เป็น component แยกตั้งแต่ v5.3

## 10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
- `driver/i2c_slave.h` — signatures ทั้งหมดสรุปจาก migration guide v6.x ยังไม่ผ่านการตรวจ header รายเวอร์ชัน (เปิดก่อนใช้จริง)
- `i2c_master_device_change_address` — ยืนยันมีใน v6.1 แล้ว ยังไม่เช็ค v5.4–v5.5
- field `scl_wait_us` ของ `i2c_device_config_t` — มีตั้งแต่เวอร์ชันไหน (เช็ค v5.2–v5.3)
- Kconfig `CONFIG_I2C_SKIP_LEGACY_CONFLICT_CHECK` — มีรายงานจาก reviewer ว่าใช้ข้าม conflict ของ legacy/new ได้ — ยังไม่ยืนยันกับ Kconfig จริง