# ESP-DSP (PIE-Optimized DSP Library) — ESP-IDF v5.x/v6.x (ESP32-S3)

## ความสัมพันธ์กับ PIE (สำคัญ)
| ชั้น | อะไร | ใช้เมื่อไร |
|---|---|---|
| Instruction | PIE `EE.*` 128-bit SIMD (skill **esp32s3-pie-simd** + TRM ch.1 + Xtensa ISA) | เขียน kernel เอง |
| **Library** | **esp-dsp (`dsps_*`)** — C API ที่ใช้ PIE อัตโนมัติเมื่อ target = S3 | **งาน DSP ปกติ — ใช้อันนี้ก่อนเสมอ** |
| Inference | esp-dl (model API) | AI model — roadmap แยก |

**Hardware Facts (TRM ch.1 + POM):** PIE ของ S3 = ส่วนขยายแบบ TIE (Xtensa): register ใหม่ 128-bit, vector ops 8/16/32-bit (รวม complex multiply), รวม load/store กับ data handling, รองรับข้อมูล non-aligned 128-bit — ผลคือ `dsps_dotprod_f32` บน S3 เร็วกว่า ANSI ~3 เท่า (432 vs 1311 cycles @ N=256 ตาม bench ทางการ)

## ติดตั้ง
```
idf.py add-dependency "espressif/esp-dsp"
```
```c
#include "esp_dsp.h"   // รวมทุกโมดูล
```

## Function Signatures (exact — ชุดที่ใช้บ่อย)
```c
// --- FFT (fc32 = complex float32; in-place; N = power of 2) ---
esp_err_t dsps_fft2r_init_fc32(void);      // ต้องเรียกก่อนใช้ FFT ครั้งแรก!
void      dsps_fft2r_fc32(float *data, int N);
void      dsps_bit_rev_fc32(float *data, int N);
void      dsps_cplx2reC_fc32(float *data, int N);   // จัด spectrum สำหรับ real-input FFT
esp_err_t dsps_fft2r_deinit_fc32(void);

// --- Vector math (step ใช้ข้าม stride ได้ ปกติใส่ 1) ---
esp_err_t dsps_add_f32(const float *in1, const float *in2, float *out, int len, int step1, int step2, int step_out);
esp_err_t dsps_sub_f32(...); esp_err_t dsps_mul_f32(...);            // รูปแบบเดียวกัน
esp_err_t dsps_addc_f32(const float *in, float *out, int len, float c, int step_in, int step_out);
esp_err_t dsps_mulc_f32(...); esp_err_t dsps_scale_f32(...);

float dsps_dotprod_f32(const float *src1, const float *src2, int len);
float dsps_dotprode_f32(const float *src1, const float *src2, int len, int step1, int step2);
float dsps_dotprod_s16(const int16_t *src1, const int16_t *src2, int len, int8_t shift);

// --- Window ---
void dsps_wind_hann_f32(float *w, int len);
void dsps_wind_blackman_f32(float *w, int len);
void dsps_wind_blackman_harris_f32(float *w, int len);

// --- FIR (fird = มี decimation) ---
esp_err_t dsps_fir_init_f32(fir_f32_t *fir, const float *coeffs, float *delay, int coeffs_len);
esp_err_t dsps_fir_f32(fir_f32_t *fir, const float *in, float *out, int len);
esp_err_t dsps_fird_init_f32(fir_f32_t *fir, const float *coeffs, float *delay,
                             int coeffs_len, int decim, int start_pos);
esp_err_t dsps_fird_f32(fir_f32_t *fir, const float *in, float *out, int len);

// --- Biquad / Matrix (⚠️ เทียบ header ของเวอร์ชันที่ใช้อีกครั้ง) ---
esp_err_t dsps_biquad_f32(const float *in, const float *coeffs, float *out, int len, int step, float *w);
esp_err_t dspm_mult_f32(const float *A, const float *B, float *C, int m, int n, int k);
```

## กับดัก Hallucination
1. **ลืม `dsps_fft2r_init_fc32()`** ก่อน FFT → crash/ผลผิด (ตาราง sin/cos ยังไม่ถูกจอง); ขนาด FFT สูงสุดกำหนดใน menuconfig ของ esp-dsp
2. **N ต้องเป็น power of 2** และ input 1 ค่า complex = 2 floats interleave → `float x[2*N]`
3. เรียก `fft2r` อย่างเดียว **ไม่พอ** — ผลลัพธ์ bit-reversed → ต้องตามด้วย `dsps_bit_rev_fc32`; ถ้า input เป็น real signal ต้อง `dsps_cplx2reC_fc32` อีกชั้น
4. ส่ง **const array / string literal** เข้า FFT → พัง (ทำงาน in-place กับข้อมูลที่แก้ได้เท่านั้น — และอยู่ใน RAM ไม่ใช่ flash)
5. FFT บนข้อมูลใน **PSRAM** ช้ากว่า DRAM ภายในหลายเท่า — buffer ร้อนควร internal (`heap_caps_malloc(..., MALLOC_CAP_INTERNAL)`)
6. `fir` handle มี state → ห้ามแชร์ตัวเดียวข้าม task; `delay` buffer ต้องขนาด `coeffs_len` และเริ่มด้วย 0
7. Build ผิด target (ไม่ใช่ S3) → ได้เวอร์ชัน ANSI ช้าโดยไม่มี error ใด ๆ
8. `dsps_dotprod_s16` มี `shift` ปรับ scale ผลลัพธ์ — ลืมใส่ค่าจะได้เลขแปลก

## ตัวอย่างมินิมัล (FFT magnitude 1024 samples)
```c
#include "esp_dsp.h"
#define N 1024

void fft_magnitude(const float *input, float *mag)
{
    static float x[2 * N];      // internal DRAM — เร็วกว่า PSRAM
    static float wind[N];

    dsps_fft2r_init_fc32();     // ครั้งแรกครั้งเดียว
    dsps_wind_hann_f32(wind, N);
    for (int i = 0; i < N; i++) {
        x[2 * i]     = input[i] * wind[i];   // real
        x[2 * i + 1] = 0.0f;                 // imag
    }
    dsps_fft2r_fc32(x, N);
    dsps_bit_rev_fc32(x, N);
    dsps_cplx2reC_fc32(x, N);
    for (int i = 0; i < N / 2; i++) {
        mag[i] = sqrtf(x[2 * i] * x[2 * i] + x[2 * i + 1] * x[2 * i + 1]);
    }
}
```