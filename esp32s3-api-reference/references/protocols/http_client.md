# ESP HTTP Client — v1.1

> ESP-IDF v5.x/v6.x · ESP32-S3 · **สถานะ: Phase 1 แปลงโครงสร้าง §0–§10 แล้ว — ยังไม่ได้ตรวจเชิงลึกกับ header จริง (ดู §10)**
> เนื้อหาทั้งหมดย้ายจากไฟล์เดิม ไม่มีการเติมข้อมูลเทคนิคใหม่ที่ยังไม่ผ่านการตรวจ

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. `esp_http_client_init()` คืน **pointer** (`esp_http_client_handle_t`) — เช็ค `if (client == NULL)` ไม่ใช่ ESP_ERROR_CHECK
2. Pattern 1 ง่ายสุด: `init → perform → cleanup` — perform จบทั้ง request
3. Pattern 2 streaming: `open → write → fetch_headers → read` — สำหรับไฟล์ใหญ่/ข้อมูลยาว
4. ลืม `esp_http_client_cleanup()` = memory leak
5. POST: ต้อง `set_method(HTTP_METHOD_POST)` + `set_post_field()` + header `Content-Type` เอง
6. Response มักรับผ่าน event handler — event `HTTP_EVENT_ON_DATA` (`evt->data`, `evt->data_len`)
7. HTTPS: `.crt_bundle_attach = esp_crt_bundle_attach` (bundle, default เปิด) หรือ `.cert_pem` สำหรับ self-signed
8. `esp_http_client_read()` คืน `-1` = error/ปิดการเชื่อมต่อ, `0` = หมด
9. `esp_http_client_get_content_length()` คืน `int64_t` → printf ด้วย `%"PRId64"`
10. เรียก HTTP ก่อน WiFi ได้ IP → พัง — ผูกกับ wifi.md ข้อ 4

## §1. Include & CMakeLists
```c
#include "esp_http_client.h"
#include "esp_crt_bundle.h"   // HTTPS ผ่าน certificate bundle (CONFIG_MBEDTLS_CERTIFICATE_BUNDLE=y ค่า default)
```
- **CMakeLists:** ⚠️ ยังไม่ได้ตรวจ component ที่ต้องระบุใน `REQUIRES` — อยู่ในคิวตรวจ §10

## §2. ภาพใหญ่
- ไม่มี hardware facts ในไฟล์เดิม — software component วิ่งบน TCP/IP (ต้องได้ IP จาก WiFi ก่อน — ดู §7 ข้อ 6 และ wifi.md)

## §3. Config Struct (ฟิลด์ที่ใช้บ่อย)
```c
esp_http_client_config_t config = {
    .url = "https://api.example.com/data",
    .method = HTTP_METHOD_GET,                  // HTTP_METHOD_GET/POST/PUT/DELETE...
    .timeout_ms = 5000,
    .buffer_size = 2048,                        // rx buffer (default เล็ก)
    .buffer_size_tx = 1024,
    .event_handler = _http_event_handler,       // รับ esp_http_client_event_t*
    .crt_bundle_attach = esp_crt_bundle_attach, // HTTPS แบบ bundle
    // .cert_pem = pem_ptr,                     // สำหรับ self-signed แทน bundle
};
```

## §4. API — Pattern 1: perform() (ง่ายสุด)
```c
esp_http_client_handle_t esp_http_client_init(const esp_http_client_config_t *config);  // ← คืน pointer!
esp_err_t esp_http_client_perform(esp_http_client_handle_t client);
esp_err_t esp_http_client_set_url(esp_http_client_handle_t client, const char *url);
esp_err_t esp_http_client_set_method(esp_http_client_handle_t client, esp_http_client_method_t method);
esp_err_t esp_http_client_set_header(esp_http_client_handle_t client, const char *key, const char *value);
esp_err_t esp_http_client_set_post_field(esp_http_client_handle_t client, const char *data, int len);
int      esp_http_client_get_status_code(esp_http_client_handle_t client);
int64_t  esp_http_client_get_content_length(esp_http_client_handle_t client);  // ระวัง: int64_t
esp_err_t esp_http_client_cleanup(esp_http_client_handle_t client);            // ← ห้ามลืม
```

## §5. API — Pattern 2: Streaming + Event Handler
```c
esp_err_t esp_http_client_open(esp_http_client_handle_t client, int write_len);
int       esp_http_client_write(esp_http_client_handle_t client, const char *buffer, int len);
int       esp_http_client_fetch_headers(esp_http_client_handle_t client);
int       esp_http_client_read(esp_http_client_handle_t client, char *buffer, int len);
bool      esp_http_client_is_chunked_response(esp_http_client_handle_t client);
```
- **รับ response ผ่าน event handler:** ตั้ง `.event_handler` ใน config — ข้อมูลมาที่ event `HTTP_EVENT_ON_DATA` ผ่าน `evt->data` + `evt->data_len`

## §6. VFS
- ไม่มี — ESP HTTP Client ไม่มีการผูก VFS (N/A สำหรับไฟล์นี้)

## §7. กับดักที่พบบ่อย (⚠️ รอผูกรหัส HALLUCINATION-HOT-LIST ในรอบถัดไป)
1. `ESP_ERROR_CHECK(esp_http_client_init(&cfg))` → คืน handle (pointer) ห้ามครอบ — เช็ค `if (client == NULL)`
2. ลืม `esp_http_client_cleanup()` → memory leak
3. POST: ต้อง `set_method(HTTP_METHOD_POST)` + `set_post_field()` + header `Content-Type` เอง
4. `esp_http_client_read` คืน `-1` = error/ปิดการเชื่อมต่อ, `0` = หมด
5. ข้อมูล response มักเก็บใน event `HTTP_EVENT_ON_DATA` (`evt->data`, `evt->data_len`) ผ่าน `.event_handler`
6. เรียก HTTP ก่อน WiFi ได้ IP → พัง — ผูกกับ wifi.md ข้อ 4
7. `esp_http_client_get_content_length` คืน `int64_t` → printf ด้วย `%"PRId64"`

## §8. ตัวอย่างมินิมัล (Pattern 1)
```c
#include "esp_http_client.h"
#include "esp_crt_bundle.h"

void http_get(void)
{
    esp_http_client_config_t cfg = {
        .url = "https://api.example.com/data",
        .timeout_ms = 5000,
        .crt_bundle_attach = esp_crt_bundle_attach,
    };
    esp_http_client_handle_t client = esp_http_client_init(&cfg);   // คืน pointer!
    esp_err_t err = esp_http_client_perform(client);
    if (err == ESP_OK) {
        ESP_LOGI("http", "status=%d len=%"PRId64,
                 esp_http_client_get_status_code(client),
                 esp_http_client_get_content_length(client));
    } else {
        ESP_LOGE("http", "failed: %s", esp_err_to_name(err));
    }
    esp_http_client_cleanup(client);
}
```

## §9. v6.x Notes
- ไม่มี version marker ในไฟล์เดิม — การค้น migration guide (ฐานความรู้) เบื้องต้นยังไม่พบ breaking change หลักของ esp_http_client ใน v6 แต่ยังสรุปไม่ได้ — ต้องตรวจกับ header v6.1 จริง (ดู §10)

## §10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
> header เป้าหมาย (tag v6.1): `components/esp_http_client/include/esp_http_client.h` (path ต้องยืนยัน)
1. CMakeLists `REQUIRES` — esp_http_client (+ esp-tls / esp_crt_bundle ต้องระบุไหม)
2. `esp_http_client_config_t` ฟิลด์เต็ม vs header v6.1 (โดยเฉพาะ timeout_ms, buffer_size/buffer_size_tx, crt_bundle_attach, cert_pem, event_handler)
3. Semantics ของ `esp_http_client_read`/`write` (คืน -1/0) — ยืนยันจาก header docs
4. `CONFIG_MBEDTLS_CERTIFICATE_BUNDLE` — ชื่อจริงและค่า default ต่อเวอร์ชัน
5. ชื่อ type จริงของ event struct (`esp_http_client_event_t`) + รายการ event ครบไหม
6. การเปลี่ยนแปลง v6.x อื่น ๆ จาก migration guide ฉบับเต็ม
7. ผูกรหัส HALLUCINATION-HOT-LIST ให้ §7 (ทำพร้อมกับอัปเดต Hot-List)