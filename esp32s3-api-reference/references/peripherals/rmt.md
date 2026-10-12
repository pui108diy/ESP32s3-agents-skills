# RMT (Remote Control Transceiver) — New Driver — v1.1
> ESP-IDF v5.x/v6.x · ESP32-S3 · **สถานะ: Phase 1 แปลงโครงสร้าง §0–§10 แล้ว — ยังไม่ได้ตรวจเชิงลึกกับ header จริง (ดู §10)**
> เนื้อหาทั้งหมดย้ายจากไฟล์เดิม ไม่มีการเติมข้อมูลเทคนิคใหม่ที่ยังไม่ผ่านการตรวจ

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. S3 มี **8 ช่อง: ch0–3 = TX, ch4–7 = RX** — แต่ละช่องมี clock divider + state machine แยกกัน
2. แชร์ RAM **384 × 32-bit words** (สูงสุด 48 words ต่อช่อง)
3. `duration` ใน `rmt_symbol_word_t` คือ **tick ตาม `resolution_hz` ไม่ใช่ ns!**
4. ต้องเรียก `rmt_enable()` ก่อนใช้ทุกครั้ง
5. `rmt_transmit()` เป็น **async** — ต้อง `rmt_tx_wait_all_done()` ก่อนแตะ payload/encoder ซ้ำ
6. `rmt_receive()` เป็น **one-shot** — ต้อง re-arm ใหม่ทุกครั้งหลังได้รับข้อมูล
7. Encoder มี **state** — ห้ามใช้ตัวแปร encoder เดียวกับหลาย channel พร้อมกัน
8. DMA: TX รองรับเฉพาะ **ch3**, RX เฉพาะ **ch7**
9. NeoPixel → ใช้ component `led_strip`; IR สำเร็จรูป → `ESP-IR`/`ir_remote`; carrier 38 kHz → `rmt_apply_carrier()` (ฮาร์ดแวร์)
10. Legacy (`rmt_config()`, `rmt_item32_t`, `rmt_write_items()`) deprecated v5.0 **ถูกลบใน v6** → ใช้ชุด tx/rx/encoder ใหม่เท่านั้น

## §1. Include & CMakeLists
```c
#include "driver/rmt_tx.h"     // TX + encoder (ครอบ rmt_encoder.h อยู่ในตัว)
#include "driver/rmt_rx.h"     // RX
// v5.3+/v6: component esp_driver_rmt (header driver/*.h ชื่อเดิม)
```
- **CMakeLists:** ⚠️ ยังไม่ได้ตรวจว่าต้องระบุ `esp_driver_rmt` (หรือ `driver` ใน v5.2) ใน `REQUIRES` หรือถูกดึงมาแบบ transitively — อยู่ในคิวตรวจ §10

## §2. Hardware Facts (ESP32-S3)
- **8 ช่อง: ch0–3 = TX, ch4–7 = RX** — แต่ละช่องมี clock divider และ state machine แยกกัน
- **แชร์ RAM 384 × 32-bit** (สูงสุด 48 words ต่อช่อง)
- รองรับ **carrier modulation (TX)** และ **demodulation/filter (RX)** ในระดับฮาร์ดแวร์
- **DMA: TX รองรับเฉพาะ ch3, RX รองรับเฉพาะ ch7**

## §3. Structs
```c
typedef union {
    struct {
        uint32_t duration0 : 15;
        uint32_t level0    : 1;
        uint32_t duration1 : 15;
        uint32_t level1    : 1;
    };
    uint32_t val;
} rmt_symbol_word_t;                    // 1 word = 2 เฟส (duration + level); duration หน่วยเป็น tick!

typedef struct {
    gpio_num_t gpio_num;
    rmt_clock_source_t clk_src;
    uint32_t resolution_hz;
    size_t mem_block_symbols;
    size_t trans_queue_depth;
    struct { uint32_t invert_out:1; uint32_t with_dma:1; } flags;
} rmt_tx_channel_config_t;

typedef struct {
    uint32_t signal_range_min_ns;
    uint32_t signal_range_max_ns;
    struct { uint32_t en_partial_rx:1; } flags;
} rmt_receive_config_t;

typedef struct {
    rmt_symbol_word_t bit0;
    rmt_symbol_word_t bit1;
    struct { uint32_t msb_first:1; } flags;
} rmt_bytes_encoder_config_t;

// ชื่อ struct ที่มีในไฟล์เดิมแต่ยังไม่มีรายละเอียดฟิลด์ (คิวตรวจ §10):
// rmt_rx_done_event_data_t, rmt_rx_event_callbacks_t
```

## §4. TX/RX & Encoder API
```c
// --- Channel ---
esp_err_t rmt_new_tx_channel(const rmt_tx_channel_config_t *config, rmt_tx_channel_handle_t *ret_tx_channel);
esp_err_t rmt_new_rx_channel(const rmt_rx_channel_config_t *config, rmt_rx_channel_handle_t *ret_rx_channel);
esp_err_t rmt_del_tx_channel(rmt_tx_channel_handle_t channel);
esp_err_t rmt_del_rx_channel(rmt_rx_channel_handle_t channel);
esp_err_t rmt_enable(rmt_channel_handle_t channel);
esp_err_t rmt_disable(rmt_channel_handle_t channel);
esp_err_t rmt_apply_carrier(rmt_channel_handle_t channel, const rmt_carrier_config_t *config);

// --- TX ---
esp_err_t rmt_transmit(rmt_tx_channel_handle_t tx_channel, rmt_encoder_handle_t encoder,
                       const void *payload, size_t payload_bytes, const rmt_transmit_config_t *config);   // async!
esp_err_t rmt_tx_wait_all_done(rmt_tx_channel_handle_t tx_channel, int timeout_ms);
esp_err_t rmt_tx_set_loop_count(rmt_tx_channel_handle_t tx_channel, uint32_t loop_count);
esp_err_t rmt_tx_register_event_callbacks(rmt_tx_channel_handle_t channel, const rmt_tx_event_callbacks_t *cbs, void *user_data);

// --- RX (one-shot) ---
esp_err_t rmt_receive(rmt_rx_channel_handle_t rx_channel, void *buffer, size_t buffer_size,
                      const rmt_receive_config_t *config);
esp_err_t rmt_rx_register_event_callbacks(rmt_rx_channel_handle_t channel, const rmt_rx_event_callbacks_t *cbs, void *user_data);

// --- Encoder ---
esp_err_t rmt_new_bytes_encoder(const rmt_bytes_encoder_config_t *config, rmt_encoder_handle_t *ret_encoder);
esp_err_t rmt_new_copy_encoder(const rmt_copy_encoder_config_t *config, rmt_encoder_handle_t *ret_encoder);
esp_err_t rmt_del_encoder(rmt_encoder_handle_t encoder);

// --- Sync ---
esp_err_t rmt_new_sync_manager(const rmt_sync_manager_config_t *config, rmt_sync_manager_handle_t *ret_synchromgr);
```

## §5. โมเดล New Driver + Kconfig + การใช้งานพิเศษ
- **โมเดลของ New Driver (v5.0+) ต่างจาก legacy สิ้นเชิง** — แยก TX/RX channel + encoder chain (bytes → copy → ต่อกันได้)
- **Kconfig ที่เกี่ยวข้อง:** `CONFIG_RMT_ISR_IRAM_SAFE`, `CONFIG_RMT_TX_ISR_CACHE_SAFE`, `CONFIG_RMT_RX_ISR_CACHE_SAFE`, `CONFIG_RMT_RECV_FUNC_IN_IRAM` ⚠️ (ชื่อ/default ยังไม่ verify — §10)
- **Encoder ที่เขียนเอง:** ตกแต่ง function ด้วย `RMT_ENCODER_FUNC_ATTR` ⚠️ (รายละเอียดเต็มอยู่ในคิวตรวจ §10)
- **NeoPixel/WS2812:** แนะนำใช้ component `led_strip` แทนการเขียน RMT เอง
- **IR:** ใช้ component `ESP-IR` หรือ `ir_remote` จาก Component Registry สำหรับโปรโตคอลสำเร็จรูป; การ modulate 38 kHz ใช้ `rmt_apply_carrier()` ที่ฮาร์ดแวร์

## §6. VFS
- ไม่มี — RMT ไม่มีการผูก VFS (N/A สำหรับไฟล์นี้)

## §7. กับดักที่พบบ่อย (⚠️ รอผูกรหัส HALLUCINATION-HOT-LIST ในรอบถัดไป)
1. ค่า `duration` ใน `rmt_symbol_word_t` คือ **tick ตามค่า `resolution_hz`** ไม่ใช่หน่วย ns
2. ต้องเรียก `rmt_enable()` ก่อนเสมอ และ `rmt_transmit()` เป็นแบบ async — ต้องใช้ `rmt_tx_wait_all_done()` ก่อนจัดการ payload/encoder ซ้ำ
3. `rmt_receive()` เป็นแบบ **one-shot** — ต้องเรียก re-arm ใหม่ทุกครั้งหลังจากได้รับข้อมูล
4. Encoder มี state — ไม่ควรใช้ตัวแปรเดียวกับหลาย channel พร้อมกัน
5. Callback ของ RX รันใน **ISR context** — ห้ามใช้ `printf` และต้องระวังการคืนค่า `true` เพื่อปลุก task

## §8. ตัวอย่างมินิมัล (TX + bytes encoder — NEC-style bit)
```c
#include "driver/rmt_tx.h"

void app_main(void)
{
    rmt_tx_channel_config_t tx_cfg = {
        .gpio_num = 4,
        .clk_src = RMT_CLK_SRC_DEFAULT,
        .resolution_hz = 1000000,
        .mem_block_symbols = 48,
        .trans_queue_depth = 4,
    };
    rmt_tx_channel_handle_t tx = NULL;
    ESP_ERROR_CHECK(rmt_new_tx_channel(&tx_cfg, &tx));
    ESP_ERROR_CHECK(rmt_enable(tx));

    rmt_bytes_encoder_config_t enc_cfg = {
        .bit0 = { .duration0 = 560, .level0 = 1, .duration1 = 560,  .level1 = 0 },
        .bit1 = { .duration0 = 560, .level0 = 1, .duration1 = 1690, .level1 = 0 },
        .flags.msb_first = true,
    };
    rmt_encoder_handle_t encoder = NULL;
    ESP_ERROR_CHECK(rmt_new_bytes_encoder(&enc_cfg, &encoder));

    uint32_t payload = 0x00FFA259;
    rmt_transmit_config_t t_cfg = { .loop_count = 0 };
    ESP_ERROR_CHECK(rmt_transmit(tx, encoder, &payload, sizeof(payload), &t_cfg));
    ESP_ERROR_CHECK(rmt_tx_wait_all_done(tx, -1));

    ESP_ERROR_CHECK(rmt_del_encoder(encoder));
    ESP_ERROR_CHECK(rmt_disable(tx));
    ESP_ERROR_CHECK(rmt_del_tx_channel(tx));
}
```

## §9. v6.x Notes
- **v5.3+/v6:** โค้ดอยู่ใน component `esp_driver_rmt` (header `driver/*.h` ชื่อเดิม)
- Legacy `driver/rmt.h` (`rmt_config()`, `rmt_item32_t`, `rmt_write_items()`) — deprecated ตั้งแต่ v5.0 **และถูกลบใน v6** → ต้องใช้ชุด API `tx/rx/encoder` ใหม่เท่านั้น

## §10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
> header เป้าหมาย (tag v6.1): `components/esp_driver_rmt/include/driver/rmt_tx.h` + `rmt_rx.h` (path ต้องยืนยัน)
1. สิ่งที่ต้องระบุใน CMakeLists `REQUIRES` ต่อเวอร์ชัน (v5.2: `driver` / v5.3+: `esp_driver_rmt`)
2. `rmt_rx_channel_config_t` ฟิลด์เต็ม — ไฟล์เดิมไม่มีรายละเอียด
3. Struct ที่เหลือ: `rmt_copy_encoder_config_t`, `rmt_carrier_config_t`, `rmt_transmit_config_t`, `rmt_sync_manager_config_t`, `rmt_rx_done_event_data_t`, `rmt_rx_event_callbacks_t`, `rmt_tx_event_callbacks_t` — โครงสร้างเต็ม vs header
4. Callback signatures (`on_rx_done`, `on_tx_done`) — คืนค่าอะไร พารามิเตอร์ครบไหม
5. การเขียน encoder เอง — `rmt_encode_t` signature + `RMT_ENCODER_FUNC_ATTR` คืออะไร (macro จริงไหม)
6. Kconfig 4 ตัว — ชื่อจริงและค่า default ต่อเวอร์ชัน
7. ⚠️ จาก migration guide v6 (ฐานความรู้ migration 5.x→6.x): `io_od_mode` ใน `rmt_tx_channel_config_t` ถูกลบ → ต้องเรียก `gpio_od_enable()` เอง — ตรวจกับ header v6.1 แล้วเขียนลง §9
8. จำนวน channel (8) + DMA channel (TX ch3, RX ch7) — ยืนยันกับ soc_caps (`SOC_RMT_TX_CANDIDATES_PER_GROUP` ฯลฯ)
9. v5.2 ใช้ path/ชื่อ component อะไร
10. ผูกรหัส HALLUCINATION-HOT-LIST ให้ §7 (ทำพร้อมกับอัปเดต Hot-List)