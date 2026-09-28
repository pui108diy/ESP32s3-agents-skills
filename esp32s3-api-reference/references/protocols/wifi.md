# WiFi Station — ESP-IDF v5.x/v6.x (ESP32-S3)

## Includes (ต้องครบทั้ง 4)
```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"    // WiFi ต้องใช้ NVS — ลืมปุ๊บ esp_wifi_init() ล้มเหลว
```

## ลำดับ init ที่ถูกต้อง (จำเป็น — สลับลำดับ = พัง)
1. `nvs_flash_init()` (+ recover pattern ดู nvs.md)
2. `esp_netif_init()`
3. `esp_event_loop_create_default()`
4. `esp_netif_create_default_wifi_sta()`
5. `wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();` → `esp_wifi_init(&cfg)`
6. register event handlers (`WIFI_EVENT` + `IP_EVENT`)
7. `esp_wifi_set_mode(WIFI_MODE_STA)`
8. `esp_wifi_set_config(WIFI_IF_STA, &wifi_config)`
9. `esp_wifi_start()` → รอ `WIFI_EVENT_STA_START` แล้วค่อย `esp_wifi_connect()`

## Function Signatures (exact)
```c
esp_err_t esp_wifi_init(const wifi_init_config_t *config);        // ใช้ macro WIFI_INIT_CONFIG_DEFAULT()
esp_err_t esp_wifi_set_mode(wifi_mode_t mode);                    // WIFI_MODE_STA / AP / APSTA
esp_err_t esp_wifi_set_config(wifi_interface_t interface, wifi_config_t *conf);  // WIFI_IF_STA / WIFI_IF_AP
esp_err_t esp_wifi_start(void);
esp_err_t esp_wifi_stop(void);
esp_err_t esp_wifi_connect(void);
esp_err_t esp_wifi_disconnect(void);
esp_err_t esp_wifi_set_storage(wifi_storage_t storage);           // WIFI_STORAGE_RAM
esp_err_t esp_wifi_scan_start(wifi_scan_config_t *config, bool block);
esp_err_t esp_wifi_scan_get_ap_records(uint16_t *number, wifi_ap_record_t *ap_records);
esp_err_t esp_wifi_deinit(void);

esp_err_t esp_netif_init(void);
esp_netif_t *esp_netif_create_default_wifi_sta(void);             // ← คืน pointer!
esp_err_t esp_event_loop_create_default(void);
esp_err_t esp_event_handler_instance_register(esp_event_base_t event_base, int32_t event_id,
                    esp_event_handler_t event_handler, void *arg, esp_event_handler_instance_t *instance);
```

## Event System
```c
// signature ของ handler — fixed 4 พารามิเตอร์
void handler(void *arg, esp_event_base_t event_base, int32_t event_id, void *event_data);

// อีเวนต์สำคัญ
WIFI_EVENT_STA_START        → เรียก esp_wifi_connect()
WIFI_EVENT_STA_DISCONNECTED → retry (event_data: wifi_event_sta_disconnected_t*)
IP_EVENT_STA_GOT_IP         → ได้ IP (event_data: ip_event_got_ip_t*) — ใช้ IPSTR/IP2STR(&e->ip_info.ip)
```

## wifi_config_t ที่ใช้จริง (STA)
```c
wifi_config_t cfg = {
    .sta = {
        .ssid = "myssid",
        .password = "mypassword",
        .threshold.authmode = WIFI_AUTH_WPA2_PSK,
    },
};
```

## กับดัก Hallucination
1. `ESP_ERROR_CHECK(esp_netif_create_default_wifi_sta())` → คืน pointer ห้ามครอบ
2. ลืม `nvs_flash_init` → esp_wifi_init พัง
3. เรียก `esp_wifi_connect()` ใน `app_main` ตอนที่ STA ยังไม่ START → ใส่ใน event handler
4. เช็ค "เน็ตต่อแล้ว" ด้วย `WIFI_EVENT_STA_CONNECTED` → ยังไม่มี IP ต้องรอ `IP_EVENT_STA_GOT_IP`
5. Kconfig: ต้องเปิด WiFi STA ใน menuconfig (default example มักเปิดแล้ว)

## ตัวอย่างมินิมัล (handler เท่านั้น — ลำดับ init ดูด้านบน)
```c
static void event_handler(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (base == WIFI_EVENT && id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (base == WIFI_EVENT && id == WIFI_EVENT_STA_DISCONNECTED) {
        esp_wifi_connect();   // retry
    } else if (base == IP_EVENT && id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *e = (ip_event_got_ip_t *)data;
        ESP_LOGI("wifi", "got ip:" IPSTR, IP2STR(&e->ip_info.ip));  // ต้อง #include "esp_log.h"
    }
}
```