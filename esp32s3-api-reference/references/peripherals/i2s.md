# I2S — v2.1
> Target: ESP32-S3 | ESP-IDF v5.2 – v6.1
> อ้างอิง: TRM v1.8 ch.28 | Datasheet v2.2 | i2s_std.h / i2s_pdm.h / i2s_tdm.h / i2s_types.h v6.1 (header จริง) | soc_caps.h v6.1 | Migration Guide v6.1 + v5.3 | docs.espressif.com I2S (S3)
> เปลี่ยนแปลง v2.1 (ตามโครงสร้าง skill v2.7.0): เพิ่ม §0 TL;DR · ผูก §7 กับดักกับรหัส Hot-List ใหม่ (I2S-01–11) · เพิ่ม §10 จุดตรวจซ้ำกับ header จริง — เนื้อหาเทคนิคคงจาก v2.0

## 0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. API = **channel-based**: `i2s_new_channel()` → `i2s_channel_init_std/pdm/tdm_mode()` → `i2s_channel_enable()` → read/write → `i2s_channel_disable()` → `i2s_del_channel()` (legacy `driver/i2s.h` ลบใน v6.0)
2. **PDM (TX และ RX) = I2S0 เท่านั้น**; Standard/TDM ใช้ได้ทั้ง I2S0/I2S1
3. ข้อมูล 24-bit ต้องตั้ง `mclk_multiple = I2S_MCLK_MULTIPLE_384`
4. `timeout_ms` เป็น**มิลลิวินาที** (portMAX_DELAY ใช้ได้ = UINT32_MAX); `size` เป็น bytes
5. Buffer DMA เดี่ยว ≤ **4092 bytes** (`dma_frame_num × slot_num × data_bit_width / 8 ≤ 4092`)
6. `i2s_channel_preload_data()` = state READY เท่านั้น + TX เท่านั้น + เช็ค `bytes_loaded`
7. Callback อ่านข้อมูลผ่าน **`dma_buf`** (field `data` deprecated ตั้งแต่ v5.3)
8. บน S3 ข้อมูลต่อ slot ใน DMA buffer เป็น **16-bit aligned** (slot 8-bit กิน 2 bytes)
9. S3 = I2S HW v2 → slot config ใช้ `left_align`/`big_endian`/`bit_order_lsb` — ไม่มี `msb_right`
10. CMakeLists: `REQUIRES esp_driver_i2s` (v5.3+); include แยก `driver/i2s_std.h` / `i2s_pdm.h` / `i2s_tdm.h`

## 1. Include & CMakeLists

```c
#include "driver/i2s_std.h"    // Standard mode (รวม i2s_channel.h อัตโนมัติ)
#include "driver/i2s_pdm.h"    // PDM TX/RX
#include "driver/i2s_tdm.h"    // TDM
```

```cmake
# v5.2:        REQUIRES driver
# v5.3+ / v6.x: driver ถูกแยกเป็น component ย่อย — ระบุตรง:
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_driver_i2s)
```
⚠️ Legacy `driver/i2s.h` deprecated ตั้งแต่ v5.0 และ**ถูกลบใน v6.0** — `i2s_driver_install()` / `i2s_config_t` / `i2s_set_adc_mode()` / `i2s_adc_enable()` / `i2s_adc_disable()` ใช้ไม่ได้ (Migration Guide v6.1)

## 2. Hardware Facts (TRM ch.28 + soc_caps v6.1)
- **2 controllers (I2S0, I2S1)** — TX/RX unit แยกอิสระ, full/half-duplex, FIFO 64×32-bit ต่อฝั่ง
- **ตารางโหมดต่อ controller:**

| โหมด | I2S0 | I2S1 |
|---|---|---|
| Standard (Philips/MSB/PCM) | ✓ | ✓ |
| TDM (สูงสุด 16 slots) | ✓ | ✓ |
| PDM TX | ✓ | ✗ |
| PDM RX | ✓ | ✗ |

- **PCM↔PDM converters เป็นของ I2S0 เท่านั้น** (TRM §28.4 + register "For I2S0 Only")
- **Sample rates:** 8/16/32/44.1/48/88.2/96/128/192 kHz — 192 kHz ไม่รองรับใน 32-bit slave mode (TRM)
- **PDM:** fixed 2 slots, ข้อมูล 16-bit เท่านั้น; RX raw mode 1.024–6.144 MHz / PCM mode 16–48 kHz; TX แนะนำไม่เกิน 48 kHz (glitch), `up_sample_fs` ≤ 480 (i2s_pdm.h v6.1)
- **S3 = I2S HW version 2** (soc_caps): slot config ใช้ `left_align`/`big_endian`/`bit_order_lsb` — **ไม่มี** `msb_right` (ของ ESP32 รุ่นแรก)
- **Multi-line PDM (v6.1):** `SOC_I2S_PDM_MAX_TX_LINES = 2` (TX 2-line DAC mode: `line_mode` + `dout2` — ทดแทน DAC ในตัว), `SOC_I2S_PDM_MAX_RX_LINES = 4` (RX หลายไมค์ผ่าน `dins[]`) — ⚠️ ฟีเจอร์ใหม่ ตรวจ header ของ IDF เวอร์ชันที่ใช้จริง
- DMA ผ่าน GDMA controller

## 3. Channel API & Lifecycle

```c
i2s_chan_handle_t tx;
i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
// default: dma_desc_num = 6, dma_frame_num = 240, auto_clear = true
i2s_new_channel(&chan_cfg, &tx, NULL);      // full-duplex: ส่ง &tx และ &rx พร้อมกันใน call เดียว
i2s_channel_init_std_mode(tx, &std_cfg);    // หรือ pdm_tx / pdm_rx / tdm mode
i2s_channel_enable(tx);                     // ต้องเรียกก่อน read/write
i2s_channel_write(tx, buf, size, &bytes_written, timeout_ms);
i2s_channel_disable(tx);                    // ต้องเรียกก่อน reconfig / delete
i2s_del_channel(tx);
```

- **State machine:** REGISTERED → (init) → READY → (enable) → RUNNING → (disable) → READY
- **`i2s_channel_preload_data(tx_handle, src, size, &bytes_loaded)`** — เขียนข้อมูลลง TX DMA buffer ล่วงหน้า (กัน underflow ตอน start): เรียกได้**เฉพาะ state READY** (init แล้ว ยังไม่ enable) และเฉพาะ TX channel; DMA เต็มจะ preload ไม่ได้ — เช็ค `bytes_loaded` เสมอ
- **Reconfig (ต้อง disable ก่อน):** `i2s_channel_reconfig_std_clock/slot/gpio` (มีเฉพาะ std; pdm/tdm มีชุดของตัวเอง)
- **`size` ใน read/write เป็น bytes**, `timeout_ms` เป็น**มิลลิวินาที** (ไม่ใช่ tick) — `portMAX_DELAY` ใช้ได้เพราะค่า = 0xFFFFFFFF = UINT32_MAX; เช็ค `bytes_written`/`bytes_read` จริงเสมอ (timeout อาจทำให้เขียนได้ไม่ครบ)

## 4. Structs (จาก header v6.1 จริง)

### i2s_std_config_t
```c
typedef struct {
    i2s_std_clk_config_t  clk_cfg;   // sample_rate_hz, clk_src, ext_clk_freq_hz(HWv2), mclk_multiple, bclk_div
    i2s_std_slot_config_t slot_cfg;  // data_bit_width, slot_bit_width, slot_mode, slot_mask, ws_width, ws_pol,
                                     // bit_shift, left_align, big_endian, bit_order_lsb (HWv2)
    i2s_std_gpio_config_t gpio_cfg;  // mclk, bclk, ws, dout, din + invert_flags{mclk_inv, bclk_inv, ws_inv}
} i2s_std_config_t;
```
- Macro: `I2S_STD_CLK_DEFAULT_CONFIG(rate)`, `I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(bits, mode)`, `I2S_STD_PCM_SLOT_DEFAULT_CONFIG`, `I2S_STD_MSB_SLOT_DEFAULT_CONFIG`
- ⚠️ **`I2S_STD_PHILIP_SLOT_DEFAULT_CONFIG` (ไม่มี S) เป็น alias เก่า** — ใช้รูป PHILIPS ให้ตรงกัน (TDM เช่นกัน)
- GPIO ที่ไม่ใช้ ตั้ง `I2S_GPIO_UNUSED` (-1)
- **24-bit ต้องตั้ง `mclk_multiple = I2S_MCLK_MULTIPLE_384`** (multiple of 3) ไม่อย่างนั้น sample rate ไม่แม่น (i2s_std.h v6.1)

### i2s_tdm_config_t
```c
typedef struct {
    i2s_tdm_clk_config_t  clk_cfg;
    i2s_tdm_slot_config_t slot_cfg;  // + slot_mask (16 ช่อง), skip_mask, total_slot
    i2s_tdm_gpio_config_t gpio_cfg;
} i2s_tdm_config_t;
```
- `total_slot`: จำนวน slot รวมของ frame — ตั้งเล็กกว่าช่องที่ enable มากสุดจะถูกยกให้เท่าช่องนั้นอัตโนมัติ
- สอง channel บน port เดียวกันที่ share BCLK/WS config → full-duplex อัตโนมัติ

### i2s_pdm_tx/rx_config_t
- RX: `data_fmt` = `I2S_PDM_DATA_FMT_RAW` (รับ raw PDM) หรือ `I2S_PDM_DATA_FMT_PCM` (ผ่าน PDM-to-PCM filter ของ I2S0), `dn_sample_mode`, gpio: `clk`, `din` (หรือ `dins[]` หลายไมค์บน v6.1)
- TX: `up_sample_fp`, `up_sample_fs` (≤ 480), `line_mode` (2-DAC mode) + gpio `dout`/`dout2`
- Macro: `I2S_PDM_RX_SLOT_PCM_FMT_DEFAULT_CONFIG` / `I2S_PDM_RX_SLOT_RAW_FMT_DEFAULT_CONFIG` / `I2S_PDM_TX_SLOT_DEFAULT_CONFIG`

## 5. DMA Buffer Sizing (สูตรทางการจาก docs)
- **ขนาด buffer DMA เดี่ยว ≤ 4092 bytes:** `dma_frame_num × slot_num × data_bit_width / 8 ≤ 4092` (เลข 4092 รากเดียวกับ default max_transfer_sz ของ SPI-DMA — ข้อจำกัดระดับ GDMA descriptor)
- เลือก `dma_frame_num` ใหญ่สุดเท่าที่ 4092 อนุญาต → ลดความถี่ interrupt
- **กันข้อมูลหลุดระหว่าง polling:** `interrupt_interval = dma_frame_num / sample_rate` แล้ว `dma_desc_num > polling_cycle / interrupt_interval` รวมถึง `recv_buffer_size > dma_desc_num × dma_buffer_size`
- ⚠️ **บน S3 ข้อมูลต่อ 1 slot ใน DMA buffer เป็น 16-bit aligned** — driver คำนวณ `((bit_width + 15) / 16) × 2` bytes ต่อ slot (slot 8-bit จริง ๆ กิน 2 bytes) → buffer จริงอาจใหญ่กว่าสูตรอย่างง่ายเมื่อใช้ slot แคบ

## 6. Callbacks
```c
i2s_event_callbacks_t cbs = {
    .on_recv       = ...,  // รับข้อมูลครบ DMA buffer
    .on_recv_q_ovf = ...,  // คิวรับล้น
    .on_sent       = ...,  // ส่งครบ
    .on_send_q_ovf = ...,  // คิวส่งล้น
};
i2s_channel_register_event_callback(handle, &cbs, user_data);
```
- `i2s_event_data_t` = `{ void *dma_buf; size_t size; }` — ⚠️ **field `data` (secondary pointer) deprecated ตั้งแต่ v5.3 → ใช้ `dma_buf`** (Migration Guide v5.3)
- Callback รันใน **ISR context** — ห้าม blocking/งานหนัก; ถ้าเปิด `CONFIG_I2S_ISR_IRAM_SAFE` callback ต้องอยู่ใน IRAM (`IRAM_ATTR`)

## 7. กับดักที่พบบ่อย (ผูกกับ HALLUCINATION-HOT-LIST v3.0)
1. `i2s_driver_install()`/`i2s_config_t`/`driver/i2s.h` → ลบใน v6.0 — ใช้ channel API + `esp_driver_i2s` → **I2S-01**
2. ไม่ `i2s_channel_enable()` ก่อน read/write และไม่ disable ก่อน reconfig/delete → error → **I2S-11**
3. `timeout_ms` เป็น ms ไม่ใช่ FreeRTOS tick; `size` เป็น bytes ไม่ใช่ samples → **I2S-03**
4. ใช้ PDM บน I2S1 → ไม่ได้ — PDM TX และ RX อยู่ I2S0 เท่านั้น → **I2S-02**
5. 24-bit ไม่ตั้ง `mclk_multiple = 384` → sample rate ไม่แม่น → **I2S-04**
6. `i2s_channel_preload_data()` เรียกตอน RUNNING → ผิด — ต้อง READY เท่านั้น + เช็ค bytes_loaded → **I2S-05**
7. DMA buffer เดี่ยวเกิน 4092 bytes → ตั้ง dma_frame_num ตามสูตร §5 → **I2S-06**
8. ใช้ macro PHILIP (ไม่มี S) → เป็น alias เก่า ใช้ PHILIPS → **I2S-07**
9. อ่าน event ผ่าน `data` field → deprecated v5.3 ใช้ `dma_buf` → **I2S-08**
10. PDM ใช้ slot/bit อื่น → fixed 2 slots + 16-bit เท่านั้น; TDM ≤ 16 slots; 192 kHz ไม่ได้ใน 32-bit slave → **I2S-09**
11. คำนวณ buffer โดยลืมว่า S3 เก็บ slot แบบ 16-bit aligned → **I2S-10**
12. ใช้ `msb_right` ใน slot config → ของ ESP32 รุ่นแรก — S3 (HW v2) ใช้ `left_align`/`big_endian`/`bit_order_lsb` → **I2S-10**
13. Full-duplex ต้องสร้าง TX+RX ใน `i2s_new_channel()` call เดียวบน port เดียวกัน (§3)
14. Callback ทำงานหนัก/blocking ใน ISR → ห้าม (§6)

## 8. ตัวอย่างมินิมัล (ตรวจแล้วกับ header v6.1)
```c
#include "driver/i2s_std.h"

void app_main(void)
{
    i2s_chan_handle_t tx = NULL;
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
    ESP_ERROR_CHECK(i2s_new_channel(&chan_cfg, &tx, NULL));

    i2s_std_config_t std_cfg = {
        .clk_cfg  = I2S_STD_CLK_DEFAULT_CONFIG(48000),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_STEREO),
        .gpio_cfg = {
            .mclk = I2S_GPIO_UNUSED,
            .bclk = 12,
            .ws   = 13,
            .dout = 14,
            .din  = I2S_GPIO_UNUSED,
        },
    };
    ESP_ERROR_CHECK(i2s_channel_init_std_mode(tx, &std_cfg));
    ESP_ERROR_CHECK(i2s_channel_enable(tx));

    int16_t frame[2] = { 1000, -1000 };
    size_t written = 0;
    ESP_ERROR_CHECK(i2s_channel_write(tx, frame, sizeof(frame), &written, 1000));
}
```
เทคนิค preload (กันเสียงแตกตอนเริ่ม stream):
```c
size_t bytes_loaded = 0;
i2s_channel_preload_data(tx, init_buf, init_size, &bytes_loaded);  // เรียกก่อน enable (state READY)
i2s_channel_enable(tx);
```

## 9. v6.x Notes
- `i2s_channel_tune_rate(handle, &tune_cfg, &tune_info)` — ปรับ clock ละเอียด เรียกได้เฉพาะ READY state (⚠️ พบใน docs/header v6.1 — ตรวจเวอร์ชันที่ใช้ก่อน)
- `esp_driver_i2s` เป็น component แยกตั้งแต่ v5.3; v6.x legacy `driver` deprecated และไม่ pull public dependency อีก

## 10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
- **Multi-line PDM** (TX `dout2`/`line_mode`, RX `dins[]`) — ยืนยันใน header v6.1 แล้ว แต่เป็นฟีเจอร์ใหม่: ถ้าใช้ IDF < v6.1 ต้องเปิด header ของเวอร์ชันนั้นก่อน
- **`i2s_channel_tune_rate()`** — มีใน v6.1; ตรวจว่ามีจริงในเวอร์ชันที่โปรเจกต์ใช้
- **`ext_clk_freq_hz`** (clock ภายนอก) — มีเมื่อ `SOC_I2S_HW_VERSION_2` (S3 = ใช่) แต่ต้องเช็ค header ของเวอร์ชันเป้าหมาย
- **PDM RX HP filter fields** (`hp_en`/`hp_cut_off_freq_hz`/`amplify_num`) — compile แบบ conditional ตาม `SOC_I2S_SUPPORTS_PDM_RX_HP_FILTER` — เช็ค soc_caps + header ของเวอร์ชันนั้น
- **Sample rate ที่ทำได้จริง** — ตัวเลขใน §2 คือ "supported frequencies" ของ TRM; ความแม่นจริงขึ้นกับ clk_src + divider — ถ้าต้องการ rate ผิดปกติ ทดสอบบนบอร์ดจริง