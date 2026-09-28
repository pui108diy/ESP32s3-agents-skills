# OTA Update (esp_https_ota + app_update) — ESP-IDF v5.x/v6.x (ESP32-S3)

## Includes
```c
#include "esp_https_ota.h"   // high-level HTTPS OTA
#include "esp_ota_ops.h"     // low-level partition operations
#include "esp_ota_app_desc.h" // อ่าน app descriptor (เช็คเวอร์ชันก่อนอัปเดต)
```

## ความต้องการก่อน (ข้ามขั้นตอนนี้ = พังทั้งอัน)
- **Partition table**: `idf.py menuconfig` → Partition Table → *"Factory app, two OTA definitions"* → ได้ `ota_0`/`ota_1` + `otadata`
- HTTPS: ใส่ cert ผ่าน `.crt_bundle_attach = esp_crt_bundle_attach` (หรือ `cert_pem` สำหรับ self-signed)
- Rollback (ถ้าต้องการ): `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y` — app ใหม่บูตได้ **ครั้งเดียว** แล้วต้องยืนยัน ไม่งั้นถูก rollback ตอน reboot

## State Machine ของ OTA image
```
ESP_OTA_IMG_NEW → (บูตครั้งแรก) → ESP_OTA_IMG_PENDING_VERIFY
  → esp_ota_mark_app_valid_cancel_rollback() → ESP_OTA_IMG_VALID ✅
  → esp_ota_mark_app_invalid_rollback_and_reboot() → กลับไป app เก่า
```
- ถ้า **ไม่เปิด** rollback config: ฟังก์ชัน mark_valid/invalid เป็น optional และเกมจบที่ set_boot_partition

## Struct
```c
typedef struct {
    esp_http_client_config_t *http_config;   // .url, .crt_bundle_attach, .timeout_ms ...
    bool partial_http_download;              // v5.3+ ⚠️ โหมดโหลดเป็นชิ้น
    size_t max_http_request_size;            // v5.3+ ⚠️ ขนาดชิ้นสูงสุดต่อ request
} esp_https_ota_config_t;
```

## Function Signatures (exact)
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

// --- Low-level (esp_ota_ops — ใช้เมื่อโหลดเอง เช่น จาก SD card) ---
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

## กับดัก Hallucination
1. เปิด rollback แล้ว **ลืม `esp_ota_mark_app_valid_cancel_rollback()`** → app ใหม่บูตได้ครั้งเดียว แล้ว reboot วนกลับ app เก่าตลอด (อาการ "อัปเดตแล้วย้อนกลับ")
2. ข้าม `esp_ota_end()` (หรือ image ไม่ครบ) แล้ว `esp_ota_set_boot_partition()` → `ESP_ERR_OTA_VALIDATE_FAILED` / บูตไม่ขึ้น
3. Partition table แบบ single factory app (default!) → ไม่มีที่ว่างสำหรับ ota_1 → OTA ทำไม่ได้เลย
4. HTTPS ไม่ใส่ cert → connection fail (`ESP_ERR_OTA_CONNECTION_FAILED` ⚠️ ชื่อ error ตรวจที่เวอร์ชันใช้)
5. เรียก `esp_ota_write()` ด้วยข้อมูล < size ที่ประกาศ → image เสีย — ส่ง data ตาม byte ที่ได้จริงเสมอ
6. OTA ระหว่าง WiFi หลุด → handle ค้าง — ต้อง `esp_https_ota_abort()` ก่อนเริ่มใหม่
7. เขียนลง partition ที่กำลังรันอยู่ (running partition) → brick ชั่วคราว — ใช้ `esp_ota_get_next_update_partition()` เสมอ
8. `OTA_SIZE_UNKNOWN` ใช้ได้แต่ flash ต้องมีที่พอ — ตรวจ `esp_partition_find` ก่อนในงาน production
9. Version check: ใช้ `esp_https_ota_get_img_desc()` เทียบ `esp_app_get_description()->version` ก่อน flash (กัน downgrade)

## ตัวอย่างมินิมัล (Advanced pattern)
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