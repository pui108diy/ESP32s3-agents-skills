# ESP HTTP Client — ESP-IDF v5.x/v6.x (ESP32-S3)

## Includes
```c
#include "esp_http_client.h"
#include "esp_crt_bundle.h"   // HTTPS ผ่าน certificate bundle (CONFIG_MBEDTLS_CERTIFICATE_BUNDLE=y ค่า default)
```

## Config Struct (ฟิลด์ที่ใช้บ่อย)
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

## Function Signatures (exact)
```c
// --- Pattern 1: perform() — ง่ายสุด ---
esp_http_client_handle_t esp_http_client_init(const esp_http_client_config_t *config);  // ← คืน pointer!
esp_err_t esp_http_client_perform(esp_http_client_handle_t client);
esp_err_t esp_http_client_set_url(esp_http_client_handle_t client, const char *url);
esp_err_t esp_http_client_set_method(esp_http_client_handle_t client, esp_http_client_method_t method);
esp_err_t esp_http_client_set_header(esp_http_client_handle_t client, const char *key, const char *value);
esp_err_t esp_http_client_set_post_field(esp_http_client_handle_t client, const char *data, int len);
int      esp_http_client_get_status_code(esp_http_client_handle_t client);
int64_t  esp_http_client_get_content_length(esp_http_client_handle_t client);  // ระวัง: int64_t
esp_err_t esp_http_client_cleanup(esp_http_client_handle_t client);            // ← ห้ามลืม

// --- Pattern 2: streaming — ไฟล์ใหญ่ / ข้อมูลยาว ---
esp_err_t esp_http_client_open(esp_http_client_handle_t client, int write_len);
int       esp_http_client_write(esp_http_client_handle_t client, const char *buffer, int len);
int       esp_http_client_fetch_headers(esp_http_client_handle_t client);
int       esp_http_client_read(esp_http_client_handle_t client, char *buffer, int len);
bool      esp_http_client_is_chunked_response(esp_http_client_handle_t client);
```

## กับดัก Hallucination
1. `ESP_ERROR_CHECK(esp_http_client_init(&cfg))` → คืน handle (pointer) ห้ามครอบ — เช็ค `if (client == NULL)`
2. ลืม `esp_http_client_cleanup()` → memory leak
3. POST: ต้อง `set_method(HTTP_METHOD_POST)` + `set_post_field()` + header `Content-Type` เอง
4. `esp_http_client_read` คืน `-1` = error/ปิดการเชื่อมต่อ, `0` = หมด
5. ข้อมูล response มักเก็บใน event `HTTP_EVENT_ON_DATA` (`evt->data`, `evt->data_len`) ผ่าน `.event_handler`
6. เรียก HTTP ก่อน WiFi ได้ IP → พัง — ผูกกับ wifi.md ข้อ 4
7. `esp_http_client_get_content_length` คืน `int64_t` → printf ด้วย `%"PRId64"`

## ตัวอย่างมินิมัล (Pattern 1)
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