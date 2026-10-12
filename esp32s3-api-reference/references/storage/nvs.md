# NVS (Non-Volatile Storage) — v1.1
> ESP-IDF v5.x/v6.x · ESP32-S3 · **สถานะ: Phase 1 แปลงโครงสร้าง §0–§10 แล้ว — ยังไม่ได้ตรวจเชิงลึกกับ header จริง (ดู §10)**
> เนื้อหาทั้งหมดย้ายจากไฟล์เดิม ไม่มีการเติมข้อมูลเทคนิคใหม่ที่ยังไม่ผ่านการตรวจ

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. `nvs_flash_init()` ครั้งเดียวตอน boot + ต้อง recover 2 error: `ESP_ERR_NVS_NO_FREE_PAGES` / `ESP_ERR_NVS_NEW_VERSION_FOUND` → erase → init ใหม่
2. ทุก `nvs_set_*` ต้องปิดท้ายด้วย `nvs_commit()` — ลืม = ข้อมูลหายเมื่อรีเซ็ต
3. key ยาวสูงสุด **15 ตัวอักษร** — namespace ก็ 15 เช่นกัน
4. `nvs_get_str/blob` ต้องเรียก **2 ครั้ง** (ครั้งแรกส่ง `out_value = NULL` เพื่อขอ size)
5. อ่าน key ที่ยังไม่เคยเขียน → `ESP_ERR_NVS_NOT_FOUND` — เช็คด้วย if ปกติ อย่าใช้ ESP_ERROR_CHECK
6. **v5.4+:** `nvs_handle_t` เป็น **opaque pointer** — ห้ามสมมติเป็นตัวเลข ห้าม printf ด้วย `%d`
7. WiFi ต้องใช้ NVS → ต้อง init NVS ก่อน `esp_wifi_init` เสมอ
8. default partition = `"nvs"` (NVS_DEFAULT_PART_NAME) — ใช้ partition อื่นได้ผ่าน `_init_partition`/`_from_partition`
9. เปิดเป็นราย namespace (`NVS_READONLY`/`NVS_READWRITE`) แล้วถือ handle ไว้ใช้
10. `erase_key`/`erase_all` ก็ต้อง commit เช่นเดียวกับ set

## §1. Include & CMakeLists
```c
#include "nvs_flash.h"   // init / erase / deinit
#include "nvs.h"         // open / get / set / commit / close
```
- **CMakeLists:** ⚠️ ยังไม่ได้ตรวจ — อยู่ในคิวตรวจ §10

## §2. ภาพใหญ่
- ไม่มี hardware facts ในไฟล์เดิม — NVS เป็นระบบจัดเก็บข้อมูลบน flash partition (ไม่ใช่ peripheral) — default partition `"nvs"` (ดู §9)

## §3. Init & Namespace API
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
```

## §4. Read/Write API (เต็มทุก type)
```c
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

## §5. Patterns มาตรฐาน
- **Init Pattern (recover 2 error — ต้องทำเสมอ):**
```c
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());     // ล้างแล้วเริ่มใหม่ (ข้อมูลเก่าหาย!)
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);
```
- **get_str/blob แบบ 2 ครั้ง:** ครั้งแรกส่ง `out_value = NULL` เพื่อขอ size (ได้ `ESP_ERR_NVS_INVALID_LENGTH`) → จอง buffer → เรียกครั้งสอง

## §6. VFS
- ไม่มี — NVS ไม่มีการผูก VFS (N/A สำหรับไฟล์นี้)

## §7. กับดักที่พบบ่อย (⚠️ รอผูกรหัส HALLUCINATION-HOT-LIST ในรอบถัดไป)
1. `nvs_set_*` แล้ว `nvs_close` เลย **ไม่มี commit** → ข้อมูลหายเมื่อรีเซ็ต
2. `nvs_get_str/blob` ต้องเรียก 2 ครั้ง: ครั้งแรกส่ง `out_value = NULL` เพื่อขอ size (ได้ `ESP_ERR_NVS_INVALID_LENGTH`) → จอง buffer → เรียกครั้งสอง
3. **key ยาวได้สูงสุด 15 ตัวอักษร** (namespace ก็ 15 เช่นกัน)
4. อ่าน key ที่ไม่มี → `ESP_ERR_NVS_NOT_FOUND` (ไม่ใช่ error ร้าย — เช็คแบบ if ปกติ อย่าใช้ ESP_ERROR_CHECK)
5. WiFi ต้องใช้ NVS → ต้อง init NVS ก่อน esp_wifi_init เสมอ (ผูกกับ wifi.md)

## §8. ตัวอย่างมินิมัล (set/get int)
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

## §9. v6.x Notes
- **v5.4:** `nvs_handle_t` เปลี่ยนจาก `uint32_t` เป็น **opaque pointer** ⚠️ → ห้ามสมมติเป็นตัวเลข ห้าม printf ด้วย `%d`
- Default partition = `"nvs"` (NVS_DEFAULT_PART_NAME)
- **v6.x:** ไฟล์เดิมไม่มีข้อมูล — การค้น migration guide (ฐานความรู้) เบื้องต้นยังไม่พบ breaking change หลัก ต้องยืนยันกับ header v6.1 (§10)

## §10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
> header เป้าหมาย (tag v6.1): `components/nvs_flash/include/nvs_flash.h` + `components/nvs_flash/include/nvs.h` (path ต้องยืนยัน)
1. CMakeLists `REQUIRES` — nvs_flash
2. v5.4 opaque pointer — ยืนยันจาก header v6.1 จริง
3. 2-call pattern ของ get_str/blob + semantics ของ `ESP_ERR_NVS_INVALID_LENGTH`
4. key/namespace จำกัด 15 ตัวอักษร — ยืนยันจาก header comment
5. ชุด set/get ครบทุก type ไหม vs header
6. `NVS_DEFAULT_PART_NAME` = `"nvs"` — ยืนยัน
7. การเปลี่ยนแปลง v6.x อื่น ๆ จาก migration guide ฉบับเต็ม
8. ผูกรหัส HALLUCINATION-HOT-LIST ให้ §7 (ทำพร้อมกับอัปเดต Hot-List)