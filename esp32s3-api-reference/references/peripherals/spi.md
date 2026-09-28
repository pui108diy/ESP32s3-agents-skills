# SPI Master — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "driver/spi_master.h"
```

## Hardware Facts (จาก TRM)
- SPI0/SPI1 = flash ภายใน **ห้ามใช้**; ใช้ `SPI2_HOST` (GPSPI2) หรือ `SPI3_HOST` (GPSPI3)
- ทั้งสองต่อ GPIO Matrix ได้ + มี GDMA
- **ไม่ใช้ DMA: transaction จำกัด 64 bytes** (SOC_SPI_MAXIMUM_BUFFER_SIZE)
- clock สูงสุด ~80 MHz บน GPSPI ของ S3

## Structs (ฟิลด์ที่ใช้จริง)
```c
typedef struct {
    int mosi_io_num;       // ชื่อใหม่ data0_io_num (union — ใช้ชื่อเดิมได้)
    int miso_io_num;
    int sclk_io_num;
    int quadwp_io_num;     // -1 ถ้าไม่ใช้
    int quadhd_io_num;     // -1
    int max_transfer_sz;   // ตั้งค่าให้ชัดเมื่อโอนใหญ่ ๆ ด้วย DMA
    uint32_t flags;        // SPICOMMON_BUSFLAG_QUAD ฯลฯ (0 สำหรับใช้งานปกติ)
    int intr_flags;
} spi_bus_config_t;

typedef struct {
    uint8_t  command_bits;       // 0 สำหรับ transaction ธรรมดา
    uint8_t  address_bits;       // 0
    uint8_t  dummy_bits;
    uint8_t  mode;               // SPI mode 0–3
    uint16_t cs_ena_pretrans;
    uint16_t cs_ena_posttrans;
    uint32_t clock_speed_hz;
    int      input_delay_ns;
    int      spics_io_num;       // ← CS ถูกคุมอัตโนมัติโดย driver
    int      queue_size;
    transaction_cb_t pre_cb;     // NULL ได้
    transaction_cb_t post_cb;    // NULL ได้
} spi_device_interface_config_t;

typedef struct {
    uint32_t flags;              // SPI_TRANS_* เช่น SPI_TRANS_USE_TXDATA (≤ 4 bytes)
    uint16_t cmd;
    uint64_t addr;
    size_t   length;             // ← หน่วย BIT! (rxlength = 0 → ใช้ค่า length)
    size_t   rxlength;
    void    *user;
    union { const void *tx_buffer; uint8_t tx_data[4]; };  // tx_buffer = NULL → ข้ามเฟสเขียน
    union { void *rx_buffer;       uint8_t rx_data[4]; };  // rx_buffer = NULL → ข้ามเฟสอ่าน
} spi_transaction_t;
```

## Function Signatures (exact)
```c
esp_err_t spi_bus_initialize(spi_host_device_t host_id, const spi_bus_config_t *bus_config,
                             spi_dma_chan_t dma_chan);      // dma_chan: SPI_DMA_DISABLED(0) หรือ SPI_DMA_CH_AUTO
esp_err_t spi_bus_add_device(spi_host_device_t host_id,
                             const spi_device_interface_config_t *dev_config, spi_device_handle_t *handle);
esp_err_t spi_device_queue_trans(spi_device_handle_t handle, spi_transaction_t *trans_desc,
                                 TickType_t ticks_to_wait);
esp_err_t spi_device_get_trans_result(spi_device_handle_t handle, spi_transaction_t **trans_desc,
                                      TickType_t ticks_to_wait);
esp_err_t spi_device_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc); // ← ไม่มีพารามิเตอร์ timeout!
esp_err_t spi_device_polling_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);
esp_err_t spi_bus_remove_device(spi_device_handle_t handle);
esp_err_t spi_bus_free(spi_host_device_t host_id);
```

## กับดัก Hallucination
1. `.length` เป็น **bit** ไม่ใช่ byte → `sizeof(buf) * 8`
2. เติมพารามิเตอร์ที่ 3 ให้ `spi_device_transmit()` → ไม่มี
3. คุม CS เองด้วย gpio_set_level → ไม่จำเป็น ใช้ `spics_io_num` ใน dev_config
4. `SPI_TRANS_USE_TXDATA` ใช้ได้เฉพาะข้อมูล ≤ 4 bytes (`tx_data[4]`)
5. DMA + buffer เป็น `const` string ที่ถูกวางใน flash rodata → พัง (ต้อง copy ไป DRAM ก่อน)
6. ไม่ใช้ DMA แต่ส่ง > 64 bytes → fail

## ตัวอย่างมินิมัล
```c
#include "driver/spi_master.h"
spi_device_handle_t dev;

void app_main(void)
{
    spi_bus_config_t bus = {
        .mosi_io_num = 11, .miso_io_num = 13, .sclk_io_num = 12,
        .quadwp_io_num = -1, .quadhd_io_num = -1,
    };
    ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &bus, SPI_DMA_CH_AUTO));

    spi_device_interface_config_t devcfg = {
        .clock_speed_hz = 10 * 1000 * 1000,
        .mode = 0,
        .spics_io_num = 10,
        .queue_size = 4,
    };
    ESP_ERROR_CHECK(spi_bus_add_device(SPI2_HOST, &devcfg, &dev));

    uint8_t tx[2] = {0xAB, 0xCD};
    spi_transaction_t t = { .length = sizeof(tx) * 8, .tx_buffer = tx };  // bit!
    ESP_ERROR_CHECK(spi_device_transmit(dev, &t));                        // blocking
}
```