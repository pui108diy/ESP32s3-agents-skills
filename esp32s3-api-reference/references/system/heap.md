# Heap / Memory Allocation API — ESP-IDF v5.x/v6.x (ESP32-S3)

> เสริม skill `esp32s3-memory-map` (ที่อยู่ regions) — ไฟล์นี้คือ **API จัดสรร**

## Includes
```c
#include "esp_heap_caps.h"   // heap_caps_*
#include "esp_system.h"      // esp_get_free_heap_size() ฯลฯ
```

## Capability Flags (บน S3)
| Flag | ความหมาย | อยู่ที่ |
|---|---|---|
| `MALLOC_CAP_8BIT` | เขียนอ่านราย byte ได้ | DRAM ภายใน + PSRAM |
| `MALLOC_CAP_DMA` | **DMA เข้าถึงได้** (SPI/I2S/RMT/UART) | DRAM ภายในเท่านั้น |
| `MALLOC_CAP_INTERNAL` | ภายใน chip | DRAM/IRAM |
| `MALLOC_CAP_SPIRAM` | PSRAM ภายนอก | PSRAM |
| `MALLOC_CAP_EXEC` | รันโค้ดได้ | IRAM |
| `MALLOC_CAP_DEFAULT` | ตามพฤติของ `malloc()` | ขึ้นกับ Kconfig SPIRAM |

## Function Signatures (exact)
```c
void  *heap_caps_malloc(size_t size, uint32_t caps);
void  *heap_caps_calloc(size_t n, size_t size, uint32_t caps);
void  *heap_caps_realloc(void *ptr, size_t size, uint32_t caps);
void  *heap_caps_aligned_alloc(size_t alignment, size_t size, uint32_t caps); // v5.0+, alignment = 2^n
void   heap_caps_aligned_free(void *ptr);
void   heap_caps_free(void *ptr);            // free() ใช้แทนได้ (heap เดียวทั้งระบบ)

size_t heap_caps_get_free_size(uint32_t caps);
size_t heap_caps_get_largest_free_block(uint32_t caps);   // ← วัด fragmentation จริง
void   heap_caps_print_heap_info(uint32_t caps);
void   heap_caps_print_all_info(void);
bool   heap_caps_check_integrity_all(bool print_errors);
esp_err_t heap_caps_malloc_extmem_enable(size_t threshold); // malloc() เริ่มใช้ PSRAM เมื่อ > threshold

uint32_t esp_get_free_heap_size(void);           // esp_system.h
uint32_t esp_get_minimum_free_heap_size(void);   // watermark ต่ำสุดตั้งแต่บูต
```

## กับดัก Hallucination
1. `malloc()` ปกติ **ไม่แตะ PSRAM** จนกว่าจะเปิด `CONFIG_SPIRAM_USE_MALLOC` (+ threshold) — งานต้องการ PSRAM ให้ระบุ `MALLOC_CAP_SPIRAM` ตรง ๆ
2. Buffer ส่งให้ SPI/I2S/RMT DMA **ต้อง `MALLOC_CAP_DMA`** — ตัวแปร local/static ธรรมดามักอยู่ DRAM ภายในจึงรอด แต่ **const array / string literal อยู่ใน flash (DROM) → DMA อ่านไม่ได้**
3. `MALLOC_CAP_DMA` + `MALLOC_CAP_SPIRAM` พร้อมกัน → S3 จองไม่ได้ (DMA cap = ภายใน) ⚠️ GDMA ของ S3 เข้าถึง PSRAM ได้บางกรณีแต่มีเงื่อนไข align/cache — ตรวจ docs ก่อน
4. Fragmentation: เช็ค `heap_caps_get_largest_free_block()` ไม่ใช่แค่ `get_free_size()` — เหลือ 100 KB แต่ block ใหญ่สุด 3 KB ก็จอง 8 KB ไม่ได้
5. อย่า cast ค่าที่ `heap_caps_malloc` คืน — เช็ค `== NULL` ทุกครั้ง (RAM มีจำกัดจริง)
6. ใช้ PSRAM แล้วช้าผิดปกติ → ย้าย buffer ร้อน (callback/DSP) กลับ DRAM ภายใน
7. Task stack อยู่ DRAM เสมอ เว้นแต่เปิด `CONFIG_SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY` + `xTaskCreateStatic`
8. `esp_get_minimum_free_heap_size()` = ตัวชี้วัดว่าเคยใกล้หมดแค่ไหน (หา leak ระยะยาว)

## ตัวอย่างมินิมัล
```c
#include "esp_heap_caps.h"

uint8_t *dma_buf = heap_caps_malloc(4096, MALLOC_CAP_DMA);   // buffer สำหรับ SPI/I2S
if (!dma_buf) { ESP_LOGE("heap", "no DMA mem"); return; }

ESP_LOGI("heap", "internal free=%u largest=%u",
         (unsigned)heap_caps_get_free_size(MALLOC_CAP_INTERNAL),
         (unsigned)heap_caps_get_largest_free_block(MALLOC_CAP_INTERNAL));
free(dma_buf);
```