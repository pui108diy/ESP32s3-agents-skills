# GPIO Driver — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "driver/gpio.h"   // ครอบคลุม gpio_num_t และทุก API ด้านล่าง
```

## Hardware Facts (จาก TRM ESP32-S3)
- GPIO ใช้ได้จริง: **0–21 และ 26–48** (รวม 45 pin) ต่อผ่าน GPIO Matrix
- **GPIO 26–32**: ต่อกับ SPI flash ภายใน — ห้ามใช้เด็ดขาด
- **GPIO 33–37**: ถูกใช้เมื่อบอร์ดเปิด Octal PSRAM
- Strapping pins (ระวัง pull ตอนบูต): **GPIO0, GPIO3 (JTAG source), GPIO45 (VDD_SPI), GPIO46 (boot mode)**
- USB-Serial-JTAG: **GPIO19 (D-), GPIO20 (D+)**

## Structs
```c
typedef struct {
    uint64_t pin_bit_mask;        // หลาย pin: (1ULL<<4) | (1ULL<<5)
    gpio_mode_t mode;
    gpio_pullup_t pull_up_en;     // GPIO_PULLUP_ENABLE / DISABLE
    gpio_pulldown_t pull_down_en; // GPIO_PULLDOWN_ENABLE / DISABLE
    gpio_int_type_t intr_type;    // GPIO_INTR_DISABLE/POSEDGE/NEGEDGE/ANYEDGE/LOW_LEVEL/HIGH_LEVEL
} gpio_config_t;
```

## Function Signatures (exact)
```c
esp_err_t gpio_config(const gpio_config_t *pGPIOConfig);
esp_err_t gpio_reset_pin(gpio_num_t gpio_num);            // ← แทน gpio_pad_select_gpio (deprecated v5.0, removed v6)
esp_err_t gpio_set_level(gpio_num_t gpio_num, uint32_t level);
int       gpio_get_level(gpio_num_t gpio_num);
esp_err_t gpio_set_direction(gpio_num_t gpio_num, gpio_mode_t mode);
esp_err_t gpio_set_pull_mode(gpio_num_t gpio_num, gpio_pull_mode_t pull);
esp_err_t gpio_set_intr_type(gpio_num_t gpio_num, gpio_int_type_t intr_type);
esp_err_t gpio_install_isr_service(int intr_alloc_flags); // ใช้ ESP_INTR_FLAG_IRAM ถ้า ISR อยู่ใน IRAM
esp_err_t gpio_isr_handler_add(gpio_num_t gpio_num, gpio_isr_t isr_handler, void *args);
esp_err_t gpio_isr_handler_remove(gpio_num_t gpio_num);
esp_err_t gpio_intr_enable(gpio_num_t gpio_num);
esp_err_t gpio_intr_disable(gpio_num_t gpio_num);
esp_err_t gpio_wakeup_enable(gpio_num_t gpio_num, gpio_int_type_t intr_type); // light/deep sleep wake
esp_err_t gpio_hold_en(gpio_num_t gpio_num);              // ค้างค่า pin ระหว่าง deep sleep
typedef void (*gpio_isr_t)(void *arg);                    // ← รับ void* ตัวเดียวเท่านั้น!
```

## กับดัก Hallucination
1. ISR ประกาศเป็น `void isr(int gpio_num)` → ผิด ต้องเป็น `void isr(void *arg)` (ส่ง pin ผ่าน `args`)
2. ลืม `gpio_install_isr_service()` ก่อน `gpio_isr_handler_add()` → `ESP_ERR_INVALID_STATE`
3. `1UL << 31` ไม่พอสำหรับ pin_bit_mask → ต้อง `1ULL`
4. `gpio_config_t` ใช้ `gpio_pullup_t` แต่ `gpio_set_pull_mode()` ใช้ `gpio_pull_mode_t` — enum คนละชุด
5. ห้าม `printf`/`ESP_LOGx` ใน ISR

## ตัวอย่างมินิมัล
```c
#include "driver/gpio.h"
#define BTN GPIO_NUM_4

static IRAM_ATTR void btn_isr(void *arg) { /* set flag / xQueueSendFromISR เท่านั้น */ }

void app_main(void)
{
    gpio_config_t io = {
        .pin_bit_mask = 1ULL << BTN,
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .intr_type = GPIO_INTR_NEGEDGE,
    };
    ESP_ERROR_CHECK(gpio_config(&io));
    ESP_ERROR_CHECK(gpio_install_isr_service(ESP_INTR_FLAG_IRAM));
    ESP_ERROR_CHECK(gpio_isr_handler_add(BTN, btn_isr, NULL));
}
```