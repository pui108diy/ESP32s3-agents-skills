# SD Card (SDMMC/SDSPI + FATFS ผ่าน VFS) — ESP-IDF v5.x/v6.x (ESP32-S3)

## Includes
```c
#include "esp_vfs_fat.h"         // mount/unmount + VFS
#include "sdmmc_cmd.h"           // sdmmc_card_t
#include "driver/sdmmc_host.h"   // โหมด SDMMC (slot 1)
// โหมด SPI: #include "driver/sdspi_host.h"
```

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 34 + POM)
- SDHOST รองรับ **การ์ดภายนอก 2 ใบ** ผ่าน GPIO matrix (สัญญาณทุกเส้น route ผ่าน GPIO matrix)
- มาตรฐาน: SD v3.0/3.01, SDIO v3.0, MMC 4.41/4.5/4.51, CE-ATA 1.1
- ความกว้าง **1/4/8-bit**; clock **สูงสุด 80 MHz** แต่ต้อง phase 0°/180° เท่านั้น + PCB layout ดี (TRM Note) — ใช้งานทั่วไป: **20 MHz (default) / 40 MHz (high-speed)**
- ใช้ DMA ภายใน (BIU + CIU architecture)
- ต้องมี **pull-up บนเส้น CMD/DATA** (บอร์ดบางรุ่นมีให้, บางรุ่นต้องใส่เอง — ไม่มี pull-up = การ์ดเจอไม่ได้)
- ⚠️ IDF `SDMMC_HOST_DEFAULT()` ใช้ **Slot 1** — บอร์ด S3 ทั่วไปต่อที่ Slot 1 (การใช้ Slot 0 บน S3 ตรวจ sdmmc_host.h ของเวอร์ชันที่ใช้)

## Pattern มาตรฐาน (all-in-one mount)
```c
// 1) host + slot + mount config → 2) esp_vfs_fat_sdcard_mount() → 3) ใช้ fopen/fwrite ปกติ → 4) unmount
```

## Structs (มาตรฐานจาก example ทางการ)
```c
sdmmc_host_t host = SDMMC_HOST_DEFAULT();          // slot 1, 1|4|8-bit flags, 20 MHz
host.flags = SDMMC_HOST_FLAG_4BIT;                 // เลือก 4-bit (default macro เปิดทุก width ไว้)

sdmmc_slot_config_t slot_config = SDMMC_SLOT_CONFIG_DEFAULT();  // CMD/CLK/D0–D3 ผ่าน GPIO matrix

esp_vfs_fat_mount_config_t mount_config = {
    .format_if_mount_failed = false,   // true = ฟอร์แมตเมื่อ mount fail (ข้อมูลหาย!)
    .max_files = 5,                    // จำนวนไฟล์เปิดพร้อมกันสูงสุด — น้อยไป = fopen fail
    .allocation_unit_size = 16 * 1024,
};
```

## Function Signatures (exact)
```c
// --- Mount (all-in-one: init driver + init card + mount FAT + register VFS) ---
esp_err_t esp_vfs_fat_sdcard_mount(const char *base_path, const sdmmc_host_t *host_config,
                                   const void *slot_config,                                    // sdmmc หรือ sdspi slot config
                                   const esp_vfs_fat_mount_config_t *mount_config,
                                   sdmmc_card_t **out_card);                                   // v5.1+ ← ใช้ตัวนี้
esp_err_t esp_vfs_fat_sdmmc_mount(const char *base_path, const sdmmc_host_t *host_config,
                                  const sdmmc_slot_config_t *slot_config,
                                  const esp_vfs_fat_mount_config_t *mount_config,
                                  sdmmc_card_t **out_card);                                    // ชื่อเดิม (ยังมี)
esp_err_t esp_vfs_fat_sdspi_mount(const char *base_path, const sdmmc_host_t *host_config,
                                  const sdspi_device_config_t *slot_config,                    // โหมด SPI!
                                  const esp_vfs_fat_mount_config_t *mount_config,
                                  sdmmc_card_t **out_card);
esp_err_t esp_vfs_fat_sdcard_unmount(const char *base_path, sdmmc_card_t *card);
esp_err_t esp_vfs_fat_sdcard_format(const char *base_path, sdmmc_card_t *card);

// --- หลัง mount ใช้ POSIX ปกติ (ผ่าน VFS) ---
FILE *f = fopen("/sdcard/data.txt", "w");
fwrite(...); fread(...); fclose(f);
// + mkdir/opendir/readdir/stat รวมทั้ง rename/remove ใช้ได้

// --- โหมด SPI: config ต่างจาก SDMMC ---
// sdmmc_host_t host = SDSPI_HOST_DEFAULT();      // .slot = SPI2_HOST
// sdspi_device_config_t slot = { .host_id = SPI2_HOST, .gpio_cs = 10, ... };
```

## กับดัก Hallucination
1. `fopen("/sdcard/...")` ก่อน mount (หรือหลัง unmount) → ENOENT — mount ก่อนเสมอ
2. `max_files` น้อยเกินไป (default ตัวอย่าง = 5) → เปิดไฟล์ที่ 6 fail ทั้งที่ heap ยังเหลือ
3. **โหมด SPI ใช้ `sdmmc_slot_config_t`** → ผิด — ต้อง `sdspi_device_config_t` + `esp_vfs_fat_sdspi_mount`/`esp_vfs_fat_sdcard_mount`
4. ลืมตั้ง `host.flags = SDMMC_HOST_FLAG_4BIT` → วิ่งแค่ 1-bit เงียบ ๆ (ช้ากว่า 4 เท่า ไม่มี error)
5. `format_if_mount_failed = true` ใน production → **ข้อมูลผู้ใช้หาย** — ใช้เฉพาะโหมด setup
6. เขียนแล้วดึงการ์ดทันที → ข้อมูลค้างใน cache — เรียก `fclose()` (และถ้า paranoid ใช้ `fflush`+`fsync` ⚠️ ตรวจ VFS support)
7. การ์ดไม่เจอ: เช็ค pull-up CMD/DATA, เช็คว่าเส้บเส้นไหน (Slot 1 ผ่าน GPIO matrix กำหนด pin ได้), เช็คไฟเลี้ยง 3.3 V
8. อยากได้ความเร็ว 40 MHz → SDMMC_FREQ_HIGHSPEED (การ์ดต้องรองรับ HS); 80 MHz = ตามเงื่อนไข TRM เท่านั้น
9. `esp_vfs_fat_sdcard_unmount` ต้องใช้ `card` handle เดิมที่ได้จาก mount — เก็บไว้ให้ดี

## ตัวอย่างมินิมัล (SDMMC 4-bit)
```c
#include "esp_vfs_fat.h"
#include "sdmmc_cmd.h"
#include "driver/sdmmc_host.h"

void sd_init(void)
{
    sdmmc_host_t host = SDMMC_HOST_DEFAULT();
    host.flags = SDMMC_HOST_FLAG_4BIT;                     // ใช้ 4 เส้น

    sdmmc_slot_config_t slot = SDMMC_SLOT_CONFIG_DEFAULT();
    esp_vfs_fat_mount_config_t mount = {
        .format_if_mount_failed = false,
        .max_files = 5,
        .allocation_unit_size = 16 * 1024,
    };
    sdmmc_card_t *card = NULL;
    ESP_ERROR_CHECK(esp_vfs_fat_sdcard_mount("/sdcard", &host, &slot, &mount, &card));

    FILE *f = fopen("/sdcard/hello.txt", "w");             // ← ต้อง mount ก่อนเสมอ
    if (f) { fprintf(f, "hello\n"); fclose(f); }

    // จบงาน: esp_vfs_fat_sdcard_unmount("/sdcard", card);
}
```