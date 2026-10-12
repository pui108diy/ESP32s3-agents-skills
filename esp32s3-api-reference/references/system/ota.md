# OTA Update (esp_https_ota + app_update) — v1.1
> ESP-IDF v5.x/v6.x · ESP32-S3 · **สถานะ: Phase 1 แปลงโครงสร้าง §0–§10 แล้ว — ยังไม่ได้ตรวจเชิงลึกกับ header จริง (ดู §10)**
> เนื้อหาทั้งหมดย้ายจากไฟล์เดิม ไม่มีการเติมข้อมูลเทคนิคใหม่ที่ยังไม่ผ่านการตรวจ

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. **Partition table ต้องเป็นแบบ two OTA** (`ota_0`/`ota_1` + `otadata`) — default single factory app = OTA ทำไม่ได้เลย
2. Pattern 1 (ง่าย): `esp_https_ota()` — URL เดียวจบ
3. Pattern 2 (advanced): `begin → perform (วนจนได้ ≠ ESP_ERR_HTTPS_OTA_IN_PROGRESS) → finish`
4. เปิด rollback แล้ว**ลืม `esp_ota_mark_app_valid_cancel_rollback()`** → app ใหม่บูตได้ครั้งเดียว แล้ว reboot วนกลับ app เก่าตลอด
5. ข้าม `esp_ota_end()` แล้ว `set_boot_partition` → `ESP_ERR_OTA_VALIDATE_FAILED` / บูตไม่ขึ้น
6. ห้ามเขียนลง **running partition** — ใช้ `esp_ota_get_next_update_partition()` เสมอ
7. HTTPS ต้องใส่ cert (`.crt_bundle_attach` หรือ `cert_pem` สำหรับ self-signed)
8. WiFi หลุดระหว่าง OTA → handle ค้าง — ต้อง `esp_https_ota_abort()` ก่อนเริ่มใหม่
9. กัน downgrade: `esp_https_ota_get_img_desc()` เทียบ `esp_app_get_description()->version` ก่อน flash
10. `OTA_SIZE_UNKNOWN` ใช้ได้เมื่อไม่รู้ขนาดล่วงหน้า แต่ flash ต้องมีที่พอ

## §1. Include & CMakeLists
```c
#include "esp_https_ota.h"    // high-level HTTPS OTA
#include "esp_ota_ops.h"      // low-level partition operations
#include "esp_ota_app_desc.h" // อ่าน app descriptor (เช็คเวอร์ชันก่อนอัปเดต)
```
- **CMakeLists:** ⚠️ ยังไม่ได้ตรวจ component ที่ต้องระบุใน `REQUIRES` — อยู่ในคิวตรวจ §10

## §2. ความต้องการก่อน (ข้ามขั้นตอนนี้ = พังทั้งอัน)
- **Partition table:** `idf.py menuconfig` → Partition Table → *"Factory app, two OTA definitions"* → ได้ `ota_0`/`ota_1` + `otadata`
- **HTTPS:** ใส่ cert ผ่าน `.crt_bundle_attach = esp_crt_bundle_attach` (หรือ `cert_pem` สำหรับ self-signed)
- **Rollback (ถ้าต้องการ):** `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y` — app ใหม่บูตได้ **ครั้งเดียว** แล้วต้องยืนยัน ไม่งั้นถูก rollback ตอน reboot

## §3. Struct + State Machine ของ OTA image
```c
typedef struct {
    esp_http_client_config_t *http_config;   // .url, .crt_bundle_attach, .timeout_ms ...
    bool partial_http_download;              // v5.3+ ⚠️ โหมดโหลดเป็นชิ้น
    size_t max_http_request_size;            // v5.3+ ⚠️ ขนาดชิ้นสูงสุดต่อ request
} esp_https_ota_config_t;
```
```
ESP_OTA_IMG_NEW → (บูตครั้งแรก) → ESP_OTA_IMG_PENDING_VERIFY
  → esp_ota_mark_app_valid_cancel_rollback() → ESP_OTA_IMG_VALID ✅
  → esp_ota_mark_app_invalid_rollback_and_reboot() → กลับไป app เก่า
```
- ถ้า **ไม่เปิด** rollback config: ฟังก์ชัน mark_valid/invalid เป็น optional และเกมจบที่ set_boot_partition

## §4. High-Level API (esp_https_ota)
```c
// --- Pattern 1: Simple (URL เดียวจบ) ---
esp_err_t esp_https_ota(const esp_https_ota_config_t *ota_config);

// --- Pattern 2: Advanced (รายงานความคืบหน้า/หลายขั้น) ---
esp_err_t esp_https_ota_begin(const esp_https_ota_config_t *ota_config, esp_https_ota_handle_t *handle);
esp_err_t esp_https_ota_perform(esp_https_ota_handle_t handle);
// ↑ วนเรียกจนได้ ESP_OK — ระหว่างทางคืน ESP_ERR_HTTPS_OTA_IN_PROGRESS
esp_err_t esp_https_ota_finish(esp_https_ota_handle_t handle);      // validate + set boot partition
esp_err_t esp_https_ota_abort(esp_https_ota_handle_t handle);
esp_err_t esp_https_ota_get_img_desc(esp_https_ota_handle_t handle, esp_app_desc_t *desc); // เช็คเวอร์ชันก่อน flash
bool      esp_https_ota_is_complete_data_received(esp_https_ota_handle_t handle);
int       esp_https_ota_get_image_size(esp_https_ota_handle_t handle); // v5.2+ ⚠️
```

## §5. Low-Level API (esp_ota_ops) + Rollback
```c
// --- ใช้เมื่อโหลดเอง เช่น จาก SD card ---
const esp_partition_t *esp_ota_get_running_partition(void);
const esp_partition_t *esp_ota_get_next_update_partition(const esp_partition_t *start_from); // NULL = auto
esp_err_t esp_ota_begin(const esp_partition_t *partition, size_t image_size, esp_ota_handle_t *out_handle);
// image_size ไม่รู้ล่วงหน้า = OTA_SIZE_UNKNOWN
esp_err_t esp_ota_write(esp_ota_handle_t handle, const void *data, size_t size);
esp_err_t esp_ota_end(esp_ota_handle_t handle);                      // validate image (checksum/hash)
esp_err_t esp_ota_set_boot_partition(const esp_partition_t *partition);
esp_err_t esp_ota_get_state_partition(const esp_partition_t *partition, esp_ota_img_states_t *ota_state);

// --- Rollback (ต้องเปิด CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE) ---
esp_err_t esp_ota_mark_app_valid_cancel_rollback(void);                       // ← เรียกเมื่อ app ใหม่ทำงานปกติ
esp_err_t esp_ota_mark_app_invalid_rollback_and_reboot(void) __attribute__((noreturn));
```

## §6. VFS
- ไม่มี — OTA ไม่มีการผูก VFS (N/A สำหรับไฟล์นี้)

## §7. กับดักที่พบบ่อย (⚠️ รอผูกรหัส HALLUCINATION-HOT-LIST ในรอบถัดไป)
1. เปิด rollback แล้ว **ลืม `esp_ota_mark_app_valid_cancel_rollback()`** → app ใหม่บูตได้ครั้งเดียว แล้ว reboot วนกลับ app เก่าตลอด (อาการ "อัปเดตแล้วย้อนกลับ")
2. ข้าม `esp_ota_end()` (หรือ image ไม่ครบ) แล้ว `esp_ota_set_boot_partition()` → `ESP_ERR_OTA_VALIDATE_FAILED` / บูตไม่ขึ้น
3. Partition table แบบ single factory app (default!) → ไม่มีที่ว่างสำหรับ ota_1 → OTA ทำไม่ได้เลย
4. HTTPS ไม่ใส่ cert → connection fail (`ESP_ERR_OTA_CONNECTION_FAILED` ⚠️ ชื่อ error ตรวจที่เวอร์ชันใช้)
5. เรียก `esp_ota_write()` ด้วยข้อมูล < size ที่ประกาศ → image เสีย — ส่ง data ตาม byte ที่ได้จริงเสมอ
6. OTA ระหว่าง WiFi หลุด → handle ค้าง — ต้อง `esp_https_ota_abort()` ก่อนเริ่มใหม่
7. เขียนลง partition ที่กำลังรันอยู่ (running partition) → brick ชั่วคราว — ใช้ `esp_ota_get_next_update_partition()` เสมอ
8. `OTA_SIZE_UNKNOWN` ใช้ได้แต่ flash ต้องมีที่พอ — ตรวจ `esp_partition_find` ก่อนในงาน production
9. Version check: ใช้ `esp_https_ota_get_img_desc()` เทียบ `esp_app_get_description()->version` ก่อน flash (กัน downgrade)

## §8. ตัวอย่างมินิมัล (Advanced pattern)
```c
#include "esp_https_ota.h"
#include "esp_crt_bundle.h"

esp_err_t ota_update(const char *url)
{
    esp_http_client_config_t http_cfg = {
        .url = url,
        .crt_bundle_attach = esp_crt_bundle_attach,
        .timeout_ms = 10000,
    };
    esp_https_ota_config_t ota_cfg = { .http_config = &http_cfg };

    esp_https_ota_handle_t h = NULL;
    ESP_ERROR_CHECK(esp_https_ota_begin(&ota_cfg, &h));
    while (1) {
        esp_err_t err = esp_https_ota_perform(h);
        if (err != ESP_ERR_HTTPS_OTA_IN_PROGRESS) {   // ← ยังไม่จบ = คืน IN_PROGRESS
            ESP_ERROR_CHECK(err);                     // จบแล้วต้อง ESP_OK
            break;
        }
        ESP_LOGI("ota", "progress %d / %d bytes",
                 esp_https_ota_get_image_size(h) > 0 ? 0 : 0, 0);   // แสดงความคืบหน้าตามจริงของโปรเจกต์
    }
    ESP_ERROR_CHECK(esp_https_ota_finish(h));         // validate + set boot partition
    esp_restart();
    return ESP_OK;                                     // ไม่มาถึงบรรทัดนี้
}

// ใน app_main หลังบูตด้วย app ใหม่ (เมื่อเปิด rollback):
// esp_ota_mark_app_valid_cancel_rollback();
```

## §9. v6.x Notes
- ไฟล์เดิมมี marker: `partial_http_download`/`max_http_request_size` (v5.3+ ⚠️), `esp_https_ota_get_image_size` (v5.2+ ⚠️) — ยังไม่ verify
- การค้น migration guide (ฐานความรู้) เบื้องต้นไม่พบ breaking change หลักของ esp_https_ota/app_update ใน v6 — ต้องยืนยันกับ header v6.1 จริง (§10)

## §10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
> header เป้าหมาย (tag v6.1): `components/app_update/include/esp_ota_ops.h` + `esp_https_ota` component (path ต้องยืนยัน)
1. CMakeLists `REQUIRES` — app_update / esp_https_ota / esp_http_client / esp_crt_bundle
2. `partial_http_download` / `max_http_request_size` — ยืนยัน v5.3+ + พฤติกรรมจริง
3. `esp_https_ota_get_image_size` — ยืนยัน v5.2+ + คืน `int` จริงไหม
4. ชื่อ error codes จริง: `ESP_ERR_HTTPS_OTA_IN_PROGRESS`, `ESP_ERR_OTA_CONNECTION_FAILED`, `ESP_ERR_OTA_VALIDATE_FAILED`
5. `esp_ota_app_desc.h` — include path จริง + สถานะใน v6 (v6 มีการ reorganize header กลุ่ม flash/app_desc — ตรวจว่ากระทบไหม)
6. `esp_app_get_description()` — ชื่อจริง + อยู่ header ใด
7. `esp_ota_handle_t` type + `OTA_SIZE_UNKNOWN` macro — ยืนยันจาก header
8. Rollback: `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` + พฤติกรรม mark_valid ใน app ใหม่
9. ผูกรหัส HALLUCINATION-HOT-LIST ให้ §7 (ทำพร้อมกับอัปเดต Hot-List)