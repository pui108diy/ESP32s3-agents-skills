---
name: esp32s3-xtensa-abi
version: 1.0.0
description: Xtensa ABI skill สำหรับ ESP32-S3 — ครอบคลุม Windowed Register ABI vs CALL0 ABI, register usage conventions, stack frame layout, ENTRY instruction, argument passing, inline assembly syntax, และกฎการเขียน assembly ที่ถูกต้องสำหรับ Xtensa LX7 dual-core
triggers:
  - Xtensa
  - ABI
  - register window
  - CALL0
  - CALL4
  - CALL8
  - CALL12
  - ENTRY
  - inline assembly
  - asm
  - calling convention
  - stack frame
  - register usage
  - IRAM_ATTR
  - DRAM_ATTR
  - special register
  - WSR
  - RSR
  - window overflow
  - assembly
---

# Skill: Xtensa ABI for ESP32-S3

> **ขอบเขต:** ใช้สำหรับเขียน inline assembly หรือ assembly function บน ESP32-S3 (Xtensa LX7 dual-core) ให้ถูกต้องตาม ABI ป้องกัน stack corruption และ register window overflow
>
> **แหล่งข้อมูลหลัก:** Xtensa ISA Reference Manual (Chapter 4 + Chapter 8) + [Espressif Xtensa ABI Research](https://deepwiki.com/kassane/espressif-toolchains-research/3.1-xtensa-abi-and-calling-conventions) + [ESP-IDF Memory Types](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/memory-types.html)

---

## 1. ภาพใหญ่: ESP32-S3 ใช้ ABI อะไร?

```
ESP32-S3 (Xtensa LX7 dual-core)
  │
  ├── Default ABI: Windowed Register ABI (GCC default)
  │     └── ใช้ register window ที่หมุนได้ 4/8/12 ต่อการเรียก
  │     └── CALL4/CALL8/CALL12 + ENTRY instruction
  │
  └── Alternative: CALL0 ABI
        └── ไม่มี register window (flat register file)
        └── ใช้ใน ROM functions, bootloader, low-level interrupt handlers
        └── ต้อง recompile GCC เพื่อเปลี่ยน ABI
```

> **แหล่งเว็บ:** [deepwiki — Xtensa ABI](https://deepwiki.com/kassane/espressif-toolchains-research/3.1-xtensa-abi-and-calling-conventions) — "The windowed ABI is the default on ESP32 (classic) and ESP32-S3. The CALL0 ABI (windowless) is the default on ESP32-S2"

> **แหล่งข้อมูล:** Xtensa ISA Manual — "The windowed register ABI works with the Windowed Register Option and is the default ABI. The CALL0 ABI can be used with any Xtensa processor. It does not make use of register windows, so it typically has slightly worse performance and code size"

**⚠️ กฎสำคัญ:** ESP32-S3 application code ทั้งหมดใช้ **Windowed Register ABI** — ห้ามใช้ CALL0 ABI ใน application code เว้นแต่จะเขียน ROM-compatible code

---

## 2. Register File — พื้นฐาน

### 2.1 Address Registers (AR)

Xtensa มี **32 physical AR registers** (a0–a31) แต่โปรแกรมเห็นได้แค่ **16 ตัวในแต่ละเวลา** (a0–a15) เพราะ window บังความจริง:

```
Physical:    a0  a1  a2  a3  a4  a5  ...  a31
               ↓   ↓   ↓   ↓   ↓   ↓        ↓
Window 0:   [a0  a1  a2  a3  a4  a5  ...  a15]  ← เริ่มต้น
Window +4:      [a0  a1  a2  a3  a4  a5  ...  a15]  ← หลัง CALL4
Window +8:          [a0  a1  a2  a3  a4  a5  ...  a15]  ← หลัง CALL8
Window +12:             [a0  a1  a2  a3  a4  a5  ...  a15]  ← หลัง CALL12
```

> **แหล่งข้อมูล:** Xtensa ISA Manual — "Xtensa register increments are 4, 8, and 12 on a per-call basis, not a fixed increment as in other instruction sets"

### 2.2 Special Registers (สำคัญสำหรับ ESP32-S3)

| SR# | Name | Width | คำอธิบาย | การเข้าถึง |
|---|---|---|---|---|
| 1 | PSCALLINC | 2 | Window increment ที่ caller ส่งมา (1=CALL4, 2=CALL8, 3=CALL12) | RSR/WSR |
| 2 | PSWOE | 1 | Window Overflow Enable | RSR/WSR |
| 3 | PSEXCM | 1 | Exception Mode (1=ห้าม ENTRY) | RSR/WSR |
| 5 | WBWindowBase | 6 | ตำแหน่ง window ปัจจุบัน | RSR/WSR |
| 6 | WBWindowStart | 16 | แต่ละ bit บอกว่า window นั้น active หรือไม่ | RSR/WSR |
| 9 | LBEG/LEND/LCOUNTER | 32/32/32 | Loop registers (zero-overhead loop) | RSR/WSR |
| 12 | SAR | 6 | Shift Amount Register (สำหรับ SLL/SSR) | RSR/WSR |
| 16/17 | ACCLO/ACCHI | 32/8 | MAC16 accumulator (low/high) | RSR/WSR |
| 234 | CCOUNT | 32 | Cycle counter (ใช้วัดเวลา) | RSR/WSR |
| 240–242 | CCOMPARE0-2 | 32 | Cycle compare registers (timer interrupt) | RSR/WSR |

> **แหล่งข้อมูล:** xtensa_isa_en_use-by-esp32s3.pdf — Table 1.5-1 + Xtensa ISA Manual Chapter 2

---

## 3. Windowed Register ABI — Register Usage

### 3.1 ตาราง Register Usage (Windowed)

| Register | การใช้งาน | หมายเหตุ |
|---|---|---|
| **a0** | Return address | สงวนไว้ — ห้ามใช้เก็บค่าอื่น |
| **a1 (sp)** | Stack pointer | สงวนไว้ — ต้อง aligned 16-byte เสมอ |
| **a2–a7** | Incoming arguments + return value | a2 = return value, a2–a7 = args |
| **a7** | Frame pointer (optional) | ใช้เมื่อมี alloca |
| **a8–a11** | Caller-saved scratch | ใช้ชั่วคราวได้ ไม่ต้องเก็บ |
| **a12–a15** | Callee-saved | ถ้าใช้ต้อง save/restore |

> **แหล่งข้อมูล:** Xtensa ISA Manual Table 8–242 — "a0 Return address, a1 (sp) Stack pointer, a2–a7 Incoming arguments, a7 Callee's stack-frame pointer (optional)"

### 3.2 Argument Passing — Windowed ABI

```
ฟังก์ชัน: int f(int a, int b, int c, int d, int e, int f, int g, int h);

ถ้าเรียกด้วย CALL4:
  Caller ใส่ args ใน:  a6, a7, a8, a9, a10, a11  (6 ตัวแรก)
  Args ที่เหลือ (g, h):  วางบน stack ที่ SP+0, SP+4

  หลัง ENTRY:  callee เห็น args ใน a2, a3, a4, a5, a6, a7
                (เพราะ window หมุนไป 4 ตำแหน่ง)

ถ้าเรียกด้วย CALL8:
  Caller ใส่ args ใน:  a10, a11, a12, a13, a14, a15  (6 ตัวแรก)
  หลัง ENTRY:  callee เห็น args ใน a2, a3, a4, a5, a6, a7
```

> **แหล่งข้อมูล:** Xtensa ISA Manual Section 8.1.4 — "Arguments are passed in both registers and memory. In general, the first six words of arguments go in the AR register file, and any remaining arguments go on the stack. For a CALLN instruction, the caller places the first arguments in registers AR[N+2] through AR[N+7]"

### 3.3 Return Values

| ประเภท | Register |
|---|---|
| 32-bit return | **a2** |
| 64-bit return | **a2, a3** |
| 128-bit return | **a2, a3, a4, a5** |
| void | ไม่ใช้ |

---

## 4. CALL4 / CALL8 / CALL12 — เลือกใช้อย่างไร?

| Instruction | Window Increment | Caller Hidden | เหมาะกับ |
|---|---|---|---|
| **CALL4** | +4 | a0–a3 | ฟังก์ชันที่รับ arg ≤ 6 words และไม่ต้องการ scratch เยอะ |
| **CALL8** | +8 | a0–a7 | ฟังก์ชันที่รับ arg ≤ 6 words แต่ต้องการ scratch เยอะ |
| **CALL12** | +12 | a0–a11 | ฟังก์ชันที่รับ arg ≤ 6 words และต้องการ scratch + local เยอะมาก |

**⚠️ กฎสำคัญ:**
- ฟังก์ชันที่รับ args ใน a2–a7 **ต้องใช้ CALL4 หรือ CALL8 เท่านั้น** (ไม่ใช้ CALL12 เพราะจะบัง args)
- Compiler เลือก CALLN อัตโนมัติ — แต่ถ้าเขียน assembly เองต้องเลือกเอง
- `callx4/callx8/callx12` ใช้เมื่อ target address อยู่ใน register (indirect call)

> **แหล่งข้อมูล:** Xtensa ISA Manual — "Calls to routines that use a2..a7 for parameters may use only CALL4 or CALL8"

### 4.1 ตัวอย่าง Assembly: CALL4

```asm
    // In procedure g, the call z = f(x, y)
    mov a6, x          // a6 คือ f's a2 (x) หลัง window หมุน +4
    mov a7, y          // a7 คือ f's a3 (y)
    call4 f            // ใส่ return address ใน f's a0, กระโดดไป f
    mov z, a6          // a6 คือ f's a2 (return value)

// Function: int f(int a, int *b) { return a + *b; }
f:
    entry sp, 16       // หมุน window +4, จอง stack 16 bytes
    l32i a3, a3, 0     // *b (a3 = b pointer)
    add a2, a2, a3     // a + *b → return in a2
    retw               // หมุน window กลับ + return
```

> **แหล่งข้อมูล:** Xtensa ISA Manual — ตัวอย่าง assembly ใน Section 8.1

---

## 5. ENTRY Instruction — หัวใจของ Windowed ABI

### 5.1 หน้าที่ 2 อย่าง

```
ENTRY as, frame_size

1. หมุน register window:
   WindowBase += PS.CALLINC  (ที่ caller ตั้งไว้ใน CALL4/8/12)
   
2. จอง stack frame:
   SP_callee = SP_caller - (frame_size × 8)
   (frame_size เป็น imm12, นับเป็น units ของ 8 bytes)
   (สูงสุด = 32760 bytes)
```

> **แหล่งข้อมูล:** Xtensa ISA Manual — "ENTRY serves two purposes: 1. it increments the register window pointer (WindowBase) by the amount requested by the caller. 2. it copies the stack pointer from caller to callee and allocates the callee's stack frame"

### 5.2 กฎที่ต้องจำ

1. **ENTRY ต้องเป็นคำสั่งแรก** ของฟังก์ชันที่ถูกเรียกด้วย CALL4/8/12
2. **`as` ต้องเป็น a0–a3** เท่านั้น — ไม่งั้น undefined behavior
3. **ENTRY ห้ามใช้** ถ้า `PS.WOE = 0` หรือ `PS.EXCM = 1`
4. **frame_size** นับเป็น units ของ 8 bytes — ถ้าใส่ `16` หมายถึง 128 bytes
5. **Stack ต้อง aligned 16-byte** เสมอ

> **แหล่งข้อมูล:** Xtensa ISA Manual — "The as operand specifies the stack pointer register; it must specify one of a0..a3 or the operation of ENTRY is undefined"

### 5.3 RETW — กลับจากฟังก์ชัน

```asm
    retw    // หมุน window กลับ + return ไป address ใน a0
```

- `retw` = return with window rotation
- ห้ามใช้ `ret` (เป็นของ CALL0 ABI)

---

## 6. Stack Frame Layout — Windowed ABI

```
High Memory
  ┌──────────────────────────┐
  │  alloca Space            │  (ถ้ามี)
  ├──────────────────────────┤
  │  Outgoing Arguments      │  (args ที่เกิน 6 words ส่งให้ฟังก์ชันที่เรียก)
  ├──────────────────────────┤
  │  Local Variables         │
  ├──────────────────────────┤
  │  Register-Spill Overflow │  (0–8 words, ขึ้นกับ CALLN)
  │  (N-4 words)             │
  ├──────────────────────────┤  ← SP หลัง call
  │  Space for Arguments     │
  ├──────────────────────────┤
  │  Register-Spill Area     │  (4 words — a0..a3 ของ caller)
  ├──────────────────────────┤
  │  Register-Spill Area     │  (4 words — เก็บเพิ่มถ้า window overflow)
  └──────────────────────────┘  ← SP ก่อน call
Low Memory
```

**⚠️ กฎสำคัญ:**
- **Register-spill overflow area = N-4 words** (N = 4, 8, หรือ 12 จาก CALLN ที่ใหญ่ที่สุดในฟังก์ชัน)
- **4 words ใต้ SP** เป็น "red zone" — interrupt/exception ใช้พื้นที่นี้เก็บ a0–a3
- **SP ต้อง aligned 16-byte** — "the stack always reaches to 16 bytes below the contents of the stack pointer"

> **แหล่งข้อมูล:** Xtensa ISA Manual Figure 8–53 + — "Because the minimum number of registers to save is four, the processor stores four of call[i-1]'s registers, a0..a3, in this space"

---

## 7. CALL0 ABI — สำหรับ ROM/Bootloader Code

### 7.1 Register Usage (CALL0)

| Register | การใช้งาน | ประเภท |
|---|---|---|
| **a0** | Return address (เก็บชั่วคราว ไม่สงวน) | — |
| **a1 (sp)** | Stack pointer | Callee-saved |
| **a2–a7** | Function arguments | Caller-saved |
| **a8** | Static chain (สำหรับ nested functions) | Caller-saved |
| **a9–a11** | Scratch | Caller-saved |
| **a12–a15** | Callee-saved registers | Callee-saved |
| **a15** | Frame pointer (optional) | Callee-saved |

> **แหล่งข้อมูล:** Xtensa ISA Manual Table 8–243 — "Register a0 holds the return address upon entry to a function, but unlike the windowed register ABI, it is not reserved for this purpose and may hold other values after the return address has been saved"

### 7.2 ความแตกต่าง Windowed vs CALL0

| คุณสมบัติ | Windowed ABI | CALL0 ABI |
|---|---|---|
| Register window | ใช้ (หมุน 4/8/12) | ไม่ใช้ (flat) |
| Call instruction | CALL4/8/12 | CALL0 |
| Entry instruction | ENTRY | (ไม่มี — จอด stack เอง) |
| Return | RETW | RET |
| Return address | a0 (สงวน) | a0 (ชั่วคราว) |
| Callee-saved | a12–a15 (ใน window) | a12–a15 (save บน stack) |
| Frame pointer | a7 (optional) | a15 (optional) |
| Stack alignment | 16-byte | 16-byte |
| Register-spill area | มี (4+N-4 words) | ไม่มี |
| Performance | ดีกว่า (register ใหม่ทุก call) | แย่กว่า (save/restore บน stack) |
| Context switch | ช้ากว่า (ต้อง spill windows) | เร็วกว่า |

> **แหล่งข้อมูล:** Xtensa ISA Manual — "The CALL0 ABI can be used with any Xtensa processor. It does not make use of register windows, so it typically has slightly worse performance and code size"

---

## 8. Window Overflow / Underflow — Exception Handling

### 8.1 Window Overflow Exception

เมื่อ `CALLN` หมุน window เกินจำนวน physical registers (32 ตัว) → **WindowOverflowException**

- **Cause code:** `AllocaCause` (89) หรือ `WindowOverflowCause` (181)
- ROM exception handler จะ spill registers ของ call ที่เก่าที่สุดลง stack
- แต่ละ spill เก็บ **a0–a3** (4 words) ในพื้นที่ใต้ SP ของ call นั้น

### 8.2 Window Underflow Exception

เมื่อ `RETW` หมุน window กลับ แต่ registers ถูก spill ไปแล้ว → **WindowUnderflowException**

- ROM exception handler จะ restore registers จาก stack

**⚠️ สำหรับ Assembly Programmer:**
- ไม่ต้องจัดการ overflow/underflow เอง — ROM handler ทำให้
- แต่ต้องแน่ใจว่า **PS.WOE = 1** (Window Overflow Enable) — default ของ ESP-IDF คือเปิดอยู่
- ถ้า `PS.WOE = 0` แล้วเรียก CALLN + ENTRY → undefined behavior

> **แหล่งข้อมูล:** Xtensa ISA Manual — "ENTRY is undefined if PS.WOE is 0 or if PS.EXCM is 1"

---

## 9. Inline Assembly Syntax (GCC)

### 9.1 รูปแบบพื้นฐาน

```c
asm volatile (
    "assembly code"
    : output_operands     // ผลลัพธ์
    : input_operands      // ตัวแปรที่ใช้
    : clobbered_regs      // register ที่ถูกทำลาย
);
```

### 9.2 ตัวอย่าง: อ่าน CCOUNT (cycle counter)

```c
static inline uint32_t get_ccount(void) {
    uint32_t ccount;
    asm volatile (
        "rsr %0, CCOUNT\n"
        : "=r" (ccount)
        :
        : "memory"
    );
    return ccount;
}
```

### 9.3 ตัวอย่าง: NOP delay

```c
#define ASM_NOP() asm volatile("nop\n")
#define ASM_NOPS(n) do { \
    for (int i = 0; i < (n); i++) { asm volatile("nop\n"); } \
} while(0)
```

### 9.4 ตัวอย่าง: Atomic compare-and-swap (S32C1I)

```c
static inline bool atomic_cas(uint32_t *addr, uint32_t compare, uint32_t set) {
    uint32_t result;
    asm volatile (
        "s32c1i %2, %3, 0\n"   // ถ้า *addr == ACC, เขียน set ลง *addr
        "mov %0, %2\n"          // result = ค่าเดิมของ *addr
        : "=r" (result)
        : "r" (set), "r" (compare), "r" (addr)
        : "memory"
    );
    return (result == compare);
}
```

> **แหล่งข้อมูล:** Xtensa ISA Manual — "S32C1I is a conditional store instruction intended for updating synchronization variables in memory shared between multiple processors"

### 9.5 Constraint Letters (Xtensa-specific)

| Constraint | หมายถึง |
|---|---|
| `r` | General-purpose AR register |
| `a` | Address register (เหมือน r แต่เน้น address) |
| `b` | Boolean register (ถ้ามี Xtbool option) |
| `f` | Floating-point register (FPU) |
| `q` | QR register (PIE/SIMD — ESP32-S3 specific) |
| `I` | Immediate 12-bit signed (-2048..2047) |
| `J` | Immediate 0 |
| `K` | Constant 1 |
| `L` | Immediate 32-bit (สำหรับ MOVI) |
| `M` | Immediate 2-8, power of 2 (สำหรับ shifted add) |
| `N` | Immediate -1..-2048 |
| `O` | Immediate -2048..2047 |

> **แหล่งเว็บ:** [GCC Xtensa Constraints](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html)

---

## 10. IRAM_ATTR และ DRAM_ATTR — การจัดวาง Code/Data

### 10.1 Memory Attributes

| Attribute | วางใน | ใช้เมื่อไร |
|---|---|---|
| `IRAM_ATTR` | IRAM (Instruction RAM) | ISR และฟังก์ชันที่ต้องทำงานขณะ cache disabled |
| `DRAM_ATTR` | DRAM (Data RAM) | ตัวแปรที่ ISR อ่าน/เขียน |
| `RTC_NOINIT_ATTR` | RTC SLOW Memory | ข้อมูลข้าม deep sleep |
| `RTC_IRAM_ATTR` | RTC FAST Memory | โค้ด wake stub |
| `EXT_RAM_BSS_ATTR` | PSRAM (bss) | ข้อมูลขนาดใหญ่ (ต้องเปิด CONFIG_SPIRAM) |

### 10.2 กฎสำคัญ

```c
// ✅ ถูก — ISR ทั้งโค้ดและข้อมูลอยู่ใน RAM
void IRAM_ATTR my_isr(void *arg) {
    static DRAM_ATTR uint32_t counter = 0;  // ต้อง DRAM_ATTR!
    counter++;
}

// ❌ ผิด — ข้อมูลอยู่ใน flash, ISR อ่านไม่ได้ขณะ cache disabled
void IRAM_ATTR bad_isr(void *arg) {
    static uint32_t counter = 0;  // อาจอยู่ใน flash!
    counter++;
    ESP_LOGI("TAG", "count=%d", counter);  // ❌ format string ใน flash!
}
```

> **แหล่งเว็บ:** [ESP-IDF Memory Types — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/memory-types.html) — "Strings or constants inside an IRAM_ATTR function may not be placed in RAM automatically. It is possible to use DRAM_ATTR attributes to mark these"

> **แหล่งเว็บ:** [ESP32 Forum — IRAM_ATTR](https://esp32.com/viewtopic.php?t=4978) — "You need internal DRAM in ISR code because the IDF will not fetch from external FLASH/RAM when you are in an ISR"

---

## 11. Assembly Function Template — Windowed ABI

```asm
    .text
    .align 4

    // int my_function(int arg1, int arg2)
    // เรียกด้วย CALL4 (เพราะรับ args ใน a2-a7)
    .global my_function
    .type my_function, @function
my_function:
    entry sp, 32          // หมุน window +4, จอง stack 32 bytes (256 bytes)
                          // ⚠️ 32 × 8 = 256 bytes, ต้อง aligned 16-byte

    // --- ฟังก์ชัน body ---
    // a2 = arg1, a3 = arg2 (หลัง ENTRY)
    // a4-a7 = scratch
    // a8-a11 = scratch (ถ้าใช้ CALL8 แทน)
    
    movi a4, 10            // load immediate
    add a2, a2, a4         // arg1 + 10
    s32i a2, sp, 0         // store ไป local var บน stack

    // --- return value ใน a2 ---
    l32i a2, sp, 0         // load กลับ
    
    retw                   // return with window rotation
    .size my_function, . - my_function
```

---

## 12. Inline Assembly Template — ใช้ใน C Code

```c
#include <stdint.h>

// อ่าน special register
static inline uint32_t read_sar(void) {
    uint32_t val;
    asm volatile ("rsr %0, SAR" : "=r"(val));
    return val;
}

// เขียน special register
static inline void write_sar(uint32_t val) {
    asm volatile ("wsr %0, SAR" : : "r"(val));
}

// Memory barrier
#define ASM_MEMW() asm volatile("memw\n")
#define ASM_EXTW() asm volatile("extw\n")

// อ่าน PS register
static inline uint32_t read_ps(void) {
    uint32_t ps;
    asm volatile ("rsr %0, PS" : "=r"(ps));
    return ps;
}

// Disable interrupts (return previous level)
static inline uint32_t disable_interrupts(void) {
    uint32_t ps;
    asm volatile (
        "rsil %0, 15\n"    // set interrupt level to 15 (max)
        : "=r"(ps)
        :
        : "memory"
    );
    return ps;
}

// Restore interrupt level
static inline void restore_interrupts(uint32_t ps) {
    asm volatile (
        "wsr %0, PS\n"
        "rsync\n"          // ensure PS is written before continuing
        : : "r"(ps)
        : "memory"
    );
}
```

---

## 13. Interrupt Handler Assembly Pattern

```c
// Interrupt handler ต้องใช้ CALL0 ABI (ไม่ใช่ windowed)
// เพราะ interrupt เกิดขึ้นทุกเมื่อ — window อาจ overflow ไม่ได้

// ESP-IDF จัดการให้ — แต่ถ้าเขียนเอง:
void IRAM_ATTR my_raw_interrupt_handler(void) {
    // ⚠️ ต้องเป็น CALL0 ABI ถ้าเป็น raw handler
    // ESP-IDF interrupt allocator ใช้ windowed ABI wrapper
    // ดังนั้นปกติไม่ต้องกังวล
    
    // อ่าน interrupt status
    uint32_t status = REG_READ(UART_INT_ST_REG(0));
    
    // เคลียร์ interrupt
    REG_WRITE(UART_INT_CLR_REG(0), status);
    
    // ส่งสัญญาณไป task
    BaseType_t hpwoken = pdFALSE;
    xQueueSendFromISR(queue, &status, &hpwoken);
    if (hpwoken) portYIELD_FROM_ISR();
}
```

> **แหล่งเว็บ:** [ESP-IDF IRAM-Safe Interrupt Handlers](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/hlinterrupts.html)

---

## 14. Data Types and Alignment

| Data Type | Size (bytes) | Alignment (bytes) |
|---|---|---|
| `char` | 1 | 1 |
| `short` | 2 | 2 |
| `int` | 4 | 4 |
| `long` | 4 | 4 |
| `long long` | 8 | 8 |
| `float` | 4 | 4 |
| `double` | 8 | 8 |
| `void *` | 4 | 4 |
| `bool` | 1 | 1 |
| User struct (max align) | — | 16 |

> **แหล่งข้อมูล:** Xtensa ISA Manual Table 8–244 — "The maximum alignment for user-defined types is 16 bytes"

**⚠️ การส่ง float/double:** ESP32-S3 มี FPU hardware แต่การส่งผ่าน argument ยังใช้ AR registers (ไม่ใช่ FP registers)

---

## 15. Checklist ก่อนเขียน Assembly / Inline Assembly

- [ ] ยืนยันว่าใช้ **Windowed ABI** (default) หรือ **CALL0 ABI** (ROM/low-level)
- [ ] ฟังก์ชันเริ่มด้วย `entry sp, frame_size` (windowed) หรือจอด stack เอง (CALL0)
- [ ] `frame_size` นับเป็น units ของ 8 bytes (เช่น `16` = 128 bytes)
- [ ] SP ต้อง aligned 16-byte เสมอ
- [ ] Return ด้วย `retw` (windowed) ไม่ใช่ `ret`
- [ ] Return value ใน `a2` (32-bit) หรือ `a2:a3` (64-bit)
- [ ] ฟังก์ชันที่รับ args ใน a2–a7 ใช้ CALL4 หรือ CALL8 เท่านั้น
- [ ] Inline assembly ระบุ `clobber list` ให้ครบ (โดยเฉพาะ `"memory"` ถ้าแก้ memory)
- [ ] ISR มี `IRAM_ATTR` และตัวแปรมี `DRAM_ATTR`
- [ ] `wsr` ตามด้วย `rsync` ถ้าต้องการให้ค่าถูก commit ก่อนคำสั่งถัดไป
- [ ] `PS.WOE = 1` ก่อนใช้ windowed ABI (ESP-IDF ทำให้แล้ว)

---

## 16. Common Mistakes ที่ต้องหลีกเลี่ยง

| ผิด | ถูก | เหตุผล |
|---|---|---|
| `ret` ใน windowed function | `retw` | `ret` ไม่หมุน window → stack corruption |
| `call0` ใน application code | `call4`/`call8`/`call12` | Application ใช้ windowed ABI |
| `entry sp, 4` (หมายถึง 4 bytes) | `entry sp, 1` (1×8=8 bytes) | frame_size นับเป็น 8-byte units |
| ใช้ `a0` เก็บค่าอื่น | ห้ามแตะ a0 | a0 = return address ใน windowed ABI |
| `call12` กับฟังก์ชันที่รับ args | `call4`/`call8` | CALL12 บัง args ใน a2–a7 |
| ลืม `"memory"` ใน clobber list | ใส่ `"memory"` | compiler อาจ reorder load/store |
| ลืม `rsync` หลัง `wsr` | ใส่ `rsync` | ค่าอาจยังไม่ commit |
| ลืม `DRAM_ATTR` บนตัวแปรใน ISR | ใส่ `DRAM_ATTR` | ตัวแปรอาจอยู่ใน flash → crash |
| `entry a5, 16` | `entry sp, 16` (sp = a1) | `as` ต้องเป็น a0–a3 เท่านั้น |
| ลืม `.align 4` ก่อน function | `.align 4` | ENTRY ต้องอยู่ที่ 4-byte boundary |

---

## 17. แหล่งข้อมูลอ้างอิง

| หัวข้อ | แหล่งที่มา |
|---|---|
| Windowed Register ABI | Xtensa ISA Reference Manual Chapter 8.1 |
| CALL0 ABI | Xtensa ISA Reference Manual Chapter 8.1.2 |
| ENTRY instruction | Xtensa ISA Reference Manual (ENTRY description) |
| Stack frame layout | Xtensa ISA Reference Manual Figure 8–53 |
| Register usage tables | Xtensa ISA Reference Manual Tables 8–242, 8–243 |
| Argument passing | Xtensa ISA Reference Manual Section 8.1.4 |
| Window overflow/underflow | Xtensa ISA Reference Manual Section 4.7 |
| Special registers | xtensa_isa_en_use-by-esp32s3.pdf Table 1.5-1 |
| ESP32-S3 default ABI | [deepwiki — Xtensa ABI](https://deepwiki.com/kassane/espressif-toolchains-research/3.1-xtensa-abi-and-calling-conventions) |
| GCC inline assembly | [GCC Machine Constraints](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html) |
| IRAM/DRAM attributes | [ESP-IDF Memory Types — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/memory-types.html) |
| IRAM-Safe ISR | [ESP-IDF High-priority Interrupts](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/hlinterrupts.html) |
| CALL0 ABI usage | [ESP32 Forum — Calling Convention](https://www.esp32.com/viewtopic.php?t=4227) |
| S32C1I atomic operation | Xtensa ISA Manual Section 4.3.13 |
| Xtensa overview (Espressif) | [Espressif Xtensa ISA PDF](https://dl.espressif.com/github_assets/espressif/xtensa-isa-doc/releases/download/latest/Xtensa.pdf) |
