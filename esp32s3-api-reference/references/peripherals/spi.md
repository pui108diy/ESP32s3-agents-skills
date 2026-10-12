# SPI Master Driver — v2.1
> Target: ESP32-S3 | ESP-IDF v5.2 – v6.1
> อ้างอิง: TRM v1.8 ch.30 (SPI) + ch.3 (GDMA) | Datasheet v2.2 | spi_common.h v6.1 (header จริง) | soc_caps.h v6.1 | Migration Guide v6.1 | docs.espressif.com SPI Master (S3)
> เปลี่ยนแปลง v2.1: แปลงโครงสร้างตาม skill v2.7 (§0 TL;DR + §10 จุดตรวจซ้ำ + ผูกรหัส Hot-List) · **กู้ §5 Structs + §8 ตัวอย่างกลับมาเต็ม** (หายไปจากไฟล์ v2.0 ที่บันทึกไว้)

## 0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. ใช้ `SPI2_HOST`/`SPI3_HOST` — SPI0/1 สงวนให้ flash/PSRAM
2. `.length`/`.rxlength` นับเป็น **bit** — ส่ง 4 bytes = `.length = 32`
3. `spi_device_transmit()` มี **2 พารามิเตอร์** — ไม่มี timeout
4. ไม่ใช้ DMA: จำกัด **64 bytes ต่อ transaction** — ยัดเกิน = HW wrap อ่าน W0 ซ้ำ (เสียเงียบ ๆ)
5. Clock: **IOMUX 80 MHz / GPIO matrix half-duplex ≤40 / full-duplex ~26 MHz**
6. DMA buffer: `heap_caps_malloc(MALLOC_CAP_DMA)` + **32-bit aligned + ความยาว %4 == 0**; rodata ใช้ไม่ได้
7. default `max_transfer_sz` = **4092 B (DMA) / 64 B (no DMA)**
8. `SPI3_HOST` **ไม่รองรับ Octal/OPI** — ต้องใช้ `SPI2_HOST`
9. GPIO26–32 ห้ามใช้เสมอ; 33–37 ห้ามเมื่อใช้ Octal flash/PSRAM
10. CMakeLists v5.3+: `REQUIRES esp_driver_spi`; v6.x มี `spi_bus_dma_memory_alloc()` จัด alignment ให้

## 1. Include & CMakeLists

```c
#include "driver/spi_master.h"   // path ชื่อเดิมทุกเวอร์ชัน v5.2 – v6.1
```

```cmake
# v5.2:        REQUIRES driver
# v5.3+ / v6.x: driver ถูกแยกเป็น component ย่อย — ระบุตรง:
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_driver_spi)
```
⚠️ v6.x: legacy `driver` component deprecated และ "no longer contains public dependencies" บน `esp_driver_spi` (Migration Guide v6.1)

## 2. Hardware Facts
**Hosts**
- `SPI1_HOST` ห้ามใช้ — SPI0/SPI1 สงวนให้ flash/PSRAM ภายใน
- `SPI2_HOST` (GP-SPI2): Single/Dual/Quad/**Octal**/QPI/OPI | full+half-duplex | DMA-controlled configurable segmented transfer (master + slave)
- `SPI3_HOST` (GP-SPI3): Single/Dual/Quad/QPI เท่านั้น — **ไม่รองรับ Octal/OPI (8-line)** และไม่รองรับ DMA-controlled configurable segmented transfer ฝั่ง slave (Datasheet v2.2 §4.2.1.5, TRM ch.30)
- ทั้งสอง host: GPIO Matrix routing + DMA channel ผ่าน GDMA controller

**Clock — แยกตามเส้นทาง pin**
| เส้นทาง | ความเร็วสูงสุด |
|---|---|
| IOMUX (dedicated SPI pins) | 80 MHz |
| GPIO Matrix — half-duplex | ≤ 40 MHz |
| GPIO Matrix — **full-duplex** | ~26 MHz (ideal slave) |
| Slave input clock | ≤ 60 MHz (fAPB_CLK ≥ 60 MHz) |

- ดัน full-duplex ผ่าน GPIO matrix ให้สูงกว่า 26 MHz: กำหนด `input_delay_ns` — driver ชดเชยด้วย dummy bits + sample time ให้อัตโนมัติ

**Pins ห้ามใช้ (Datasheet v2.2)**
- **GPIO26–32**: SPI0/1 (flash/PSRAM) — ห้ามใช้เด็ดขาด
- **GPIO33–37**: DQ4–DQ7 + DQS/DM ใน 8-line mode → โมดูล **Octal PSRAM (N8R8/N16R8) ใช้ GPIO33–37 ไม่ได้**

## 3. Transaction Limits — per-transaction ไม่ใช่ per-bus
**ไม่ใช้ DMA (CPU-controlled)**
- จำกัด **64 bytes ต่อ 1 transaction** (`SOC_SPI_MAXIMUM_BUFFER_SIZE` = 64)
- ที่มา: HW buffer `SPI_W0_REG~W15_REG` = 16 × 32-bit = 64 B (TRM ch.30)
- ⚠️ ยัดเกิน 64 B ใน transaction เดียว: hardware **wrap กลับไปอ่าน W0 ซ้ำ** (byte64 = `SPI_W0_REG[7:0]` อีกครั้ง) → ข้อมูลเสียแบบเงียบ ไม่มี error
- แก้: แบ่งหลาย transaction หรือเปิด DMA

**ใช้ DMA (GDMA)**
- DMA เปิด → default `max_transfer_sz` = **4092 bytes**; DMA ปิด → default = **64 bytes**
- `spi_bus_initialize()` รับ `dma_chan` = `SPI_DMA_DISABLED` (0) หรือ `SPI_DMA_CH_AUTO` (3) เท่านั้น

## 4. DMA Buffer Rules
- DMA-capable internal memory: `heap_caps_malloc(size, MALLOC_CAP_DMA)`
- **32-bit aligned + ความยาวเป็นจำนวนเท่าของ 4 bytes** — ไม่ผ่าน = driver สร้าง temp buffer + memcpy เอง → ช้าลง
- buffer `const` (flash rodata) ใช้กับ DMA ไม่ได้ — copy ไป DRAM ก่อน
- บน PSRAM: `MALLOC_CAP_SPIRAM | MALLOC_CAP_DMA` (cache-safe เท่านั้น)
- **v6.x helper**: `void *spi_bus_dma_memory_alloc(spi_host_device_t host_id, size_t size, uint32_t extra_heap_caps)` — เรียก**หลัง** `spi_bus_initialize()` จัด cache + alignment ให้เอง
- Nuance (TRM ch.3): ตัว hardware ไม่บังคับ word-align ใน internal RAM แบบ non-burst — ข้อบังคับ align อยู่ระดับ driver; **external RAM บังคับ burst mode + block align (16/32/64 B)**

## 5. Structs (จาก spi_common.h v6.1 จริง)
```c
typedef struct {
    union {
        struct {
            union { int mosi_io_num; int data0_io_num; };
            union { int miso_io_num; int data1_io_num; };
            int sclk_io_num;
            union { int quadwp_io_num; int data2_io_num; };
            union { int quadhd_io_num; int data3_io_num; };
            int data4_io_num;    // ⚠️ field ใหม่ — เช็ค header ของเวอร์ชันที่ใช้ (§10)
            int data5_io_num;
            int data6_io_num;
            int data7_io_num;
        };
        int iocfg[9];
    };
    bool data_io_default_level;   // ⚠️ ใหม่ใน v6.1
    int max_transfer_sz;
    uint32_t dma_burst_size;      // ⚠️ ใหม่ใน v6.1
    uint32_t flags;
    esp_intr_cpu_affinity_t isr_cpu_id;  // ⚠️ ใหม่ใน v6.1
    int intr_flags;
} spi_bus_config_t;

typedef struct {
    uint8_t command_bits;
    uint8_t address_bits;
    uint8_t dummy_bits;
    uint8_t mode;              // SPI mode 0–3
    uint16_t cs_ena_pretrans;
    uint8_t cs_ena_posttrans;
    int clock_speed_hz;
    int input_delay_ns;        // ชดเชย timing บน GPIO Matrix full-duplex
    int spics_io_num;          // CS อัตโนมัติ — ห้าม gpio_set_level เอง
    uint32_t flags;            // SPI_DEVICE_* flags
    int queue_size;
    transaction_cb_t pre_cb;
    transaction_cb_t post_cb;
} spi_device_interface_config_t;

typedef struct {
    uint32_t flags;
    uint16_t cmd;
    uint64_t addr;
    size_t length;             // ⚠️ หน่วย BIT
    size_t rxlength;           // BIT; ถ้า = 0 ใช้ค่า length
    void *user;
    union {
        const void *tx_buffer;
        const uint8_t tx_data[4];   // ใช้เมื่อ SPI_TRANS_USE_TXDATA (≤ 4 B)
    };
    union {
        void *rx_buffer;
        uint8_t rx_data[4];         // ใช้เมื่อ SPI_TRANS_USE_RXDATA
    };
} spi_transaction_t;
```

## 6. Function Signatures (คงเดิม v5.2 – v6.1)
```c
esp_err_t spi_bus_initialize(spi_host_device_t host_id, const spi_bus_config_t *bus_config, spi_dma_chan_t dma_chan);
esp_err_t spi_bus_add_device(spi_host_device_t host_id, const spi_device_interface_config_t *dev_config, spi_device_handle_t *handle);
esp_err_t spi_device_queue_trans(spi_device_handle_t handle, spi_transaction_t *trans_desc, TickType_t ticks_to_wait);
esp_err_t spi_device_get_trans_result(spi_device_handle_t handle, spi_transaction_t **trans_desc, TickType_t ticks_to_wait);
esp_err_t spi_device_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);   // ไม่มี timeout
esp_err_t spi_device_polling_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);
esp_err_t spi_bus_remove_device(spi_device_handle_t handle);
esp_err_t spi_bus_free(spi_host_device_t host_id);
```

## 7. กับดักที่พบบ่อย (ผูก HALLUCINATION-HOT-LIST v3.1)
1. `.length`/`.rxlength` เป็น **bit** — `sizeof(buf) * 8`; `rxlength = 0` → ใช้ค่า `length` → **SPI-02**
2. `spi_device_transmit()` 2 พารามิเตอร์ ไม่มี timeout → **SPI-01**
3. `SPI_TRANS_USE_TXDATA`/`RXDATA` ใช้ได้เฉพาะ ≤ 4 bytes ผ่าน `tx_data[4]`/`rx_data[4]`
4. CS คุมอัตโนมัติผ่าน `spics_io_num` — ห้าม `gpio_set_level` เอง
5. DMA + buffer `const`/rodata → fail; copy ลง DRAM ก่อน → **SPI-07**
6. 64 B = per-transaction + wrap รอบ W0 (เงียบ) → **SPI-05**
7. ความเร็วสูงสุดขึ้นกับ IOMUX vs GPIO Matrix vs duplex → **SPI-06**
8. `SPI3_HOST` ไม่รองรับ Octal/OPI → **SPI-10**
9. default `max_transfer_sz` = 4092/64 — ตั้งเฉพาะเมื่อต้องการค่าอื่น → **SPI-09**
10. v6.x มี `spi_bus_dma_memory_alloc()` — เรียกหลัง `spi_bus_initialize()` → **SPI-08**
11. CMakeLists v5.3+ → `esp_driver_spi` → **SPI-11**
12. `CONFIG_SPI_MASTER_IN_IRAM` บน v6.x = invisible + ต้อง `CONFIG_FREERTOS_IN_IRAM` → **SPI-12**

## 8. ตัวอย่างมินิมัล
```c
#include "driver/spi_master.h"

#define PIN_MOSI 11
#define PIN_MISO 13
#define PIN_SCLK 12
#define PIN_CS   10

spi_device_handle_t spi;

void app_main(void)
{
    spi_bus_config_t buscfg = {
        .mosi_io_num = PIN_MOSI,
        .miso_io_num = PIN_MISO,
        .sclk_io_num = PIN_SCLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        // .max_transfer_sz = 4092,  // default เมื่อเปิด DMA อยู่แล้ว
    };
    spi_device_interface_config_t devcfg = {
        .clock_speed_hz = 10 * 1000 * 1000,  // 10 MHz — ปลอดภัยบน GPIO Matrix ทุกโหมด
        .mode = 0,
        .spics_io_num = PIN_CS,              // CS อัตโนมัติ
        .queue_size = 7,
    };

    ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO));
    ESP_ERROR_CHECK(spi_bus_add_device(SPI2_HOST, &devcfg, &spi));  // 3 พารามิเตอร์

    uint8_t tx[16] = { /* ... */ };
    spi_transaction_t t = {
        .tx_buffer = tx,
        .length    = sizeof(tx) * 8,   // BIT: 16 B = 128
        .rxlength  = 0,                // 0 = ใช้ length
    };
    ESP_ERROR_CHECK(spi_device_transmit(spi, &t));  // 2 พารามิเตอร์

    spi_bus_remove_device(spi);
    spi_bus_free(SPI2_HOST);
}
```

## 9. v6.x Notes
- `CONFIG_SPI_MASTER_IN_IRAM`: invisible by default + **depends on `CONFIG_FREERTOS_IN_IRAM`** — กัน crash เมื่อ SPI-in-IRAM เรียก FreeRTOS ที่ไม่อยู่ใน IRAM (แลกมาด้วย IRAM usage สูงขึ้น)
- `esp_async_memcpy.h` / `esp_dma_utils.h` ย้ายไป component **`esp_driver_dma`** — ต้อง `REQUIRES esp_driver_dma` ถ้าใช้

## 10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
- `spi_bus_config_t` field ใหม่ (`data4–7_io_num`, `data_io_default_level`, `dma_burst_size`, `isr_cpu_id`) — ยืนยันใน v6.1; บน v5.2–v5.5 เปิด spi_common.h ของเวอร์ชันนั้นก่อนใช้
- `spi_bus_dma_memory_alloc()` — ยืนยันใน v6.1; มีตั้งแต่เวอร์ชันไหนยังไม่เช็ค (สงสัย v5.4+) — ตรวจก่อนใช้บน v5.x
- `CONFIG_SPI_MASTER_IN_IRAM` พฤติกรรมบน v5.x (ก่อนกลายเป็น invisible ใน v6)
- ความเร็วจริงที่ 80 MHz ผ่าน IOMUX — ขึ้นกับบอร์ด/สาย/อุปกรณ์ปลายทาง — ทดสอบจริงเสมอ