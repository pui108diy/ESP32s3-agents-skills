# RMT (Remote Control Transceiver) — ESP-IDF v5.x/v6.x (ESP32-S3)

## Includes
```c
#include "driver/rmt_tx.h"     // TX + encoder (ครอบ rmt_encoder.h อยู่ในตัว)
#include "driver/rmt_rx.h"     // RX
```
> v5.3+/v6: code อยู่ใน component `esp_driver_rmt` (header `driver/*.h` ชื่อเดิม)

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 37)
- **8 ช่อง: ch0–3 = TX, ch4–7 = RX** — ทุกช่องมี clock divider + state machine ของตัวเอง
- ทั้ง 8 ช่อง**แชร์ RAM 384 × 32-bit** (= **48 words ต่อช่อง** เมื่อใช้ครบ 8 ช่อง)
- **Carrier modulation บน TX / demodulation + filter บน RX** (สำหรับ IR 38 kHz ไม่ต้องทำเองในซอฟต์แวร์)
- TX: Wrap mode (loop ข้อมูลใน RAM), **Continuous TX**, นับรอบ loop ได้
- **DMA: TX ได้เฉพาะ ch3, RX เฉพาะ ch7** — สำหรับข้อมูลยาว ๆ

## โมเดลของ New Driver (v5.0+) — แตกต่างจาก legacy สิ้นเชิง
```
TX: rmt_new_tx_channel → rmt_enable → rmt_transmit(payload → encoder → symbols → RAM/DMA → สาย)
    encoder 3 ชั้นได้: bytes → copy → กำหนดเอง (chain) เช่น header(bytes) + body(copy)
RX: rmt_new_rx_channel → rmt_enable → rmt_receive(จอง buffer) → event on_recv_done → re-arm ทุกครั้ง!
```

## Structs (ฟิลด์ที่ใช้จริง)
```c
// หน่วยพื้นฐานของทุกอย่าง: 1 word = 2 เฟส (duration+level)
typedef union {
    struct {
        uint32_t duration0 : 15;   // หน่วย = tick ตาม resolution_hz (ไม่ใช่ ns!)
        uint32_t level0    : 1;
        uint32_t duration1 : 15;
        uint32_t level1    : 1;
    };
    uint32_t val;
} rmt_symbol_word_t;               // duration สูงสุด 32767 ticks

typedef struct {
    gpio_num_t gpio_num;
    rmt_clock_source_t clk_src;        // RMT_CLK_SRC_DEFAULT
    uint32_t resolution_hz;            // ตั้ง 1000000 → 1 tick = 1 µs
    size_t mem_block_symbols;          // S3: 48 (1 block); มากกว่านั้นต้อง DMA/ยืม block ว่าง
    size_t trans_queue_depth;          // คิว transaction (ให้ >= 1)
    struct { uint32_t invert_out:1; uint32_t with_dma:1; /* ... */ } flags;
} rmt_tx_channel_config_t;             // rx คล้ายกัน + ไม่มี trans_queue_depth

typedef struct {
    int loop_count;                    // 0 = ไม่ loop
    struct { uint32_t eot_level:1; /* ... */ } flags;
} rmt_transmit_config_t;

typedef struct {
    uint32_t signal_range_min_ns;      // กรอง spike สั้นกว่านี้ทิ้ง
    uint32_t signal_range_max_ns;      // ถือว่าเป็น idle ถ้ายาวกว่านี้
    struct { uint32_t en_partial_rx:1; } flags;
} rmt_receive_config_t;

typedef struct {
    rmt_symbol_word_t bit0;            // symbol ของ bit 0
    rmt_symbol_word_t bit1;            // symbol ของ bit 1
    struct { uint32_t msb_first:1; } flags;
} rmt_bytes_encoder_config_t;

typedef struct { /* ว่าง — copy encoder ไม่มี config */ } rmt_copy_encoder_config_t;

// events
typedef struct {
    const rmt_symbol_word_t *received_symbols;   // ชี้เข้า buffer ที่ส่งให้ rmt_receive
    size_t num_of_symbols;
} rmt_rx_done_event_data_t;

typedef struct {
    bool (*on_recv_done)(rmt_channel_handle_t channel, const rmt_rx_done_event_data_t *edata, void *user_ctx);
} rmt_rx_event_callbacks_t;          // TX: rmt_tx_event_callbacks_t { on_trans_done }
```

## Function Signatures (exact)
```c
// --- Channel ---
esp_err_t rmt_new_tx_channel(const rmt_tx_channel_config_t *config, rmt_tx_channel_handle_t *ret_tx_channel);
esp_err_t rmt_new_rx_channel(const rmt_rx_channel_config_t *config, rmt_rx_channel_handle_t *ret_rx_channel);
esp_err_t rmt_del_tx_channel(rmt_tx_channel_handle_t channel);     // ต้อง disable ก่อน
esp_err_t rmt_del_rx_channel(rmt_rx_channel_handle_t channel);
esp_err_t rmt_enable(rmt_channel_handle_t channel);                // ← ต้องเรียกก่อน transmit/receive!
esp_err_t rmt_disable(rmt_channel_handle_t channel);
esp_err_t rmt_apply_carrier(rmt_channel_handle_t channel, const rmt_carrier_config_t *config);
// TX = ใส่ carrier 38 kHz / RX = demod; config = NULL = ปิด

// --- TX ---
esp_err_t rmt_transmit(rmt_tx_channel_handle_t tx_channel, rmt_encoder_handle_t encoder,
                       const void *payload, size_t payload_bytes, const rmt_transmit_config_t *config); // async!
esp_err_t rmt_tx_wait_all_done(rmt_tx_channel_handle_t tx_channel, int timeout_ms);   // -1 = รอไม่จำกัด
esp_err_t rmt_tx_set_loop_count(rmt_tx_channel_handle_t tx_channel, uint32_t loop_count);
esp_err_t rmt_tx_register_event_callbacks(rmt_tx_channel_handle_t channel,
                                          const rmt_tx_event_callbacks_t *cbs, void *user_data);

// --- RX ---
esp_err_t rmt_receive(rmt_rx_channel_handle_t rx_channel, void *buffer, size_t buffer_size,
                      const rmt_receive_config_t *config);          // ← one-shot: re-arm ทุกครั้งหลังได้ข้อมูล
esp_err_t rmt_rx_register_event_callbacks(rmt_rx_channel_handle_t channel,
                                          const rmt_rx_event_callbacks_t *cbs, void *user_data);

// --- Encoders ---
esp_err_t rmt_new_bytes_encoder(const rmt_bytes_encoder_config_t *config, rmt_encoder_handle_t *ret_encoder);
esp_err_t rmt_new_copy_encoder(const rmt_copy_encoder_config_t *config, rmt_encoder_handle_t *ret_encoder);
esp_err_t rmt_del_encoder(rmt_encoder_handle_t encoder);

// --- Sync (ส่งหลายช่องพร้อมกัน) ---
esp_err_t rmt_new_sync_manager(const rmt_sync_manager_config_t *config, rmt_sync_manager_handle_t *ret_synchromgr);
```

## Version Marker
- Legacy `driver/rmt.h` (`rmt_config()`, `rmt_item32_t`, `rmt_write_items()`) — deprecated v5.0, **ถูกลบใน v6** → ใช้ tx/rx/encoder ชุดใหม่เท่านั้น
- Kconfig ที่เกี่ยวข้อง: `CONFIG_RMT_ISR_IRAM_SAFE` / `CONFIG_RMT_TX_ISR_CACHE_SAFE` / `CONFIG_RMT_RX_ISR_CACHE_SAFE` / `CONFIG_RMT_RECV_FUNC_IN_IRAM`
- Encoder function ที่เขียนเอง ตกแต่งด้วย `RMT_ENCODER_FUNC_ATTR`

## กับดัก Hallucination
1. `rmt_symbol_word_t` ใส่ duration เป็น **ns** → ผิด มันคือ **tick ตาม `resolution_hz`** (ตั้ง 1 MHz ให้ tick = µs ง่ายสุด)
2. ลืม `rmt_enable()` ก่อน transmit → `ESP_ERR_INVALID_STATE`
3. `rmt_transmit()` เป็น **async** — free/แก้ payload ทันทีหลังเรียก = use-after-free → ต้อง `rmt_tx_wait_all_done()` ก่อน
4. `rmt_receive()` เป็น **one-shot** — ได้ event แล้วต้องเรียกใหม่ทุกครั้ง (มิฉะนั้นรับครั้งเดียวแล้วเงียบ)
5. Buffer ที่ส่งให้ `rmt_receive()` ต้องอยู่ถึงตอน event มา (อย่าเป็น local ที่ตายไปแล้ว)
6. **48 symbols/ช่อง** บน S3 (RAM แชร์ 384 words) — IR ธรรมดาพอ, สัญญาณยาวต้องเปิด DMA (TX เฉพาะ ch3)
7. Encoder **มี state** — ใช้ encoder เดียวกันกับหลาย channel พร้อมกัน = race (สร้างคนละตัว)
8. **NeoPixel/WS2812 อย่าเขียน encoder เอง** — ใช้ component `led_strip` (ดูด้านล่าง)
9. ส่ง IR 38 kHz: ใช้ `rmt_apply_carrier()` ให้ฮาร์ดแวร์ modulate — อย่าเข้ารหัส carrier ลงใน symbol เอง
10. Callback RX รันใน ISR context — ห้าม printf, คืน `true` เมื่อปลุก task priority สูงขึ้น

## ตัวอย่างมินิมัล (TX + bytes encoder, tick = 1 µs)
```c
#include "driver/rmt_tx.h"

void app_main(void)
{
    rmt_tx_channel_config_t tx_cfg = {
        .gpio_num = 4,
        .clk_src = RMT_CLK_SRC_DEFAULT,
        .resolution_hz = 1000000,      // 1 tick = 1 µs
        .mem_block_symbols = 48,       // S3: 1 block = 48 words
        .trans_queue_depth = 4,
    };
    rmt_tx_channel_handle_t tx = NULL;
    ESP_ERROR_CHECK(rmt_new_tx_channel(&tx_cfg, &tx));
    ESP_ERROR_CHECK(rmt_enable(tx));   // ← ก่อน transmit เสมอ

    rmt_bytes_encoder_config_t enc_cfg = {   // สไตล์ NEC: bit0 = 560+560, bit1 = 560+1690 (µs)
        .bit0 = { .duration0 = 560, .level0 = 1, .duration1 = 560,  .level1 = 0 },
        .bit1 = { .duration0 = 560, .level0 = 1, .duration1 = 1690, .level1 = 0 },
        .flags.msb_first = true,
    };
    rmt_encoder_handle_t encoder = NULL;
    ESP_ERROR_CHECK(rmt_new_bytes_encoder(&enc_cfg, &encoder));

    uint32_t payload = 0x00FFA259;     // NEC: addr + ~addr + cmd + ~cmd
    rmt_transmit_config_t t_cfg = { .loop_count = 0 };
    ESP_ERROR_CHECK(rmt_transmit(tx, encoder, &payload, sizeof(payload), &t_cfg));
    ESP_ERROR_CHECK(rmt_tx_wait_all_done(tx, -1));   // ← รอก่อนแตะ payload ซ้ำ

    ESP_ERROR_CHECK(rmt_del_encoder(encoder));
    ESP_ERROR_CHECK(rmt_disable(tx));
    ESP_ERROR_CHECK(rmt_del_tx_channel(tx));
}
```

## ทางลัด: NeoPixel ใช้ component `led_strip` (อย่าเขียน RMT เอง)
```c
// idf.py add-dependency "espressif/led_strip"
#include "led_strip.h"
led_strip_config_t s = { .strip_gpio_num = 8, .max_leds = 30, .led_model = LED_MODEL_WS2812 };
led_strip_rmt_config_t r = { .resolution_hz = 10 * 1000 * 1000 };
led_strip_handle_t strip;
led_strip_new_rmt_device(&s, &r, &strip);
led_strip_set_pixel(strip, 0, 255, 0, 0);   // index, R, G, B
led_strip_refresh(strip);
```
> IR protocol สำเร็จรูป: มองหา component `ESP-IR` / `ir_remote` ใน Component Registry