# WiFi Station — v1.1
> ESP-IDF v5.x/v6.x · ESP32-S3 · **สถานะ: Phase 1 แปลงโครงสร้าง §0–§10 แล้ว — ยังไม่ได้ตรวจเชิงลึกกับ header จริง (ดู §10)**
> เนื้อหาทั้งหมดย้ายจากไฟล์เดิม ไม่มีการเติมข้อมูลเทคนิคใหม่ที่ยังไม่ผ่านการตรวจ

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. Include ต้องครบ **4 ตัว**: esp_wifi, esp_event, esp_netif, nvs_flash
2. **ลำดับ init 9 ขั้น — สลับลำดับ = พัง** (ดู §2)
3. `esp_wifi_connect()` ต้องเรียก**หลัง `WIFI_EVENT_STA_START`** (ใน event handler ไม่ใช่ app_main)
4. "เน็ตต่อแล้ว" = `IP_EVENT_STA_GOT_IP` **ไม่ใช่** `WIFI_EVENT_STA_CONNECTED` (ยังไม่มี IP)
5. `esp_netif_create_default_wifi_sta()` คืน **pointer** — ห้าม ESP_ERROR_CHECK
6. ลืม `nvs_flash_init()` → `esp_wifi_init()` ล้มเหลว (WiFi ต้องใช้ NVS)
7. Retry pattern: เรียก `esp_wifi_connect()` ซ้ำใน `WIFI_EVENT_STA_DISCONNECTED`
8. แสดง IP ใช้ `IPSTR` / `IP2STR(&e->ip_info.ip)`
9. `esp_wifi_set_storage(WIFI_STORAGE_RAM)` — ไม่บันทึก config ลง NVS ของตัวเอง
10. Kconfig: ต้องเปิด WiFi STA ใน menuconfig (default example มักเปิดแล้ว)

## §1. Include & CMakeLists
```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"    // WiFi ต้องใช้ NVS — ลืมปุ๊บ esp_wifi_init() ล้มเหลว
```
- **CMakeLists:** ⚠️ ยังไม่ได้ตรวจ component ที่ต้องระบุใน `REQUIRES` — อยู่ในคิวตรวจ §10

## §2. ลำดับ init ที่ถูกต้อง (จำเป็น — สลับลำดับ = พัง)
1. `nvs_flash_init()` (+ recover pattern ดู nvs.md)
2. `esp_netif_init()`
3. `esp_event_loop_create_default()`
4. `esp_netif_create_default_wifi_sta()`
5. `wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();` → `esp_wifi_init(&cfg)`
6. register event handlers (`WIFI_EVENT` + `IP_EVENT`)
7. `esp_wifi_set_mode(WIFI_MODE_STA)`
8. `esp_wifi_set_config(WIFI_IF_STA, &wifi_config)`
9. `esp_wifi_start()` → รอ `WIFI_EVENT_STA_START` แล้วค่อย `esp_wifi_connect()`

## §3. Structs & Config
```c
// wifi_config_t ที่ใช้จริง (STA)
wifi_config_t cfg = {
    .sta = {
        .ssid = "myssid",
        .password = "mypassword",
        .threshold.authmode = WIFI_AUTH_WPA2_PSK,
    },
};

// signature ของ event handler — fixed 4 พารามิเตอร์
void handler(void *arg, esp_event_base_t event_base, int32_t event_id, void *event_data);
```

## §4. API
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

## §5. Event System
```c
// อีเวนต์สำคัญ
WIFI_EVENT_STA_START        → เรียก esp_wifi_connect()
WIFI_EVENT_STA_DISCONNECTED → retry (event_data: wifi_event_sta_disconnected_t*)
IP_EVENT_STA_GOT_IP         → ได้ IP (event_data: ip_event_got_ip_t*) — ใช้ IPSTR/IP2STR(&e->ip_info.ip)
```

## §6. VFS
- ไม่มี — WiFi ไม่มีการผูก VFS (N/A สำหรับไฟล์นี้)

## §7. กับดักที่พบบ่อย (⚠️ รอผูกรหัส HALLUCINATION-HOT-LIST ในรอบถัดไป)
1. `ESP_ERROR_CHECK(esp_netif_create_default_wifi_sta())` → คืน pointer ห้ามครอบ
2. ลืม `nvs_flash_init` → esp_wifi_init พัง
3. เรียก `esp_wifi_connect()` ใน `app_main` ตอนที่ STA ยังไม่ START → ใส่ใน event handler
4. เช็ค "เน็ตต่อแล้ว" ด้วย `WIFI_EVENT_STA_CONNECTED` → ยังไม่มี IP ต้องรอ `IP_EVENT_STA_GOT_IP`
5. Kconfig: ต้องเปิด WiFi STA ใน menuconfig (default example มักเปิดแล้ว)

## §8. ตัวอย่างมินิมัล (handler เท่านั้น — ลำดับ init ดู §2)
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

## §9. v6.x Notes
- ไฟล์เดิมไม่มี version marker — การค้น migration guide (ฐานความรู้) เบื้องต้นไม่พบ breaking change หลักของ esp_wifi ใน v6 แต่ยังสรุปไม่ได้ — ต้องตรวจกับ header v6.1 จริง (§10)

## §10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
> header เป้าหมาย (tag v6.1): `components/esp_wifi/include/esp_wifi.h` + `esp_netif` + `esp_event` (path ต้องยืนยัน)
1. CMakeLists `REQUIRES` — esp_wifi / esp_event / esp_netif / nvs_flash
2. `wifi_init_config_t` + `WIFI_INIT_CONFIG_DEFAULT()` — ฟิลด์/macro ยืนยันจาก header
3. `wifi_config_t` union ฟิลด์เต็ม (sta/ap — เช่น threshold.authmode, scan_method ฯลฯ)
4. Event data structs: `wifi_event_sta_disconnected_t`, `ip_event_got_ip_t` — ฟิลด์เต็ม
5. `esp_wifi_scan_start` block semantics + `wifi_scan_config_t` ฟิลด์
6. Kconfig ชื่อจริงของการเปิด WiFi STA
7. `esp_wifi_set_storage` — enum ค่าครบไหม
8. การเปลี่ยนแปลง v6.x จาก migration guide ฉบับเต็ม (โดยเฉพาะกลุ่ม esp_wifi/esp_netif ถ้ามี)
9. ผูกรหัส HALLUCINATION-HOT-LIST ให้ §7 (ทำพร้อมกับอัปเดต Hot-List)