# ESP HTTP Server — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "esp_http_server.h"    // component: esp_http_server
// HTTPS → esp_https_server (roadmap)
```

## Structs
```c
httpd_config_t config = HTTPD_DEFAULT_CONFIG();
// ฟิลด์ที่ปรับบ่อย: .server_port (default 80), .stack_size (default 4096),
// .max_uri_handlers (default 8), .task_priority, .core_id,
// .uri_match_fn (ต้องตั้ง httpd_uri_match_wildcard เพื่อใช้ "/*"),
// .keep_alive_enable, .open_fn, .close_fn, .lru_purge_enable

typedef struct {
    const char *uri;              // เช่น "/api/data" หรือ "/*" (ต้องเปิด wildcard)
    httpd_method_t method;        // HTTP_GET / HTTP_POST / HTTP_DELETE ...
    esp_err_t (*handler)(httpd_req_t *r);   // ← คืน esp_err_t รับ httpd_req_t*
    void *user_ctx;               // context ผูกกับ handler
} httpd_uri_t;
```

## Function Signatures (exact)
```c
// --- Lifecycle ---
esp_err_t httpd_start(httpd_handle_t *handle, const httpd_config_t *config);
esp_err_t httpd_stop(httpd_handle_t handle);        // รอปิด connection อ่อน ๆ ก่อนคืน
esp_err_t httpd_register_uri_handler(httpd_handle_t handle, const httpd_uri_t *uri_handler);
esp_err_t httpd_unregister_uri_handler(httpd_handle_t handle, const char *uri, httpd_method_t method);
esp_err_t httpd_unregister_uri(httpd_handle_t handle, const char *uri);

// --- ใน handler: อ่าน request ---
size_t    httpd_req_get_hdr_value_len(httpd_req_t *r, const char *field);   // 0 = ไม่มี header
esp_err_t httpd_req_get_hdr_value_str(httpd_req_t *r, const char *field, char *val, size_t val_size);
esp_err_t httpd_req_get_url_query_str(httpd_req_t *r, char *buf, size_t buf_len);
esp_err_t httpd_query_key_value(const char *qry_str, const char *key, char *val, size_t val_size);
int       httpd_req_recv(httpd_req_t *r, char *buf, size_t buf_len);        // คืน byte อ่านได้, < 0 = error
// ขนาด body ทั้งหมดอยู่ที่ r->content_len

// --- ใน handler: ตอบ response ---
esp_err_t httpd_resp_send(httpd_req_t *req, const char *buf, ssize_t buf_len);  // buf_len ใช้ HTTPD_RESP_USE_STRLEN (-1) ได้
esp_err_t httpd_resp_send_chunk(httpd_req_t *req, const char *buf, ssize_t buf_len); // buf_len = 0 = จบ chunked
esp_err_t httpd_resp_set_type(httpd_req_t *req, const char *type);         // HTTPD_TYPE_JSON, HTTPD_TYPE_TEXT ...
esp_err_t httpd_resp_set_hdr(httpd_req_t *req, const char *field, const char *value);
esp_err_t httpd_resp_send_err(httpd_req_t *req, httpd_err_code_t error, const char *msg);
esp_err_t httpd_resp_set_status(req, "200 OK") // ชื่อจริง: httpd_resp_set_status(httpd_req_t *req, const char *status)
```

## กับดัก Hallucination
1. **handler ทุกตัวรันใน httpd task เดียว แบบ serialized** — handler ตัวหนึ่ง block ทุก request หยุดหมด; งานหนักส่งต่อ queue ให้ task อื่น
2. อ่าน POST body ด้วย `httpd_req_recv()` **ครั้งเดียว** อาจไม่ครบ → วนรวมยอดจนได้ `r->content_len` bytes
3. URI `"/*"` ใช้ตรง ๆ → 404 — ต้องตั้ง `config.uri_match_fn = httpd_uri_match_wildcard` ก่อน `httpd_start`
4. ลงทะเบียน URI+method ซ้ำ → error; ลบก่อนด้วย `httpd_unregister_uri_handler`
5. `httpd_resp_send` กับ buf_len เป็น `int` บวกเท่านั้น; ใช้ string → `HTTPD_RESP_USE_STRLEN`
6. Response เป็น JSON → `httpd_resp_set_type(req, HTTPD_TYPE_JSON)` (ไม่ต้องเขียน Content-Type เอง)
7. Server ผูกกับ netif ของ WiFi — ต้องได้ IP ก่อนจะ `httpd_start` มีความหมาย (ดู wifi.md)
8. ลืม `httpd_stop()` ตอน teardown → task + socket ค้าง

## ตัวอย่างมินิมัล (GET + POST)
```c
#include "esp_http_server.h"

static esp_err_t get_handler(httpd_req_t *req)
{
    const char *resp = "{\"status\":\"ok\"}";
    httpd_resp_set_type(req, HTTPD_TYPE_JSON);
    return httpd_resp_send(req, resp, HTTPD_RESP_USE_STRLEN);
}

static esp_err_t post_handler(httpd_req_t *req)
{
    char buf[128];
    int received = 0;
    while (received < req->content_len) {                       // วนรวมยอด
        int ret = httpd_req_recv(req, buf + received, sizeof(buf) - received - 1);
        if (ret <= 0) return ESP_FAIL;
        received += ret;
    }
    buf[received] = '\0';
    return httpd_resp_send(req, "accepted", HTTPD_RESP_USE_STRLEN);
}

void start_server(void)
{
    httpd_handle_t server = NULL;
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();
    ESP_ERROR_CHECK(httpd_start(&server, &config));

    httpd_uri_t get_uri = { .uri = "/api/data", .method = HTTP_GET,  .handler = get_handler,  .user_ctx = NULL };
    httpd_uri_t post_uri = { .uri = "/api/data", .method = HTTP_POST, .handler = post_handler, .user_ctx = NULL };
    httpd_register_uri_handler(server, &get_uri);
    httpd_register_uri_handler(server, &post_uri);
}
```