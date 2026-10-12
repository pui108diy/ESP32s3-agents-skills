# LEDC (LED PWM Controller) — v2.1
> Target: ESP32-S3 | ESP-IDF v5.2 – v6.1
> อ้างอิง: TRM v1.8 ch.35 (LEDC) + ch.7 (Clock) | Datasheet v2.2 | ledc.h v6.1 (header จริง) | soc_caps.h v6.1 | Migration Guide v6.1
> เปลี่ยนแปลง v2.1: แปลงโครงสร้างตาม skill v2.7 — เพิ่ม §0 TL;DR · แยก §5 Duty Workflow + §6 Fade & Callback (กับดักย้าย §5→§7) · เพิ่ม §10 จุดตรวจซ้ำ · กู้ตัวอย่างมินิมัลเต็ม

## 0. TL;DR — ถ้าจำได้แค่ 10 ข้อ
1. **8 channels / 4 timers / 14-bit สูงสุด** — S3 มี `LEDC_LOW_SPEED_MODE` เท่านั้น
2. **S3 ไม่มี DAC** — งาน analog out ใช้ LEDC แทน
3. Clock source เป็น **global ของทุก timer** — ห้ามผสม APB+XTAL ข้าม timer
4. `ledc_set_duty()` เฉย ๆ ไม่มีผล — ต้อง `ledc_update_duty()` หรือใช้ `ledc_set_duty_and_update()`
5. ที่ resolution สูงสุด (14-bit) **ห้ามตั้ง duty = 2^res (16384)** — ใช้สูงสุด 2^res − 1
6. `hpoint` = จุดที่สัญญาณ**ขึ้น high** (เฟสพัลส์) — Lpoint = Hpoint + duty
7. `ledc_set_fade_with_time()` มี **4 พารามิเตอร์** — fade_mode เป็นของ `ledc_fade_start()`
8. ต้อง `ledc_fade_func_install()` ก่อนใช้ fade; S3 หยุดกลางคันได้ด้วย `ledc_fade_stop()`
9. `ledc_cbs_t` มี callback **เดียว `.fade_cb`** (return bool)
10. ตัวเลข f_max ในตาราง = ค่าที่ **CLK_DIV = 1** (ขีดจำกัดบนทางทฤษฎี); v6 ลบ `ledc_timer_set` + macro clock เปลี่ยน

## 1. Include & CMakeLists

```c
#include "driver/ledc.h"   // path ชื่อเดิมทุกเวอร์ชัน v5.2 – v6.1
```

```cmake
# v5.2:        REQUIRES driver
# v5.3+ / v6.x: แยก component — ระบุตรง:
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_driver_ledc)
```
⚠️ v6.x: `esp_driver_gpio` ถูกถอดออกจาก dependency สาธารณะของ `esp_driver_ledc` — โค้ดที่ใช้ GPIO API ร่วมด้วยต้อง `REQUIRES esp_driver_ledc esp_driver_gpio`

## 2. Hardware Facts (Datasheet v2.2, TRM ch.35)
- **8 channels** (`LEDC_CHANNEL_0..7`), **4 timers** (`LEDC_TIMER_0..3`) — channel ผูก timer ผ่าน `timer_sel` / `ledc_bind_channel_timer()`
- **S3 มี `LEDC_LOW_SPEED_MODE` เท่านั้น** — `LEDC_HIGH_SPEED_MODE` compile error
- **S3 ไม่มี DAC ในตัว** — `dac_output_enable()` ไม่มี; ใช้ LEDC ทดแทน
- Duty resolution สูงสุด **14 bit** (`LEDC_TIMER_1_BIT`..`LEDC_TIMER_14_BIT`)
- **Clock sources — global ทุก timer บน S3:** APB_CLK 80 MHz / XTAL_CLK 40 MHz / RC_FAST_CLK ~17.5 MHz ผ่าน `LEDC_APB_CLK_SEL` ตัวเดียว (TRM Register 35.3) — per-timer clock มีเฉพาะ ESP32/S2
- **Fractional divider:** จำนวนเต็ม 10 bit + เศษส่วน 8 bit (สูงสุด 1023 + 255/256)
- **สูตร:** f_PWM = f_CLK / (CLK_DIV × 2^DUTY_RES)
- **Hardware fade CPU-independent:** พารามิเตอร์ fade (scale/num/cycle) กว้าง 10 bit ต่อ field (TRM CONF1)
- **`ledc_fade_stop()` หยุด fade กลางคันได้บน S3** (`SOC_LEDC_SUPPORT_FADE_STOP=1` — ต่างจาก ESP32/C3)
- RC_FAST ยังเดินเมื่อ APB ปิดใน low-power mode (TRM ch.7)

**ตาราง f_max @ APB 80 MHz — ค่าที่ LEDC_CLK_DIV = 1 (ขีดจำกัดบนทางทฤษฎี)**
| duty_resolution | f_max @ 80 MHz |
|---|---|
| 14 bit | ~4.88 kHz |
| 12 bit | ~19.5 kHz |
| 10 bit | ~78 kHz |
| 8 bit | ~312 kHz |

**Hpoint — เฟสเริ่มต้นของพัลส์**
- TRM: `Timerx_cnt == Hpoint` → high, `Timerx_cnt == Lpoint` → low, **Lpoint = Hpoint + duty** → ปรับ hpoint = เลื่อนเฟสพัลส์ (กระจายพัลส์หลาย channel ไม่ให้ขึ้นพร้อมกัน)
- `LEDC_HPOINT_CHn` กว้าง 14 bit บน S3 → ช่วง [0, 16383]

## 3. Structs (ledc.h v6.1 จริง)
```c
typedef struct {
    ledc_mode_t speed_mode;            // S3: LEDC_LOW_SPEED_MODE only
    ledc_timer_bit_t duty_resolution;  // LEDC_TIMER_1_BIT .. LEDC_TIMER_14_BIT (enum ไม่ใช่ int)
    ledc_timer_t timer_num;            // LEDC_TIMER_0..3
    uint32_t freq_hz;
    ledc_clk_cfg_t clk_cfg;            // LEDC_AUTO_CLK / LEDC_USE_APB_CLK / LEDC_USE_XTAL_CLK / LEDC_USE_RC_FAST_CLK
    bool deconfigure;                  // true = deconfigure timer นี้
} ledc_timer_config_t;

typedef struct {
    int gpio_num;
    ledc_mode_t speed_mode;
    ledc_channel_t channel;            // LEDC_CHANNEL_0..7
    ledc_intr_type_t intr_type;        // ⚠️ v6.x: deprecated — ใช้ ledc_cb_register() แทน
    ledc_timer_t timer_sel;
    uint32_t duty;                     // [0, 2**duty_resolution] — ระวังข้อ 5 ของ §0
    int hpoint;                        // เฟสเริ่มพัลส์ high — [0, 16383] บน S3
    ledc_sleep_mode_t sleep_mode;      // ⚠️ ยืนยันใน v6.1 — เช็ค header เวอร์ชันเก่าก่อน (§10)
    struct {
        unsigned int output_invert: 1;
    } flags;
} ledc_channel_config_t;

typedef bool (*ledc_cb_t)(const ledc_cb_param_t *param, void *user_arg);
typedef struct {
    ledc_cb_t fade_cb;                 // callback เดียว: fade จบ (param->event == LEDC_FADE_END_EVT)
} ledc_cbs_t;
```

## 4. Function Signatures (ledc.h v6.1)
```c
// Timer
esp_err_t ledc_timer_config(const ledc_timer_config_t *timer_conf);
esp_err_t ledc_timer_rst(ledc_mode_t speed_mode, ledc_timer_t timer_num);
esp_err_t ledc_timer_pause(ledc_mode_t speed_mode, ledc_timer_t timer_num);
esp_err_t ledc_timer_resume(ledc_mode_t speed_mode, ledc_timer_t timer_num);
esp_err_t ledc_set_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num, uint32_t freq_hz);
uint32_t ledc_get_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num);
uint32_t ledc_find_suitable_duty_resolution(uint32_t src_clk_freq, uint32_t timer_freq);
//   ⚠️ ledc_timer_set() ถูกลบใน v6.x → ใช้ ledc_timer_config / ledc_set_freq

// Channel
esp_err_t ledc_channel_config(const ledc_channel_config_t *ledc_conf);
esp_err_t ledc_set_pin(int gpio_num, ledc_mode_t speed_mode, ledc_channel_t channel);
esp_err_t ledc_bind_channel_timer(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_timer_t timer_sel);
esp_err_t ledc_stop(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t idle_level);

// Duty — ดู workflow ใน §5
esp_err_t ledc_set_duty(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty);
esp_err_t ledc_update_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
esp_err_t ledc_set_duty_and_update(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, uint32_t hpoint);
esp_err_t ledc_set_duty_with_hpoint(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, uint32_t hpoint);
uint32_t ledc_get_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
uint32_t ledc_get_max_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
int ledc_get_hpoint(ledc_mode_t speed_mode, ledc_channel_t channel);

// Fade — ดู workflow ใน §6
esp_err_t ledc_fade_func_install(int intr_alloc_flags);
esp_err_t ledc_fade_func_uninstall(void);
esp_err_t ledc_set_fade_with_time(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, int desired_fade_time_ms);   // 4 พารามิเตอร์!
esp_err_t ledc_set_fade_with_step(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t target_duty, uint32_t scale, uint32_t cycle_num);
esp_err_t ledc_set_fade(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, ledc_duty_direction_t fade_direction, uint32_t step_num, uint32_t duty_cycle_num, uint32_t duty_scale);  // 7 พารามิเตอร์
esp_err_t ledc_fade_start(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_fade_mode_t fade_mode);  // LEDC_FADE_NO_WAIT / LEDC_FADE_WAIT_DONE
esp_err_t ledc_fade_stop(ledc_mode_t speed_mode, ledc_channel_t channel);

// Callback
esp_err_t ledc_cb_register(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_cbs_t *cbs, void *user_arg);
//   ⚠️ ledc_isr_register() deprecated ใน v6.x
```

## 5. Duty Workflow
- duty เป็น **raw integer ไม่ใช่ %** — 13-bit เต็ม = 8191, 14-bit เต็ม = 16383
- **ที่ resolution สูงสุด (14-bit) ห้ามตั้ง duty = 2^res (16384)** — ฮาร์ดแวร์ทำงานผิดพลาด; ใช้สูงสุด 2^res − 1
- เปลี่ยน duty 2 แบบ:
  - `ledc_set_duty()` **แล้วต้อง** `ledc_update_duty()` ถึงมีผล
  - `ledc_set_duty_and_update(mode, ch, duty, hpoint)` — ทีเดียวจบ (atomic)
- พร้อมเฟส: `ledc_set_duty_with_hpoint()`; อ่านค่า: `ledc_get_duty()` / `ledc_get_max_duty()` / `ledc_get_hpoint()`
- เลือก resolution อัตโนมัติ: `ledc_find_suitable_duty_resolution(src_clk_freq, timer_freq)`
- เปลี่ยน freq/divider ของ timer → กระทบ**ทุก channel ที่ bind** อยู่กับ timer นั้น

## 6. Fade Workflow & Callbacks
```c
ledc_fade_func_install(0);   // ครั้งเดียวต่อระบบ ก่อนใช้ fade (flags = ESP_INTR_FLAG_* หรือ 0)

ledc_set_fade_with_time(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 8191, 2000);   // → duty 8191 ใน 2 s
ledc_fade_start(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, LEDC_FADE_WAIT_DONE); // block รอจบ
// หรือ LEDC_FADE_NO_WAIT แล้วรอผ่าน callback
ledc_fade_stop(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);   // หยุดกลางคันได้ (S3)
```
- Callback (fade จบ):
```c
static bool fade_done_cb(const ledc_cb_param_t *param, void *user_arg)
{
    if (param->event == LEDC_FADE_END_EVT) { /* สั้น ๆ — รันใน ISR context */ }
    return false;   // ต้อง return bool
}
ledc_cbs_t cbs = { .fade_cb = fade_done_cb };
ledc_cb_register(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, &cbs, NULL);
```
- เปิด `CONFIG_LEDC_CTRL_FUNC_IN_IRAM` → callback + ฟังก์ชันที่มันเรียกต้องอยู่ใน IRAM (`IRAM_ATTR`)

## 7. กับดักที่พบบ่อย (ผูก HALLUCINATION-HOT-LIST v3.1)
1. `LEDC_HIGH_SPEED_MODE` → compile error → **LEDC-01**
2. `ledc_set_duty()` เฉย ๆ → ต้อง `ledc_update_duty()` → **LEDC-02**
3. duty เป็น raw integer ไม่ใช่ %
4. duty = 2^resolution ที่ 14-bit → ฮาร์ดแวร์พัง → **LEDC-06**
5. ไม่ `ledc_fade_func_install()` → fade ไม่ทำงาน
6. `duty_resolution` ต้องใช้ enum `LEDC_TIMER_x_BIT` ไม่ใช่ int
7. เปลี่ยน freq ของ timer → กระทบทุก channel ที่ bind
8. จำกัด 8 channels / 4 timers
9. `ledc_set_fade_with_time()` มี 4 พารามิเตอร์ (fade_mode อยู่ที่ `ledc_fade_start`) → **LEDC-08**
10. `ledc_cbs_t` มี callback เดียว `.fade_cb` → **LEDC-09**
11. clock source เป็น global ของทุก timer → **LEDC-05**
12. v6.x removals: `ledc_timer_set` ลบ / `ledc_isr_register` + `intr_type` deprecated / `LEDC_APB_CLK_HZ` ลบ → **LEDC-10**
13. `periph_module_reset(PERIPH_LEDC_MODULE)` ใช้ไม่ได้ v6.x — clock gating/reset อยู่ใน driver layer แล้ว

## 8. ตัวอย่างมินิมัล
```c
#include "driver/ledc.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

void app_main(void)
{
    ledc_timer_config_t timer = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .timer_num = LEDC_TIMER_0,
        .duty_resolution = LEDC_TIMER_13_BIT,
        .freq_hz = 5000,
        .clk_cfg = LEDC_AUTO_CLK,
    };
    ESP_ERROR_CHECK(ledc_timer_config(&timer));

    ledc_channel_config_t ch = {
        .gpio_num = 4,
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel = LEDC_CHANNEL_0,
        .timer_sel = LEDC_TIMER_0,
        .duty = 0,
        .hpoint = 0,
    };
    ESP_ERROR_CHECK(ledc_channel_config(&ch));
    ESP_ERROR_CHECK(ledc_fade_func_install(0));

    while (1) {   // หายใจ LED: 0 → 8191 → 0 ใน 2 s ต่อทิศ
        ESP_ERROR_CHECK(ledc_set_fade_with_time(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 8191, 2000));  // 4 พารามิเตอร์
        ESP_ERROR_CHECK(ledc_fade_start(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, LEDC_FADE_WAIT_DONE)); // block รอจบ
        ESP_ERROR_CHECK(ledc_set_fade_with_time(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0, 2000));
        ESP_ERROR_CHECK(ledc_fade_start(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, LEDC_FADE_WAIT_DONE));
    }
}
```

## 9. v6.x Notes
- `ledc_timer_set()` ลบ → `ledc_timer_config()`/`ledc_set_freq()`; `ledc_isr_register()` + `intr_type` deprecated → `ledc_cb_register()`
- Macro ที่ลบ: `LEDC_APB_CLK_HZ`, `LEDC_REF_CLK_HZ`, `LEDC_SLOW_CLK_RTC8M`, `LEDC_USE_RTC8M_CLK` → `LEDC_USE_RC_FAST_CLK`
- `esp_driver_ledc` เป็น component แยกตั้งแต่ v5.3 (+ `esp_driver_gpio` ถ้าใช้ GPIO API ร่วม)
- `driver/periph_ctrl.h` ลบ → `esp_private/periph_ctrl.h` (แต่ปกติไม่ต้องเรียกเอง — driver จัดการ)

## 10. จุดที่ต้องตรวจซ้ำกับ header จริง (ก่อนใช้งานจริง)
- `ledc_sleep_mode_t sleep_mode` field — ยืนยันใน v6.1; บน v5.2–v5.5 เปิด ledc.h ก่อนใช้
- คำเตือน duty = 2^res ที่ resolution สูงสุด — ยืนยันข้อความเต็มใน docs ของ C3; สำหรับ S3 ยึด header (range [0, 2**res]) + ใช้ 2^res − 1 เพื่อความปลอดภัย
- `CONFIG_LEDC_CTRL_FUNC_IN_IRAM` พฤติกรรมบน v5.x — เช่นเดียวกับ v6 หรือไม่
- `ledc_find_suitable_duty_resolution()` — มีตั้งแต่เวอร์ชันไหน (ยืนยัน v6.1; ยังไม่เช็ค v5.2–v5.5)