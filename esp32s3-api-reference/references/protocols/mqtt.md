# ESP-MQTT Client — v1.1
> ESP-IDF v5.x/v6.x · ESP32-S3 · **สถานะ: Phase 1 แปลงโครงสร้าง §0–§10 แล้ว — ยังไม่ได้ตรวจเชิงลึกกับ header จริง (ดู §10)**
> เนื้อหาทั้งหมดย้ายจากไฟล์เดิม ไม่มีการเติมข้อมูลเทคนิคใหม่ที่ยังไม่ผ่านการตรวจ

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. config แบบ v5+ เป็น**โครงสร้างซ้อน** — `.broker.address.uri` — สไตล์ v4 (`.host`/`.port`) = compile error
2. `esp_mqtt_client_init()` คืน **pointer** ไม่ใช่ esp_err_t
3. ลำดับ: `init → register_event → start` — ลืม `esp_mqtt_client_start()` = ไม่มีการเชื่อมต่อเลย
4. subscribe ต้องรอ `MQTT_EVENT_CONNECTED` — ก่อนหน้านั้นคืน -1
5. `evt->data` **ไม่มี null-terminator** — ใช้ `%.*s` คู่กับ `data_len` เสมอ
6. payload ใหญ่กว่า `.buffer.size` → `MQTT_EVENT_DATA` ถูกแบ่งหลาย chunk — เช็ค `total_data_len` + `current_data_offset`
7. `publish()` คืน: `>0` = msg_id (QoS>0), `0` = QoS0 สำเร็จ, `-1` = fail — **0 ไม่ใช่ error!**
8. `enqueue()` + `store=true` = ยัดเข้า outbox รอส่งเมื่อ online
9. MQTT วิ่งบน TCP/IP — ต้องได้ IP จาก WiFi ก่อน (ผูกกับ wifi.md)
10. `esp_mqtt_client_unsubscribe()` มีพารามิเตอร์ qos ด้วย (คนละเรื่องกับ publish) ⚠️ รอยืนยัน §10

## §1. Include & CMakeLists
```c
#include "mqtt_client.h"   // component: mqtt (esp-mqtt)
```
- **CMakeLists:** ⚠️ ยังไม่ได้ตรวจ — อยู่ในคิวตรวจ §10

## §2. ภาพใหญ่
- MQTT = protocol component (ไม่มี hardware เฉพาะ) วิ่งบน TCP/IP → **ต้องได้ IP จาก WiFi ก่อน** (ดู wifi.md ลำดับ init + รอ `IP_EVENT_STA_GOT_IP`)
- โมเดล: publish/subscribe ผ่าน broker, QoS 0/1/2, keepalive, Last Will (LWT), outbox buffer สำหรับข้อความที่ส่งตอน offline

## §3. Structs
```c
// v5.x = โครงสร้างซ้อน (nested) — ต่างจาก v4.x ที่เป็น field ตรง ๆ ⚠️
esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = "mqtt://broker.emqx.io:1883",  // mqtt:// = 1883, mqtts:// = 8883, ws://
    .broker.address.port = 1883,
    .broker.verification.certificate = NULL,             // TLS: PEM ของ CA
    .credentials.username = "user",
    .credentials.authentication.password = "pass",
    .credentials.client_id = NULL,                       // default ESP32_%CHIPID%
    .session.keepalive = 15,
    .session.last_will.topic = "status/device1",         // LWT
    .session.last_will.msg = "offline",
    .session.last_will.qos = 1,
    .session.last_will.retain = false,
    .network.timeout_ms = 10000,
    .network.disable_auto_reconnect = false,             // default = auto reconnect
    .buffer.size = 1024,                                 // rx buffer — payload ใหญ่ต้องขยาย!
    .buffer.out_size = 512,                              // tx buffer
    .task.stack_size = 6144,                             // stack ของ mqtt task
};

typedef struct {
    esp_mqtt_event_id_t event_id;
    esp_mqtt_client_handle_t client;
    void *user_context;
    char *data;              // payload — ⚠️ ไม่มี null-terminator!
    int data_len;            // ใช้คู่กับ data เสมอ
    int total_data_len;      // payload รวม (เมื่อถูกแบ่งเป็นหลาย event)
    int current_data_offset; // offset ของ chunk ปัจจุบัน
    char *topic;
    int topic_len;
    int msg_id;
    int session_present;
} esp_mqtt_event_t;
```

## §4. Client Lifecycle API
```c
esp_mqtt_client_handle_t esp_mqtt_client_init(const esp_mqtt_client_config_t *config); // ← คืน pointer!
esp_err_t esp_mqtt_client_set_uri(esp_mqtt_client_handle_t client, const char *uri);
esp_err_t esp_mqtt_client_start(esp_mqtt_client_handle_t client);   // ← ต้องเรียกเองหลัง init
esp_err_t esp_mqtt_client_reconnect(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_disconnect(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_stop(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_destroy(esp_mqtt_client_handle_t client); // หลัง stop เมื่อไม่ใช้แล้ว
```

## §5. Pub/Sub API + Events
```c
int esp_mqtt_client_subscribe(esp_mqtt_client_handle_t client, const char *topic, int qos);
// คืน msg_id (>=0) หรือ -1 ถ้า fail
int esp_mqtt_client_unsubscribe(esp_mqtt_client_handle_t client, const char *topic, int qos);
// ⚠️ มีพารามิเตอร์ qos ด้วย (คนละเรื่องกับ publish)

int esp_mqtt_client_publish(esp_mqtt_client_handle_t client, const char *topic,
                            const char *data, int len, int qos, int retain);
// คืน: msg_id (>0) ถ้า QoS>0, 0 ถ้า QoS0 สำเร็จ, -1 ถ้า fail
int esp_mqtt_client_enqueue(esp_mqtt_client_handle_t client, const char *topic,
                            const char *data, int len, int qos, int retain, bool store);
// ยัดเข้า outbox รอส่งเมื่อ online (store=true)

esp_err_t esp_mqtt_client_register_event(esp_mqtt_client_handle_t client, esp_mqtt_event_id_t event,
                                         esp_event_handler_t event_handler, void *event_handler_arg);
// มักใช้ ESP_EVENT_ANY_ID ครอบทุก event ใน handler เดียว
```
- **Events:** `MQTT_EVENT_BEFORE_CONNECT`, `MQTT_EVENT_CONNECTED`, `MQTT_EVENT_DISCONNECTED`, `MQTT_EVENT_SUBSCRIBED`, `MQTT_EVENT_UNSUBSCRIBED`, `MQTT_EVENT_PUBLISHED`, `MQTT_EVENT_DATA`, `MQTT_EVENT_ERROR`, `MQTT_EVENT_DELETED`

## §6. VFS
- ไม่มี — MQTT ไม่มีการผูก VFS (N/A สำหรับไฟล์นี้)

## §7. กับดักที่พบบ่อย (⚠️ รอผูกรหัส HALLUCINATION-HOT-LIST ในรอบถัดไป)
1. เขียน `.host` / `.port` (สไตล์ **v4.x**) บน v5+ → **compile error** — ต้อง `.broker.address.uri`
2. ใช้ `evt->data` เป็น string ตรง ๆ → **ไม่มี null-terminator** ต้อง copy + `\0` เองหรือใช้ `%.*s` กับ `data_len`
3. Payload ใหญ่กว่า `.buffer.size` → `MQTT_EVENT_DATA` ถูกแบ่งหลาย chunk → เช็ค `total_data_len` + `current_data_offset`
4. Subscribe ก่อน `MQTT_EVENT_CONNECTED` → คืน -1
5. `esp_mqtt_client_publish()` คืน 0 เมื่อ QoS0 สำเร็จ — เข้าใจ 0 เป็น error
6. ลืม `esp_mqtt_client_start()` หลัง init → ไม่มีการเชื่อมต่อเลย
7. `esp_mqtt_client_destroy()` ตอน task กำลังรัน → ต้อง stop ก่อน
8. เรียก publish ก่อน WiFi ได้ IP → fail/ค้าง — ผูกกับ wifi.md

## §8. ตัวอย่างมินิมัล (สมมติ WiFi ต่อแล้ว)
```c
#include "mqtt_client.h"

static void mqtt_event_handler(void *arg, esp_event_base_t base, int32_t id, void *event_data)
{
    esp_mqtt_event_t *evt = (esp_mqtt_event_t *)event_data;
    switch ((esp_mqtt_event_id_t)id) {
    case MQTT_EVENT_CONNECTED:
        esp_mqtt_client_subscribe(evt->client, "sensor/+/temp", 1);   // subscribe เมื่อเชื่อมต่อแล้วเท่านั้น
        esp_mqtt_client_publish(evt->client, "status/device1", "online", 6, 1, 0);
        break;
    case MQTT_EVENT_DATA:
        ESP_LOGI("mqtt", "TOPIC=%.*s", evt->topic_len, evt->topic);     // %.*s เพราะไม่มี '\0'
        ESP_LOGI("mqtt", "DATA=%.*s", evt->data_len, evt->data);
        break;
    default:
        break;
    }
}

void mqtt_start(void)
{
    const esp_mqtt_client_config_t cfg = {
        .broker.address.uri = "mqtt://broker.emqx.io:1883",
        .session.keepalive = 15,
        .buffer.size = 1024,
    };
    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&cfg);  // คืน pointer ไม่ใช่ esp_err_t
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    esp_mqtt_client_start(client);                                  // ← อย่าลืม!
}
```

## §9. v6.x Notes
- **v4.x → v5.x** (จากไฟล์เดิม): config เปลี่ยนจาก field ตรง (`.host`/`.port`) เป็นโครงสร้างซ้อน (`.broker.address.uri` ฯลฯ)
- **v6.x:** ไฟล์เดิมไม่มีข้อมูล — การค้น migration guide (ฐานความรู้) เบื้องต้นยังไม่พบ breaking change หลัก แต่ต้องตรวจกับ header v6.1 จริง (ดู §10)

## §10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
> header เป้าหมาย (tag v6.1): `components/mqtt/esp-mqtt/include/mqtt_client.h` (path ต้องยืนยัน)
1. CMakeLists `REQUIRES` — mqtt (esp-mqtt)
2. `esp_mqtt_client_config_t` ฟิลด์เต็ม vs header v6.1 — v6 มีเปลี่ยนโครงสร้างเพิ่มเติมจาก v5 ไหม
3. `esp_mqtt_event_t` ฟิลด์เต็ม
4. ลายเซ็น `esp_mqtt_client_unsubscribe` — มีพารามิเตอร์ qos จริงไหม (ไฟล์เดิม mark ⚠️)
5. Return values ของ `publish`/`enqueue`/`subscribe` — ยืนยันจาก header docs
6. รายการ events ครบไหม (9 events จากไฟล์เดิม vs header)
7. การเปลี่ยนแปลง v6.x อื่น ๆ จาก migration guide ฉบับเต็ม
8. ผูกรหัส HALLUCINATION-HOT-LIST ให้ §7 (ทำพร้อมกับอัปเดต Hot-List)