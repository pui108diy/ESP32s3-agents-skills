# Heap / Memory Allocation API — v1.1
> ESP-IDF v5.x/v6.x · ESP32-S3 · **สถานะ: Phase 1 แปลงโครงสร้าง §0–§10 แล้ว — ยังไม่ได้ตรวจเชิงลึกกับ header จริง (ดู §10)**
> เนื้อหาทั้งหมดย้ายจากไฟล์เดิม ไม่มีการเติมข้อมูลเทคนิคใหม่ที่ยังไม่ผ่านการตรวจ
> เสริม skill `esp32s3-memory-map` (ที่อยู่ regions) — ไฟล์นี้คือ **API จัดสรร**

## §0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. Buffer ส่งให้ SPI/I2S/RMT/UART DMA **ต้อง `MALLOC_CAP_DMA`** — ตัวแปร local/static ธรรมดามักรอด แต่ **const array / string literal อยู่ใน flash (DROM) → DMA อ่านไม่ได้**
2. `malloc()` ปกติ **ไม่แตะ PSRAM** จนกว่าจะเปิด `CONFIG_SPIRAM_USE_MALLOC` — งานต้องการ PSRAM ระบุ `MALLOC_CAP_SPIRAM` ตรง ๆ
3. `MALLOC_CAP_DMA` + `MALLOC_CAP_SPIRAM` พร้อมกัน → S3 จองไม่ได้ ⚠️ (GDMA ของ S3 เข้าถึง PSRAM ได้บางกรณีแต่มีเงื่อนไข — ดู §10)
4. Fragmentation วัดด้วย `heap_caps_get_largest_free_block()` — เหลือ 100 KB แต่ block ใหญ่สุด 3 KB ก็จอง 8 KB ไม่ได้
5. เช็ค `== NULL` ทุกครั้งหลังจอง (RAM มีจำกัดจริง) — อย่า cast ค่าที่ `heap_caps_malloc` คืน
6. PSRAM ช้ากว่า DRAM ภายใน — buffer ร้อน (callback/DSP) ย้ายกลับ internal
7. Task stack อยู่ DRAM เสมอ เว้นแต่เปิด `CONFIG_SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY` + `xTaskCreateStatic`
8. `esp_get_minimum_free_heap_size()` = watermark ต่ำสุดตั้งแต่บูต — ตัวชี้วัดหา leak ระยะยาว
9. `heap_caps_free()` = `free()` ใช้แทนกันได้ (heap เดียวทั้งระบบ)
10. ไฟล์นี้ = **API จัดสรร** — ที่อยู่ regions ดู skill `esp32s3-memory-map`

## §1. Include & CMakeLists
```c
#include "esp_heap_caps.h"   // heap_caps_*
#include "esp_system.h"      // esp_get_free_heap_size() ฯลฯ
```
- **CMakeLists:** ⚠️ ยังไม่ได้ตรวจ component ที่ต้องระบุใน `REQUIRES` — อยู่ในคิวตรวจ §10

## §2. Capability Flags บน S3
| Flag | ความหมาย | อยู่ที่ |
|---|---|---|
| `MALLOC_CAP_8BIT` | เขียนอ่านราย byte ได้ | DRAM ภายใน + PSRAM |
| `MALLOC_CAP_DMA` | **DMA เข้าถึงได้** (SPI/I2S/RMT/UART) | DRAM ภายในเท่านั้น |
| `MALLOC_CAP_INTERNAL` | ภายใน chip | DRAM/IRAM |
| `MALLOC_CAP_SPIRAM` | PSRAM ภายนอก | PSRAM |
| `MALLOC_CAP_EXEC` | รันโค้ดได้ | IRAM |
| `MALLOC_CAP_DEFAULT` | ตามพฤติของ `malloc()` | ขึ้นกับ Kconfig SPIRAM |

## §3. Allocation API
```c
void  *heap_caps_malloc(size_t size, uint32_t caps);
void  *heap_caps_calloc(size_t n, size_t size, uint32_t caps);
void  *heap_caps_realloc(void *ptr, size_t size, uint32_t caps);
void  *heap_caps_aligned_alloc(size_t alignment, size_t size, uint32_t caps); // v5.0+, alignment = 2^n
void   heap_caps_aligned_free(void *ptr);
void   heap_caps_free(void *ptr);            // free() ใช้แทนได้ (heap เดียวทั้งระบบ)
```

## §4. Diagnostic API
```c
size_t heap_caps_get_free_size(uint32_t caps);
size_t heap_caps_get_largest_free_block(uint32_t caps);   // ← วัด fragmentation จริง
void   heap_caps_print_heap_info(uint32_t caps);
void   heap_caps_print_all_info(void);
bool   heap_caps_check_integrity_all(bool print_errors);
esp_err_t heap_caps_malloc_extmem_enable(size_t threshold); // malloc() เริ่มใช้ PSRAM เมื่อ > threshold

uint32_t esp_get_free_heap_size(void);           // esp_system.h
uint32_t esp_get_minimum_free_heap_size(void);   // watermark ต่ำสุดตั้งแต่บูต
```

## §5. ความสัมพันธ์กับ memory-map + นโยบาย PSRAM
- ไฟล์นี้คือ **API จัดสรร** — แผนที่ regions (DRAM/IRAM/DROM/PSRAM) อยู่ที่ skill `esp32s3-memory-map`
- นโยบายจากกับดัก: `malloc()` ไม่แตะ PSRAM จนกว่าเปิด `CONFIG_SPIRAM_USE_MALLOC` (+ threshold) / buffer ร้อนควรอยู่ DRAM ภายใน / task stack อยู่ DRAM เสมอ (เว้นแต่ config พิเศษ)

## §6. VFS
- ไม่มี — heap API ไม่มีการผูก VFS (N/A สำหรับไฟล์นี้)

## §7. กับดักที่พบบ่อย (⚠️ รอผูกรหัส HALLUCINATION-HOT-LIST ในรอบถัดไป)
1. `malloc()` ปกติ **ไม่แตะ PSRAM** จนกว่าจะเปิด `CONFIG_SPIRAM_USE_MALLOC` (+ threshold) — งานต้องการ PSRAM ให้ระบุ `MALLOC_CAP_SPIRAM` ตรง ๆ
2. Buffer ส่งให้ SPI/I2S/RMT DMA **ต้อง `MALLOC_CAP_DMA`** — ตัวแปร local/static ธรรมดามักอยู่ DRAM ภายในจึงรอด แต่ **const array / string literal อยู่ใน flash (DROM) → DMA อ่านไม่ได้**
3. `MALLOC_CAP_DMA` + `MALLOC_CAP_SPIRAM` พร้อมกัน → S3 จองไม่ได้ (DMA cap = ภายใน) ⚠️ GDMA ของ S3 เข้าถึง PSRAM ได้บางกรณีแต่มีเงื่อนไข align/cache — ตรวจ docs ก่อน
4. Fragmentation: เช็ค `heap_caps_get_largest_free_block()` ไม่ใช่แค่ `get_free_size()` — เหลือ 100 KB แต่ block ใหญ่สุด 3 KB ก็จอง 8 KB ไม่ได้
5. อย่า cast ค่าที่ `heap_caps_malloc` คืน — เช็ค `== NULL` ทุกครั้ง (RAM มีจำกัดจริง)
6. ใช้ PSRAM แล้วช้าผิดปกติ → ย้าย buffer ร้อน (callback/DSP) กลับ DRAM ภายใน
7. Task stack อยู่ DRAM เสมอ เว้นแต่เปิด `CONFIG_SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY` + `xTaskCreateStatic`
8. `esp_get_minimum_free_heap_size()` = ตัวชี้วัดว่าเคยใกล้หมดแค่ไหน (หา leak ระยะยาว)

## §8. ตัวอย่างมินิมัล
```c
#include "esp_heap_caps.h"

uint8_t *dma_buf = heap_caps_malloc(4096, MALLOC_CAP_DMA);   // buffer สำหรับ SPI/I2S
if (!dma_buf) { ESP_LOGE("heap", "no DMA mem"); return; }

ESP_LOGI("heap", "internal free=%u largest=%u",
         (unsigned)heap_caps_get_free_size(MALLOC_CAP_INTERNAL),
         (unsigned)heap_caps_get_largest_free_block(MALLOC_CAP_INTERNAL));
free(dma_buf);
```

## §9. v6.x Notes
- ไฟล์เดิมไม่มี version marker — การค้น migration guide (ฐานความรู้) เบื้องต้นพบประเด็นเกี่ยวข้อง: `esp_dma_capable_malloc`/`esp_dma_capable_calloc` ถูกลบใน v6 (ดู §10 ข้อ 2 — ยังไม่ verify)

## §10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
> header เป้าหมาย (tag v6.1): `components/heap/include/esp_heap_caps.h` + `esp_system.h` (path ต้องยืนยัน)
1. CMakeLists `REQUIRES` — heap / esp_system
2. ⚠️ จาก migration guide v6 (ฐานความรู้ migration 5.x→6.x): `esp_dma_capable_malloc` และ `esp_dma_capable_calloc` **ถูกลบใน v6** → ใช้ `heap_caps_malloc`/`heap_caps_calloc` กับ `MALLOC_CAP_DMA|MALLOC_CAP_CACHE_ALIGNED` — ยืนยันกับ header v6.1 แล้วเขียนลง §9 (หมายเหตุ: คู่ฟังก์ชันนี้ไม่ได้อยู่ในไฟล์เดิม แต่เกี่ยวข้องกับหัวข้อ DMA allocation โดยตรง)
3. GDMA บน S3 เข้าถึง PSRAM ได้จริงไหม ด้วยเงื่อนไขอะไร (align/cache) — ไฟล์เดิม mark ⚠️ ไว้
4. `heap_caps_aligned_alloc` — ยืนยัน marker v5.0+ + ข้อกำหนด alignment = 2^n
5. ชุด `MALLOC_CAP_*` flags ครบไหม vs header v6.1 (มี `MALLOC_CAP_CACHE_ALIGNED` ฯลฯ เพิ่มไหม)
6. `MALLOC_CAP_DEFAULT` พฤติกรรมตาม Kconfig SPIRAM — ยืนยัน
7. `esp_get_free_heap_size()` คืน `uint32_t` — ยืนยัน + มี API 64-bit เพิ่มไหม
8. `CONFIG_SPIRAM_USE_MALLOC` + threshold semantics — ยืนยันชื่อและพฤติกรรม
9. ผูกรหัส HALLUCINATION-HOT-LIST ให้ §7 (ทำพร้อมกับอัปเดต Hot-List)