---
name: esp32s3-freertos
version: 1.0.0
description: FreeRTOS skill สำหรับ ESP32-S3 บน ESP-IDF — ครอบคลุม task, queue, semaphore, task notification, software timer, critical section และกฎ ISR safety สำหรับ SMP dual-core
triggers:
  - FreeRTOS
  - task
  - queue
  - semaphore
  - mutex
  - xTaskCreate
  - xQueue
  - xSemaphore
  - vTaskDelay
  - interrupt
  - ISR
  - timer
  - critical section
  - RTOS
---

# Skill: FreeRTOS for ESP32-S3 (ESP-IDF)

> **ขอบเขต:** ใช้สำหรับเขียน C code บน ESP32-S3 ด้วย ESP-IDF ที่ใช้ IDF FreeRTOS (based on Vanilla FreeRTOS v10.5.1 with SMP modifications)
>
> **แหล่งข้อมูลหลัก:** FreeRTOS Reference Manual V10.0.0 + [ESP-IDF FreeRTOS (IDF) Documentation](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/freertos_idf.html)

---

## 1. ข้อแตกต่างสำคัญ: IDF FreeRTOS vs Vanilla FreeRTOS

**⚠️ จำให้ขึ้นใจ — ESP32-S3 เป็น dual-core SMP:**

| หัวข้อ | Vanilla FreeRTOS | IDF FreeRTOS (ESP32-S3) |
|---|---|---|
| Core count | Single core | Dual core (Core 0 = PRO_CPU, Core 1 = APP_CPU) |
| Task creation | `xTaskCreate()` | `xTaskCreatePinnedToCore()` — เพิ่มพารามิเตอร์ `xCoreID` |
| Critical section | ปิด interrupt | ใช้ **spinlock** (`portMUX_TYPE`) เพราะปิด interrupt ไม่พอใน SMP |
| Base version | — | Based on Vanilla FreeRTOS v10.5.1 |

**ค่า xCoreID:**
- `0` = pin ไว้ที่ Core 0 (PRO_CPU) เท่านั้น
- `1` = pin ไว้ที่ Core 1 (APP_CPU) เท่านั้น
- `tskNO_AFFINITY` = รันได้ทั้งสอง core (scheduler เลือกให้)

> แหล่งเว็บ: [ESP-IDF FreeRTOS SMP — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/freertos_idf.html)

---

## 2. Task Management

### 2.1 สร้าง Task

```c
// ESP32-S3 ใช้ xTaskCreatePinnedToCore เพื่อระบุ core
 BaseType_t xTaskCreatePinnedToCore(
     TaskFunction_t pvTaskCode,      // ฟังก์ชัน task
     const char * const pcName,      // ชื่อ (สำหรับ debug)
     const uint32_t usStackDepth,    // ขนาด stack ในหน่วย "words" ไม่ใช่ bytes
     void * const pvParameters,      // พารามิเตอร์ส่งเข้า task
     UBaseType_t uxPriority,         // priority (0 = idle, สูงสุด = configMAX_PRIORITIES-1)
     TaskHandle_t * const pxCreatedTask,  // handle (ใส่ NULL ถ้าไม่ต้องการ)
     const BaseType_t xCoreID        // 0, 1, หรือ tskNO_AFFINITY
 );
```

**⚠️ กฎสำคัญ:**
- `usStackDepth` นับเป็น **words** ไม่ใช่ bytes — ถ้าใส่ 1000 หมายถึง 4000 bytes (32-bit word)
- Task ที่สร้างใหม่จะอยู่ใน **Ready state** ทันที
- ถ้า priority สูงกว่า task ปัจจุบัน → จะ preempt ทันที
- สามารถสร้าง task ได้ทั้งก่อนและหลัง `vTaskStartScheduler()`

### 2.2 โครงสร้าง Task มาตรฐาน

```c
void vMyTask(void *pvParameters) {
    // --- ส่วน initialization (รันครั้งเดียว) ---
    uint32_t param = (uint32_t)pvParameters;

    // --- ส่วน infinite loop (รันตลอด) ---
    for (;;) {
        // task logic ที่นี่
        vTaskDelay(pdMS_TO_TICKS(100)); // หน่วงเวลา 100ms
    }

    // --- จะไม่มาถึงที่นี่ ถ้าไม่เรียก vTaskDelete ---
    vTaskDelete(NULL); // ลบตัวเอง (NULL = ลบ task ปัจจุบัน)
}
```

### 2.3 Delay และ Timing

```c
// หน่วงเวลาแบบ relative (นับจากตอนเรียก)
vTaskDelay(pdMS_TO_TICKS(100));  // หน่วง 100ms

// หน่วงเวลาแบบ absolute (คงที่ ใช้สำหรับ periodic task)
TickType_t xLastWakeTime = xTaskGetTickCount();
for (;;) {
    vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(10)); // ทุก 10ms แม่นยำ
    // task logic
}

// รอไม่จำกัดเวลา
vTaskDelay(portMAX_DELAY);
```

**⚠️ ความแตกต่าง:**
- `vTaskDelay()` — เวลาออกจาก blocked state นับจากเรียก
- `vTaskDelayUntil()` — เวลาออกจาก blocked state คงที่ เหมาะกับ periodic task

> แหล่งข้อมูล: FreeRTOS Reference Manual V10.0.0 — "vTaskDelay() results in the calling task entering the Blocked state for the specified number of ticks from the time vTaskDelay() was called"

### 2.4 Priority

| ค่า | ความหมาย |
|---|---|
| `0` | `tskIDLE_PRIORITY` — เท่ากับ Idle task |
| `1–24` | ปกติใช้ทั่วไป (configMAX_PRIORITIES default = 25) |
| สูงสุด | `configMAX_PRIORITIES - 1` |

**กฎ scheduling:** Priority-based preemptive — task ที่มี priority สูงกว่าจะ preempt task ที่ต่ำกว่าเสมอ

```c
// เปลี่ยน priority ขณะรัน
vTaskPrioritySet(xTaskHandle, 5);

// ดู priority ปัจจุบัน
UBaseType_t uxPriority = uxTaskPriorityGet(NULL);
```

---

## 3. Queue — ส่งข้อมูลระหว่าง Task / ISR

### 3.1 สร้าง Queue

```c
QueueHandle_t xQueue = xQueueCreate(
    10,                        // จำนวน item สูงสุด
    sizeof(struct MyMessage)   // ขนาดแต่ละ item (bytes)
);
// ตรวจสอบ: if (xQueue == NULL) → สร้างไม่สำเร็จ (หน่วยความจำไม่พอ)
```

### 3.2 ส่ง/รับจาก Task

```c
// ส่ง (ไปท้าย queue)
xQueueSend(xQueue, &msg, pdMS_TO_TICKS(100));  // รอสูงสุด 100ms

// ส่งไปหน้า queue (LIFO)
xQueueSendToFront(xQueue, &msg, 0);  // ไม่รอ

// รับ
struct MyMessage received;
xQueueReceive(xQueue, &received, portMAX_DELAY);  // รอไม่จำกัด

// ดูจำนวน item ใน queue
UBaseType_t count = uxQueueMessagesWaiting(xQueue);
```

### 3.3 ส่ง/รับจาก ISR — ⚠️ ต้องใช้ FromISR version

```c
void IRAM_ATTR vMyISR(void *arg) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;  // ⚠️ ต้องตั้งค่าเริ่มต้นเป็น pdFALSE

    struct MyMessage msg = {.id = 1, .value = 42};
    xQueueSendFromISR(xQueue, &msg, &xHigherPriorityTaskWoken);

    // ⚠️ ต้องตรวจสอบและเรียก context switch
    if (xHigherPriorityTaskWoken == pdTRUE) {
        portYIELD_FROM_ISR();
    }
}
```

> แหล่งข้อมูล: FreeRTOS Reference Manual — "xHigherPriorityTaskWoken must be set to pdFALSE before it is used"

---

## 4. Semaphore & Mutex

### 4.1 Binary Semaphore — สำหรับ synchronization (ISR → Task)

```c
// สร้าง
SemaphoreHandle_t xSemaphore = xSemaphoreCreateBinary();
// ⚠️ สร้างมาแล้ว "empty" ต้อง give ก่อนถึง take ได้

// Task: รอ event
xSemaphoreTake(xSemaphore, portMAX_DELAY);  // รอไม่จำกัด
// ... ประมวลผล ...

// ISR: ส่งสัญญาณ
void IRAM_ATTR vISR(void *arg) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xSemaphoreGiveFromISR(xSemaphore, &xHigherPriorityTaskWoken);
    if (xHigherPriorityTaskWoken) portYIELD_FROM_ISR();
}
```

### 4.2 Counting Semaphore — สำหรับนับ resource

```c
SemaphoreHandle_t xSemaphore = xSemaphoreCreateCounting(
    10,  // สูงสุด
    0    // ค่าเริ่มต้น
);
```

### 4.3 Mutex — สำหรับ mutual exclusion (ปกป้อง shared resource)

```c
SemaphoreHandle_t xMutex = xSemaphoreCreateMutex();

// Task A: ล็อก
if (xSemaphoreTake(xMutex, pdMS_TO_TICKS(100)) == pdPASS) {
    // ... เข้าถึง shared resource ...
    xSemaphoreGive(xMutex);  // ⚠️ ต้อง give กลับเสมอ
}

// ⚠️ Mutex มี priority inheritance (ไม่เหมือน binary semaphore)
// ⚠️ ห้ามใช้ mutex ใน ISR — ใช้ binary semaphore แทน
```

**⚠️ ความแตกต่าง Mutex vs Binary Semaphore:**

| คุณสมบัติ | Mutex | Binary Semaphore |
|---|---|---|
| Priority inheritance | ✅ มี | ❌ ไม่มี |
| ใช้ใน ISR | ❌ ห้าม | ✅ ได้ (ผ่าน FromISR) |
| ผู้ take ต้องเป็นผู้ give | ✅ ใช่ | ❌ ไม่จำเป็น |
| วัตถุประสงค์ | ปกป้อง resource | ส่งสัญญาณ event |

> แหล่งข้อมูล: FreeRTOS Reference Manual — "Mutexes and binary semaphores are both referenced using variables that have an SemaphoreHandle_t type"

---

## 5. Task Notification — ทางเลือกที่เร็วและเบากว่า Semaphore

### 5.1 ใช้แทน Binary Semaphore

```c
// Task: รอ notification (แทน xSemaphoreTake)
ulTaskNotifyTake(
    pdTRUE,           // pdTRUE = เคลียร์ค่าเป็น 0 หลังออก (เหมือน binary semaphore)
    portMAX_DELAY     // รอไม่จำกัด
);

// ISR หรือ Task อื่น: ส่ง notification (แทน xSemaphoreGive)
xTaskNotifyGive(xTaskHandle);  // จาก task
vTaskNotifyGiveFromISR(xTaskHandle, &xHigherPriorityTaskWoken);  // จาก ISR
```

### 5.2 ใช้แทน Counting Semaphore

```c
// Task: รอ
ulTaskNotifyTake(
    pdFALSE,          // pdFALSE = ลดค่าทีละ 1 (เหมือน counting semaphore)
    portMAX_DELAY
);
```

> แหล่งข้อมูล: FreeRTOS Reference Manual — "ulTaskNotifyTake() is intended for use when a task notification is used as a faster and lighter weight alternative to a binary semaphore or a counting semaphore"

**⚠️ ข้อดี Task Notification:**
- เร็วกว่า semaphore 45% และใช้หน่วยความจำน้อยกว่า
- ไม่ต้องสร้าง object แยก (ฝังอยู่ใน TCB ของ task)

---

## 6. Software Timer

```c
#include "freertos/timers.h"

// สร้าง timer
TimerHandle_t xTimer = xTimerCreate(
    "MyTimer",              // ชื่อ
    pdMS_TO_TICKS(1000),    // period (ticks)
    pdTRUE,                 // pdTRUE = auto-reload, pdFALSE = one-shot
    (void *)1,              // timer ID (ใช้ใน callback)
    vTimerCallback          // callback function
);

// Callback — ⚠️ รันใน Timer Service Task context ห้าม block!
void vTimerCallback(TimerHandle_t xTimer) {
    int id = (int)pvTimerGetTimerID(xTimer);
    // ทำงานสั้นๆ เท่านั้น ห้าม vTaskDelay หรือรอ semaphore
}

// เริ่ม
xTimerStart(xTimer, pdMS_TO_TICKS(100));

// หยุด
xTimerStop(xTimer, 0);

// เปลี่ยน period
xTimerChangePeriod(xTimer, pdMS_TO_TICKS(500), 0);
```

> แหล่งเว็บ: [controllerstech.com — ESP32 FreeRTOS Software Timers](https://controllerstech.com/esp32-freertos-software-timers-and-task-notifications/)

---

## 7. Critical Section (SMP-safe) — ⚠️ จำเป็นสำหรับ ESP32-S3

**⚠️ ใน SMP การปิด interrupt ไม่พอ — ต้องใช้ spinlock**

### 7.1 จาก Task

```c
// ประกาศ spinlock (static)
static portMUX_TYPE my_spinlock = portMUX_INITIALIZER_UNLOCKED;

// เข้า critical section
taskENTER_CRITICAL(&my_spinlock);
// ... ปกป้อง shared resource (สั้นๆ!) ...
taskEXIT_CRITICAL(&my_spinlock);
```

### 7.2 จาก ISR

```c
void IRAM_ATTR vMyISR(void *arg) {
    BaseType_t saved = taskENTER_CRITICAL_FROM_ISR(&my_spinlock);
    // ... ปกป้อง shared resource ...
    taskEXIT_CRITICAL_FROM_ISR(&my_spinlock, saved);
}
```

> แหล่งเว็บ: [ESP-IDF FreeRTOS (IDF) — Critical Sections](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/system/freertos_idf.html) — "in an SMP system, disabling interrupts is not a valid method of ensuring mutual exclusion. Critical sections that utilize a spinlock should be used instead."

---

## 8. กฎ ISR Safety — ⚠️ อ่านทุกข้อก่อนเขียน ISR

1. **ใช้ `IRAM_ATTR`** เสมอสำหรับฟังก์ชัน ISR และฟังก์ชันที่ ISR เรียก
2. **ใช้ `...FromISR()` version** เท่านั้น — ห้ามใช้ `xQueueSend()`, `xSemaphoreGive()` ใน ISR
3. **ตั้ง `xHigherPriorityTaskWoken = pdFALSE`** ก่อนใช้
4. **เรียก `portYIELD_FROM_ISR()`** ถ้า `xHigherPriorityTaskWoken == pdTRUE`
5. **ห้าม block** — ห้าม `vTaskDelay()`, ห้าม `xSemaphoreTake(x, portMAX_DELAY)`
6. **ห้าม malloc/free** — ใช้ queue หรือ pre-allocated buffer แทน
7. **ห้าม printf** — ใช้ `ESP_DRAM_LOGI` หรือส่งผ่าน queue ไป task
8. **สั้นที่สุด** — defer งานหนักไป task ผ่าน queue/semaphore

---

## 9. โครงสร้างไฟล์ ESP-IDF มาตรฐาน

```
my_project/
├── CMakeLists.txt
├── main/
│   ├── CMakeLists.txt
│   ├── main.c              ← app_main() อยู่ที่นี่
│   └── my_task.c
└── components/
    └── my_component/
        ├── CMakeLists.txt
        ├── include/
        │   └── my_component.h
        └── my_component.c
```

```c
// main.c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "esp_log.h"

static const char *TAG = "MAIN";

void app_main(void) {
    ESP_LOGI(TAG, "Starting...");

    // สร้าง task ที่นี่ (scheduler รันอยู่แล้วใน ESP-IDF)
    xTaskCreatePinnedToCore(vMyTask, "MyTask", 4096, NULL, 5, NULL, 0);

    // app_main จะ return ได้ — scheduler ยังรันต่อ
}
```

**⚠️ ข้อสำคัญ:** ใน ESP-IDF, scheduler เริ่มรันก่อน `app_main()` แล้ว — **ไม่ต้องเรียก `vTaskStartScheduler()`**

---

## 10. Checklist ก่อนส่งมอบโค้ด

- [ ] ทุก task ใช้ `xTaskCreatePinnedToCore` ไม่ใช่ `xTaskCreate`
- [ ] `usStackDepth` นับเป็น words (คูณ 4 เพื่อได้ bytes)
- [ ] ทุก ISR มี `IRAM_ATTR`
- [ ] ใน ISR ใช้ `...FromISR()` version เท่านั้น
- [ ] มี `portYIELD_FROM_ISR()` หลัง ISR ถ้าจำเป็น
- [ ] Mutex ไม่ถูกใช้ใน ISR
- [ ] Critical section ใช้ spinlock (`portMUX_TYPE`) ไม่ใช่ปิด interrupt ธรรมดา
- [ ] Timer callback ไม่มี blocking call
- [ ] มี error handling หลัง `xQueueCreate`, `xSemaphoreCreate*`, `xTaskCreate*`
- [ ] ไม่มี `vTaskStartScheduler()` ใน `app_main()` (ESP-IDF ทำให้แล้ว)

---

## 11. Common Mistakes ที่ต้องหลีกเลี่ยง

| ผิด | ถูก | เหตุผล |
|---|---|---|
| `xTaskCreate(...)` | `xTaskCreatePinnedToCore(..., xCoreID)` | ESP32-S3 เป็น SMP |
| `usStackDepth = 4000` (หมายถึง 4KB) | `usStackDepth = 1000` (1000 words = 4KB) | นับเป็น words |
| `xSemaphoreGive()` ใน ISR | `xSemaphoreGiveFromISR()` | ISR ต้องใช้ FromISR |
| Mutex ใน ISR | Binary semaphore ใน ISR | Mutex มี priority inheritance ไม่ปลอดภัยใน ISR |
| `taskENTER_CRITICAL()` ไม่มี spinlock | `taskENTER_CRITICAL(&spinlock)` | SMP ต้องมี spinlock |
| `vTaskStartScheduler()` ใน app_main | ลบออก | ESP-IDF เริ่ม scheduler ให้แล้ว |
| `printf()` ใน ISR | ส่งผ่าน queue ไป task | printf ไม่ ISR-safe และช้า |