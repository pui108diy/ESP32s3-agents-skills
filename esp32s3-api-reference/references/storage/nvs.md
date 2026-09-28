# NVS (Non-Volatile Storage) — ESP-IDF v5.x/v6.x (ESP32-S3)

## Includes
```c
#include "nvs_flash.h"   // init / erase / deinit
#include "nvs.h"         // open / get / set / commit / close
```

## Function Signatures (exact)
```c
// --- เริ่มระบบ (เรียกครั้งเดียวตอน boot) ---
esp_err_t nvs_flash_init(void);
esp_err_t nvs_flash_init_partition(const char *partition_label);
esp_err_t nvs_flash_erase(void);
esp_err_t nvs_flash_deinit(void);

// --- เปิด namespace ---
esp_err_t nvs_open(const char *namespace, nvs_open_mode_t open_mode, nvs_handle_t *out_handle);
// open_mode: NVS_READONLY / NVS_READWRITE
esp_err_t nvs_open_from_partition(const char *part_name, const char *namespace,
                                  nvs_open_mode_t open_mode, nvs_handle_t *out_handle);

// --- เขียน ---
esp_err_t nvs_set_i8 (nvs_handle_t h, const char *key, int8_t v);
esp_err_t nvs_set_u8 (nvs_handle_t h, const char *key, uint8_t v);
esp_err_t nvs_set_i16(nvs_handle_t h, const char *key, int16_t v);
esp_err_t nvs_set_u16(nvs_handle_t h, const char *key, uint16_t v);
esp_err_t nvs_set_i32(nvs_handle_t h, const char *key, int32_t v);
esp_err_t nvs_set_u32(nvs_handle_t h, const char *key, uint32_t v);
esp_err_t nvs_set_i64(nvs_handle_t h, const char *key, int64_t v);
esp_err_t nvs_set_u64(nvs_handle_t h, const char *key, uint64_t v);
esp_err_t nvs_set_str(nvs_handle_t h, const char *key, const char *value);
esp_err_t nvs_set_blob(nvs_handle_t h, const char *key, const void *val, size_t len);

// --- อ่าน ---
esp_err_t nvs_get_i32(nvs_handle_t h, const char *key, int32_t *out_value);   // มีครบทุก type เหมือนชุด set
esp_err_t nvs_get_str(nvs_handle_t h, const char *key, char *out_value, size_t *length);   // ⚠️ ต้องเรียก 2 ครั้ง
esp_err_t nvs_get_blob(nvs_handle_t h, const char *key, void *out_value, size_t *length);  // ⚠️ ต้องเรียก 2 ครั้ง

// --- ปิด/ลบ ---
esp_err_t nvs_commit(nvs_handle_t handle);     // ← ห้ามลืมหลัง set ทุกครั้ง!
void      nvs_close(nvs_handle_t handle);
esp_err_t nvs_erase_key(nvs_handle_t handle, const char *key);
esp_err_t nvs_erase_all(nvs_handle_t handle);
```

## Version Marker
- **v5.4**: `nvs_handle_t` เปลี่ยนจาก `uint32_t` เป็น **opaque pointer** ⚠️ → ห้ามสมมติเป็นตัวเลข ห้าม printf ด้วย `%d`
- Default partition = `"nvs"` (NVS_DEFAULT_PART_NAME)

## กับดัก Hallucination
1. `nvs_set_*` แล้ว `nvs_close` เลย **ไม่มี commit** → ข้อมูลหายเมื่อรีเซ็ต
2. `nvs_get_str/blob` ต้องเรียก 2 ครั้ง: ครั้งแรกส่ง `out_value = NULL` เพื่อขอ size (ได้ `ESP_ERR_NVS_INVALID_LENGTH`) → จอง buffer → เรียกครั้งสอง
3. **key ยาวได้สูงสุด 15 ตัวอักษร** (namespace ก็ 15 เช่นกัน)
4. อ่าน key ที่ไม่มี → `ESP_ERR_NVS_NOT_FOUND` (ไม่ใช่ error ร้าย — เช็คแบบ if ปกติ อย่าใช้ ESP_ERROR_CHECK)
5. WiFi ต้องใช้ NVS → ต้อง init NVS ก่อน esp_wifi_init เสมอ (ผูกกับ wifi.md)

## Init Pattern มาตรฐาน (ต้อง recover 2 error นี้)
```c
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());     // ล้างแล้วเริ่มใหม่ (ข้อมูลเก่าหาย!)
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);
```

## ตัวอย่างมินิมัล (set/get int)
```c
nvs_handle_t h;
ESP_ERROR_CHECK(nvs_open("storage", NVS_READWRITE, &h));
ESP_ERROR_CHECK(nvs_set_i32(h, "count", 42));
ESP_ERROR_CHECK(nvs_commit(h));                     // ← ห้ามลืม

int32_t v = 0;
if (nvs_get_i32(h, "count", &v) == ESP_OK) {        // NOT_FOUND = ยังไม่เคยเขียน
    ESP_LOGI("nvs", "count = %" PRId32, v);
}
nvs_close(h);
```