# I2S (Inter-IC Sound) — New Channel Driver — ESP-IDF v5.x/v6.x (ESP32-S3)

## Includes
```c
#include "driver/i2s_std.h"    // Standard (Philips/PCM/MSB) — ใช้บ่อยสุด
#include "driver/i2s_pdm.h"    // PDM TX/RX (mic PDM)
#include "driver/i2s_tdm.h"    // TDM multi-channel
```

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 28)
- **2 controllers: I2S0, I2S1** — แต่ละตัวมี TX unit และ RX unit แยกอิสระ (full-duplex/half-duplex ได้)
- โหมด: **Standard / TDM (Philips, MSB, PCM) / PDM** — **PDM (TX และ PCM↔PDM) รองรับเฉพาะ I2S0**
- Sample rate รองรับ: 8/16/32/44.1/48/88.2/96/128/192 kHz (192 kHz ไม่รองรับใน 32-bit slave mode)
- ข้อมูล 8/16/24/32 บิต + **DMA** ขนย้ายข้อมูลอัตโนมัติ

## State Machine (สำคัญ — ผิดลำดับ = error)
```
i2s_new_channel → [REGISTERED] → i2s_channel_init_std_mode → [READY] → i2s_channel_enable → [RUNNING]
      ↑                                                                                      │
      └────────── reconfig_* / del ต้องทำหลัง i2s_channel_disable ◄──────────────────────────┘
```

## Structs (ฟิลด์ที่ใช้จริง)
```c
// --- ขั้นที่ 1: จอง channel ---
typedef struct {
    i2s_port_t id;              // I2S_NUM_0 / I2S_NUM_1
    i2s_role_t role;            // I2S_ROLE_MASTER / I2S_ROLE_SLAVE
    size_t dma_desc_num;        // จำนวน DMA descriptor (default ~6)
    size_t dma_frame_num;       // frame ต่อ descriptor (default ~240)
    bool auto_clear;            // TX: ส่ง 0 อัตโนมัติเมื่อ underflow (กันเสียงหึ่ง)
    int intr_priority;
    struct { /* io_loop_back, lcd_cam_en ... */ } flags;
} i2s_chan_config_t;
#define I2S_CHANNEL_DEFAULT_CONFIG(id, role) ...

// --- ขั้นที่ 2: init โหมด std ---
typedef struct {
    i2s_std_clk_config_t clk_cfg;
    i2s_std_slot_config_t slot_cfg;
    i2s_std_gpio_config_t gpio_cfg;   // { .mclk, .bclk, .ws, .dout, .din } — ไม่ใช้ = I2S_GPIO_UNUSED (-1)
} i2s_std_config_t;

#define I2S_STD_CLK_DEFAULT_CONFIG(rate) ...                       // mclk_multiple อัตโนมัติ
#define I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(bits, mode) ...        // ใช้ตัวนี้บ่อยสุด
#define I2S_STD_MSB_SLOT_DEFAULT_CONFIG(bits, mode) ...
#define I2S_STD_PCM_SLOT_DEFAULT_CONFIG(bits, mode) ...
// bits: I2S_DATA_BIT_WIDTH_16BIT ... | mode: I2S_SLOT_MODE_STEREO / MONO

// clk: { .mclk_hz, .sample_rate_hz, .mclk_multiple = I2S_MCLK_MULTIPLE_256, .clk_src }
```

## Function Signatures (exact)
```c
// --- Channel lifecycle ---
esp_err_t i2s_new_channel(const i2s_chan_config_t *chan_cfg,
                          i2s_chan_handle_t *tx_handle, i2s_chan_handle_t *rx_handle);
// ฝั่งไหนไม่ใช้ ส่ง NULL เข้าไป; ส่งทั้งคู่บน port เดียว = เตรียม full-duplex
esp_err_t i2s_del_channel(i2s_chan_handle_t handle);

// --- Init ตามโหมด (state ต้องเป็น REGISTERED) ---
esp_err_t i2s_channel_init_std_mode(i2s_chan_handle_t handle, const i2s_std_config_t *std_cfg);
esp_err_t i2s_channel_init_pdm_tx_mode(i2s_chan_handle_t handle, const i2s_pdm_tx_config_t *pdm_cfg);   // I2S0 only
esp_err_t i2s_channel_init_pdm_rx_mode(i2s_chan_handle_t handle, const i2s_pdm_rx_config_t *pdm_cfg);   // I2S0 only
esp_err_t i2s_channel_init_tdm_mode(i2s_chan_handle_t handle, const i2s_tdm_config_t *tdm_cfg);

// --- Enable/Disable ---
esp_err_t i2s_channel_enable(i2s_chan_handle_t handle);      // ← ต้องเรียกก่อน write/read!
esp_err_t i2s_channel_disable(i2s_chan_handle_t handle);     // ← ต้องเรียกก่อน reconfig/del

// --- ขนข้อมูล (หน่วย byte; timeout หน่วยมิลลิวินาที) ---
esp_err_t i2s_channel_write(i2s_chan_handle_t handle, const void *src, size_t size,
                            size_t *bytes_written, uint32_t timeout_ms);
esp_err_t i2s_channel_read(i2s_chan_handle_t handle, void *dest, size_t size,
                           size_t *bytes_read, uint32_t timeout_ms);
esp_err_t i2s_channel_preload_data(i2s_chan_handle_t handle, const void *src,
                                   size_t size, size_t *bytes_loaded);   // ใส่ข้อมูลก่อน enable (กัน click แรก)

// --- Reconfig (ต้อง disable ก่อน) ---
esp_err_t i2s_channel_reconfig_std_clock(i2s_chan_handle_t handle, const i2s_std_clk_config_t *clk_cfg);
esp_err_t i2s_channel_reconfig_std_slot(i2s_chan_handle_t handle, const i2s_std_slot_config_t *slot_cfg);
esp_err_t i2s_channel_reconfig_std_gpio(i2s_chan_handle_t handle, const i2s_std_gpio_config_t *gpio_cfg);

// --- Event callbacks (ISR context!) ---
esp_err_t i2s_channel_register_event_callback(i2s_chan_handle_t handle,
                                              const i2s_event_callbacks_t *callbacks, void *user_data);
// i2s_event_callbacks_t: on_recv, on_recv_q_ovf, on_sent, on_send_q_ovf
```

## Version Marker
- Legacy `driver/i2s.h` (`i2s_config_t`, `i2s_driver_install()`, `i2s_write()`, `i2s_set_pin()`) — deprecated v5.0, **ถูกลบใน v6** ⚠️ ตัวนี้เป็นแหล่ง hallucination ชั้นเซียนเพราะโค้ดเก่าในเน็ตเยอะมาก
- v5.3+/v6: code แยกเป็น `esp_driver_i2s` (header ชื่อเดิม)

## กับดัก Hallucination
1. เขียน `i2s_driver_install()` / `i2s_config_t` จากโค้ดเก่า → **v6 ลบแล้ว** → ใช้ new_channel + init_std_mode
2. `i2s_channel_write()` โดยไม่ `i2s_channel_enable()` ก่อน → error
3. `timeout_ms` เป็น **มิลลิวินาที** (`uint32_t`) ไม่ใช่ tick (`portMAX_DELAY` ใช้ได้เพราะค่า = 0xFFFFFFFF)
4. **PDM บน I2S_NUM_1** → ไม่ได้ — S3 มี PDM เฉพาะ I2S0 (TRM ch.28)
5. Full-duplex: ต้องจอง tx+rx คู่กันตั้งแต่ `i2s_new_channel` บน port เดียวกัน แล้ว init ทั้งคู่ด้วย config **เข้ากันได้** (role/clk เดียวกัน) จึงจะกลายเป็น full-duplex อัตโนมัติ
6. เปลี่ยน sample rate/slot ระหว่างช่องกำลังรัน → ต้อง `i2s_channel_disable()` ก่อนแล้วเรียก `reconfig_*`
7. `size` ใน write/read นับเป็น **byte ทั้งก้อน** (ไม่ใช่จำนวน sample) — 1 frame stereo 16-bit = 4 bytes
8. DMA buffer = `dma_desc_num × dma_frame_num × bytes/frame` — เล็กเกิน = underflow/overflow บ่อย; ใหญ่เกิน = latency สูง
9. GPIO ที่ไม่ใช้ (เช่น RX-only ไม่มี dout) ต้องตั้ง `I2S_GPIO_UNUSED` ไม่ใช่ -1 ตรง ๆ หรือ 0
10. ต้องการเสียงไม่มี click ตอนเริ่ม → ใช้ `i2s_channel_preload_data()` + `auto_clear = true`
11. Callback (`on_recv` ฯลฯ) รันใน ISR — ห้ามโลจิกซับซ้อน/printf/float

## ตัวอย่างมินิมัล (TX Standard Philips 48 kHz stereo 16-bit)
```c
#include "driver/i2s_std.h"

void app_main(void)
{
    i2s_chan_handle_t tx = NULL;
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
    ESP_ERROR_CHECK(i2s_new_channel(&chan_cfg, &tx, NULL));       // RX ไม่ใช้ = NULL

    i2s_std_config_t std_cfg = {
        .clk_cfg  = I2S_STD_CLK_DEFAULT_CONFIG(48000),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_STEREO),
        .gpio_cfg = {
            .mclk = I2S_GPIO_UNUSED,   // codec ภายนอกบางตัวต้องการ MCLK → ใส่ pin
            .bclk = 12,
            .ws   = 13,
            .dout = 14,
            .din  = I2S_GPIO_UNUSED,
        },
    };
    ESP_ERROR_CHECK(i2s_channel_init_std_mode(tx, &std_cfg));
    ESP_ERROR_CHECK(i2s_channel_enable(tx));

    int16_t frame[2] = { 1000, -1000 };               // L, R
    size_t written = 0;
    ESP_ERROR_CHECK(i2s_channel_write(tx, frame, sizeof(frame), &written, 1000)); // ms
}
```