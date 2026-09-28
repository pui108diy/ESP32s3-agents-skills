# LEDC (LED PWM Controller) — ESP-IDF v5.x/v6.x (ESP32-S3)

## Include
```c
#include "driver/ledc.h"
```

## Hardware Facts (จาก TRM ESP32-S3 v1.8 บทที่ 35)
- **8 ช่องสัญญาณ PWM** (PWM0–PWM7) + **4 ตัวจับเวลาอิสระ** แบบหารความถี่เศษส่วนได้
- คล็อกที่เลือกได้: **APB_CLK, RC_FAST_CLK, XTAL_CLK** (เลือกทั้งชุดผ่าน `LEDC_APB_CLK_SEL`)
- Duty resolution สูงสุด **14 บิต** บน S3
- **S3 ไม่มี high-speed mode** — ใช้ `LEDC_LOW_SPEED_MODE` เท่านั้น
- Fade (ค่อย ๆ เปลี่ยน duty) ทำในฮาร์ดแวร์ ไม่แตะ CPU + มีอินเทอร์รัปต์เมื่อ fade จบ

## Structs
```c
typedef struct {
    ledc_mode_t speed_mode;        // ESP32-S3: LEDC_LOW_SPEED_MODE เท่านั้น!
    ledc_timer_bit_t duty_resolution; // LEDC_TIMER_1_BIT .. LEDC_TIMER_14_BIT
    ledc_timer_t timer_num;        // LEDC_TIMER_0..3
    uint32_t freq_hz;
    ledc_clk_cfg_t clk_cfg;        // LEDC_AUTO_CLK, LEDC_USE_APB_CLK, LEDC_USE_XTAL_CLK, LEDC_USE_RC_FAST_CLK
    bool deconfigure;
} ledc_timer_config_t;

typedef struct {
    int gpio_num;                  // ใดก็ได้ผ่าน GPIO Matrix
    ledc_mode_t speed_mode;        // LEDC_LOW_SPEED_MODE
    ledc_channel_t channel;        // LEDC_CHANNEL_0..7
    ledc_intr_type_t intr_type;    // LEDC_INTR_DISABLE / LEDC_INTR_FADE_END
    ledc_timer_t timer_sel;        // LEDC_TIMER_0..3
    uint32_t duty;                 // 0 .. (2^duty_resolution - 1)
    int hpoint;                    // จุดเริ่มนับ duty ใน period (default 0)
    struct {
        unsigned int output_invert: 1; // กลับขั้วเอาต์พุต (active-low)
    } flags;
} ledc_channel_config_t;

// v5.0+ callback API
typedef struct {
    ledc_cb_t on_duty_trans_done;  // fade/duty transition จบ
    ledc_cb_t on_loop_done;        // จบรอบ fade แบบ loop
    ledc_cb_t on_ovf_cnt;          // นับ overflow ครบ
} ledc_cbs_t;

typedef bool (*ledc_cb_t)(const ledc_cb_param_t *param, void *user_arg);
// ledc_cb_param_t: .event, .speed_mode, .channel, .duty, .hpoint
```

## Function Signatures (exact — ชุดเต็ม)
```c
// --- Timer ---
esp_err_t ledc_timer_config(const ledc_timer_config_t *timer_conf);
esp_err_t ledc_timer_rst(ledc_mode_t speed_mode, ledc_timer_t timer_num);
esp_err_t ledc_timer_pause(ledc_mode_t speed_mode, ledc_timer_t timer_num);
esp_err_t ledc_timer_resume(ledc_mode_t speed_mode, ledc_timer_t timer_num);
esp_err_t ledc_timer_set(ledc_mode_t speed_mode, ledc_timer_t timer_num,
                         uint32_t clock_divider, uint32_t duty_resolution, ledc_clk_cfg_t clk_cfg); // low-level
esp_err_t ledc_set_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num, uint32_t freq_hz);
uint32_t  ledc_get_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num);

// --- Channel ---
esp_err_t ledc_channel_config(const ledc_channel_config_t *ledc_conf);
esp_err_t ledc_set_pin(int gpio_num, ledc_mode_t speed_mode, ledc_channel_t ledc_channel);
esp_err_t ledc_bind_channel_timer(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_timer_t idx);
esp_err_t ledc_stop(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t idle_level);

// --- Duty ---
esp_err_t ledc_set_duty(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty);
esp_err_t ledc_update_duty(ledc_mode_t speed_mode, ledc_channel_t channel);   // ← ต้องเรียกหลัง set_duty เสมอ!
esp_err_t ledc_set_duty_and_update(ledc_mode_t speed_mode, ledc_channel_t channel,
                                   uint32_t duty, uint32_t hpoint);          // atomic (v5.0+)
esp_err_t ledc_set_duty_with_hpoint(ledc_mode_t speed_mode, ledc_channel_t channel,
                                    uint32_t duty, uint32_t hpoint);
int       ledc_get_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
uint32_t  ledc_get_max_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
uint32_t  ledc_get_hpoint(ledc_mode_t speed_mode, ledc_channel_t channel);

// --- Fade ---
esp_err_t ledc_fade_func_install(int intr_alloc_flags);   // เรียกก่อนใช้ fade ทั้งหมด!
void      ledc_fade_func_uninstall(void);
esp_err_t ledc_set_fade_with_time(ledc_mode_t speed_mode, ledc_channel_t channel,
                                  uint32_t target_duty, int max_fade_time_ms, ledc_fade_mode_t fade_mode);
esp_err_t ledc_set_fade_with_step(ledc_mode_t speed_mode, ledc_channel_t channel,
                                  uint32_t target_duty, uint32_t scale, uint32_t cycle_num, ledc_fade_mode_t fade_mode);
esp_err_t ledc_set_fade(ledc_mode_t speed_mode, ledc_channel_t channel,
                        uint32_t target_duty, uint32_t scale, uint32_t cycle_num, ledc_fade_mode_t fade_mode); // low-level
esp_err_t ledc_fade_start(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_fade_mode_t fade_mode);
esp_err_t ledc_fade_stop(ledc_mode_t speed_mode, ledc_channel_t channel);  // หยุด fade ค้าง (v5.x+)
// ledc_fade_mode_t: LEDC_FADE_NO_WAIT / LEDC_FADE_WAIT_DONE

// --- Callback (v5.0+) ---
esp_err_t ledc_cb_register(ledc_mode_t speed_mode, ledc_channel_t channel,
                           ledc_cbs_t *cbs, void *user_arg);
```

## สูตรความถี่ × ความละเอียด
$$f_{pwm,max} = \frac{f_{clk}}{2^{duty\_resolution}}$$
| duty_resolution | f_max @ 80 MHz |
|---|---|
| 14 บิต | ~4.88 kHz |
| 12 บิต | ~19.5 kHz |
| 10 บิต | ~78 kHz |
| 8 บิต | ~312 kHz |

## กับดัก Hallucination
1. โค้ดจาก ESP32 รุ่นแรกใช้ `LEDC_HIGH_SPEED_MODE` → **S3 ไม่มี high-speed mode** (compile error)
2. `ledc_set_duty()` แล้วไม่เรียก `ledc_update_duty()` → duty **ไม่เปลี่ยน** (ค่ารออยู่ใน shadow register)
3. `duty` เป็น **ค่าดิบ** ไม่ใช่ % — ที่ 13 บิต: 50% = 4096 (ไม่ใช่ 50)
4. duty เกิน `(2^resolution) - 1` → ตัวนับ duty ในฮาร์ดแวร์ overflow พัง (docs ระบุชัด: ที่ resolution สูงสุด ห้าม set เป็น `2^resolution`)
5. ลืม `ledc_fade_func_install()` ก่อน fade → เรียก fade ไม่ได้
6. `duty_resolution` เป็น **enum** `LEDC_TIMER_13_BIT` ไม่ใช่ตัวเลข 13
7. เปลี่ยน freq/timer ที่ timer ตัวเดียว → กระทบทุก channel ที่ผูก timer นั้น (ใช้ `ledc_bind_channel_timer` ย้ายก่อน)
8. LEDC มี 8 channel / 4 timer บน S3 — จองเกินไม่ได้

## ตัวอย่างมินิมัล (หายใจ LED ด้วย fade)
```c
#include "driver/ledc.h"

void app_main(void)
{
    ledc_timer_config_t timer = {
        .speed_mode = LEDC_LOW_SPEED_MODE,      // S3 = low speed เท่านั้น
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

    while (1) {
        ESP_ERROR_CHECK(ledc_set_fade_with_time(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 8191, 2000, LEDC_FADE_NO_WAIT));
        ESP_ERROR_CHECK(ledc_fade_start(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, LEDC_FADE_NO_WAIT));
        vTaskDelay(pdMS_TO_TICKS(2100));
    }
}
```