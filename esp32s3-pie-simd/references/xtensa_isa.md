# Xtensa CPU Instruction Set Reference Manual

> This document compiles all Xtensa CPU instructions, registers, and usage notes, for easy instruction lookup and use by a code agent.

---

## Table of Contents

1. [Register Overview](#1-register-overview)
2. [Special Registers](#2-special-registers)
3. [User Registers](#3-user-registers)
4. [Instruction Formats](#4-instruction-formats)
5. [Instruction List](#5-instruction-list)
6. [Instruction Details](#6-instruction-details)
7. [Exceptions and Interrupts](#7-exceptions-and-interrupts)
8. [Memory Management](#8-memory-management)

---

## 1. Register Overview

### 1.1 Address Registers (AR)

| Register | Name | Description |
|--------|------|------|
| a0-a15 | Address registers | General-purpose 32-bit registers, used for address calculation and data operations |
| a0 | Return address | Function call return address |
| a1 (sp) | Stack pointer | Points to the current top of stack |
| a2-a7 | Arguments/return values | Used to pass function arguments and return values |

### 1.2 Boolean Registers (BR)

| Register | Name | Description |
|--------|------|------|
| b0-b15 | Boolean registers | 1-bit boolean values, used for conditional testing |

### 1.3 Floating-Point Registers (FR)

| Register | Name | Description |
|--------|------|------|
| f0-f15 | Floating-point registers | 32-bit IEEE754 single-precision floating-point numbers |

---

## 2. Special Registers

### 2.1 Loop/Shift/Control Registers

| SR# | Name | Description | Width | Privileged | Reset Value |
|-----|------|------|------|------|--------|
| 0 | LBEG | Loop start address | 32 | No | 0 |
| 1 | LEND | Loop end address | 32 | No | 0 |
| 2 | LCOUNT | Loop counter | 32 | No | 0 |
| 3 | SAR | Shift amount register | 6 | No | 0 |
| 4 | BR / b0..15 | Boolean register | 16 | No | 0 |
| 5 | LITBASE | Literal base address | 21 | No | 0 |
| 12 | SCOMPARE1 | S32C1I conditional-store compare register | 32 | No | 0 |

### 2.2 Window Registers

| SR# | Name | Description | Width | Privileged |
|-----|------|------|------|------|
| 72 | WINDOWBASE | Window base | 4-8 | Yes |
| 73 | WINDOWSTART | Window start | 16 | Yes |

### 2.3 Program State Register (PS)

| SR# | Name | Description | Width | Privileged |
|-----|------|------|------|------|
| 230 | PS | Program state | 15 | Yes |

**PS register fields:**
- **INTLEVEL** (bits 0-3): Interrupt level
- **EXCM** (bit 4): Exception mode
- **UM** (bit 5): User mode
- **RING** (bits 6-7): Privilege ring
- **OWB** (bits 8-11): Old window base
- **CALLINC** (bits 16-17): Call increment
- **WOE** (bit 18): Window overflow enable

### 2.4 Exception-Related Registers

| SR# | Name | Description | Width | Privileged |
|-----|------|------|------|------|
| 177 | EPC1 | Exception program counter (Level-1) | 32 | Yes |
| 178-183 | EPC2-EPC7 | Exception program counter (high level) | 32 | Yes |
| 192 | DEPC | Double-exception program counter | 32 | Yes |
| 194-199 | EPS2-EPS7 | Exception program state | 32 | Yes |
| 209 | EXCSAVE1 | Exception save (Level-1) | 32 | Yes |
| 210-215 | EXCSAVE2-EXCSAVE7 | Exception save (high level) | 32 | Yes |
| 232 | EXCCAUSE | Exception cause | 32 | Yes |
| 238 | EXCVADDR | Exception virtual address | 32 | Yes |

### 2.5 Interrupt-Related Registers

| SR# | Name | Description | Width | Privileged |
|-----|------|------|------|------|
| 226 | INTERRUPT | Interrupt status (read) | 32 | Yes |
| 226 | INTSET | Interrupt set (write) | 32 | Yes |
| 227 | INTCLEAR | Interrupt clear | 32 | Yes |
| 228 | INTENABLE | Interrupt enable | 32 | Yes |

### 2.6 Timer Registers

| SR# | Name | Description | Width | Privileged |
|-----|------|------|------|------|
| 234 | CCOUNT | Clock counter | 32 | Yes |
| 240-242 | CCOMPARE0-CCOMPARE2 | Clock compare registers | 32 | Yes |

### 2.7 MAC16-Related Registers

| SR# | Name | Description | Width | Privileged |
|-----|------|------|------|------|
| 16 | ACCLO | MAC accumulator, low 32 bits | 32 | No |
| 17 | ACCHI | MAC accumulator, high 8 bits | 8 | No |
| 32-35 | M0..3 / MR | MAC16 data registers / read registers (same group) | 32 | No |

### 2.8 Debug-Related Registers

| SR# | Name | Description | Width | Privileged |
|-----|------|------|------|------|
| 96 | IBREAKENABLE | Instruction breakpoint enable | 2 | Yes |
| 104 | DDR | Debug data register | 32 | Yes |
| 128-129 | IBREAKA0-IBREAKA1 | Instruction breakpoint address | 32 | Yes |
| 144-145 | DBREAKA0-DBREAKA1 | Data breakpoint address | 32 | Yes |
| 160-161 | DBREAKC0-DBREAKC1 | Data breakpoint control | 32 | Yes |
| 233 | DEBUGCAUSE | Debug cause | 32 | Yes |
| 236 | ICOUNT | Instruction count | 32 | Yes |
| 237 | ICOUNTLEVEL | Instruction count level | 4 | Yes |

### 2.9 TLB-Related Registers

| SR# | Name | Description | Width | Privileged |
|-----|------|------|------|------|
| 83 | PTEVADDR | PTE virtual address | 32 | Yes |
| 90 | RASID | Ring ASID | 32 | Yes |
| 91 | ITLBCFG | Instruction TLB configuration | 32 | Yes |
| 92 | DTLBCFG | Data TLB configuration | 32 | Yes |

### 2.10 Other Registers

| SR# | Name | Description | Width | Privileged |
|-----|------|------|------|------|
| 89 | MMID | Memory-mapped ID | 32 | Yes |
| 99 | ATOMCTL | Atomic control | 32 | Yes |
| 224 | CPENABLE | Coprocessor enable | 32 | Yes |
| 231 | VECBASE | Vector base address | 32 | Yes |
| 235 | PRID | Processor ID | 32 | Yes |
| 244-247 | MISC0-MISC3 | Miscellaneous registers | 32 | Yes |

**Note:** THREADPTR is user register #231, and is not a special register (SR). See [3. User Registers](#3-user-registers).

---

## 3. User Registers

| Number | Name | Description | Width |
|------|------|------|------|
| 0-15 | FCR | Floating-point control register | 32 |
| 0-15 | FSR | Floating-point status register | 32 |
| 231 | THREADPTR | Thread pointer | 32 |

---

## 4. Instruction Formats

### 4.1 RRR Format (24 bits)

```
23 20 19 16 15 12 11 8 7 4 3 0
+--+--+--+--+--+--+--+--+--+--+
| op2 | op1 | t | s | r | op0 |
+--+--+--+--+--+--+--+--+--+--+
```

### 4.2 RRI8 Format (24 bits)

```
23 20 19 16 15 12 11 8 7 0
+--+--+--+--+--+--+--+--+
| op2 | op1 | t | s | imm8 |
+--+--+--+--+--+--+--+--+
```

### 4.3 RRI4 Format (24 bits)

```
23 20 19 16 15 12 11 8 7 4 3 0
+--+--+--+--+--+--+--+--+--+--+
| op2 | op1 | t | s | r | imm4 |
+--+--+--+--+--+--+--+--+--+--+
```

### 4.4 RSR Format (24 bits)

```
23 20 19 16 15 8 7 4 3 0
+--+--+--+--+--+--+--+--+
| op2 | op1 | sr | t | op0 |
+--+--+--+--+--+--+--+--+
```

### 4.5 CALL Format (24 bits)

```
23 20 19 18 17 16 15 0
+--+--+--+--+--+--+--+
| op0 | n | offset (18 bits) |
+--+--+--+--+--+--+--+
```

### 4.6 BRI8 Format (24 bits)

```
23 20 19 18 17 16 15 12 11 8 7 0
+--+--+--+--+--+--+--+--+--+--+
| op0 | n | m | r | s | imm8 |
+--+--+--+--+--+--+--+--+--+--+
```

### 4.7 BRI12 Format (24 bits)

```
23 20 19 18 17 16 15 12 11 0
+--+--+--+--+--+--+--+--+--+
| op0 | n | m | s | imm12 |
+--+--+--+--+--+--+--+--+--+
```

### 4.8 RI16 Format (24 bits)

```
23 20 19 16 15 0
+--+--+--+--+--+
| op0 | t | imm16 |
+--+--+--+--+--+
```

### 4.9 RRRN Format (16 bits)

```
15 12 11 8 7 4 3 0
+--+--+--+--+--+--+
| op0 | t | s | r |
+--+--+--+--+--+--+
```

### 4.10 RI7 Format (16 bits)

```
15 12 11 8 7 4 3 0
+--+--+--+--+--+--+
| op0 | t | s | imm7(4+3) |
+--+--+--+--+--+--+
```

### 4.11 RI6 Format (16 bits)

```
15 12 11 8 7 6 5 4 3 0
+--+--+--+--+--+--+--+--+
| op0 | t | z | s | imm6(2+4) |
+--+--+--+--+--+--+--+--+
```

---

## 5. Instruction List

### 5.1 Arithmetic Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| ADD | Addition | RRR | Core |
| ADDI | Immediate addition | RRI8 | Core |
| ADDMI | Immediate addition (16-bit) | RRI8 | Core |
| ADDX2 | Shift-left-1 addition | RRR | Core |
| ADDX4 | Shift-left-2 addition | RRR | Core |
| ADDX8 | Shift-left-3 addition | RRR | Core |
| SUB | Subtraction | RRR | Core |
| SUBX2 | Shift-left-1 subtraction | RRR | Core |
| SUBX4 | Shift-left-2 subtraction | RRR | Core |
| SUBX8 | Shift-left-3 subtraction | RRR | Core |
| NEG | Negate | RRR | Core |
| ABS | Absolute value | RRR | Core |

### 5.2 Logical Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| AND | Bitwise AND | RRR | Core |
| OR | Bitwise OR | RRR | Core |
| XOR | Bitwise XOR | RRR | Core |
| ANDB | Boolean AND | RRR | Boolean |
| ORB | Boolean OR | RRR | Boolean |
| XORB | Boolean XOR | RRR | Boolean |

### 5.3 Shift Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| SLL | Logical shift left | RRR | Core |
| SRL | Logical shift right | RRR | Core |
| SRA | Arithmetic shift right | RRR | Core |
| SLLI | Immediate logical shift left | RRR | Core |
| SRLI | Immediate logical shift right | RRR | Core |
| SRAI | Immediate arithmetic shift right | RRR | Core |
| SRC | Funnel (rotate) shift | RRR | Core |
| SSR | Set shift amount for right shift | RRR | Core |
| SSL | Set shift amount for left shift | RRR | Core |
| SSA8L | Set 8-bit left-shift alignment | RRR | Core |
| SSA8B | Set 8-bit right-shift alignment | RRR | Core |
| SSAI | Set immediate shift alignment | RRR | Core |

### 5.4 Multiply Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| MULL | 32-bit multiply (low 32 bits) | RRR | 32-bit Multiply |
| MULUH | Unsigned 32-bit multiply (high 32 bits) | RRR | 32-bit Multiply |
| MULSH | Signed 32-bit multiply (high 32 bits) | RRR | 32-bit Multiply |
| MUL16U | Unsigned 16-bit multiply | RRR | 16-bit Multiply |
| MUL16S | Signed 16-bit multiply | RRR | 16-bit Multiply |

### 5.5 Divide Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| QUOU | Unsigned divide | RRR | 32-bit Divide |
| QUOS | Signed divide | RRR | 32-bit Divide |
| REMU | Unsigned remainder | RRR | 32-bit Divide |
| REMS | Signed remainder | RRR | 32-bit Divide |

### 5.6 MAC16 Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| MUL.AA.* | Multiply | RRR | MAC16 |
| MULA.AA.* | Multiply-add | RRR | MAC16 |
| MULS.AA.* | Multiply-subtract | RRR | MAC16 |
| MUL.AD.* | Multiply (mixed) | RRR | MAC16 |
| MULA.AD.* | Multiply-add (mixed) | RRR | MAC16 |
| MULS.AD.* | Multiply-subtract (mixed) | RRR | MAC16 |
| MUL.DA.* | Multiply (mixed) | RRR | MAC16 |
| MULA.DA.* | Multiply-add (mixed) | RRR | MAC16 |
| MULS.DA.* | Multiply-subtract (mixed) | RRR | MAC16 |
| MUL.DD.* | Multiply | RRR | MAC16 |
| MULA.DD.* | Multiply-add | RRR | MAC16 |
| MULS.DD.* | Multiply-subtract | RRR | MAC16 |
| UMUL.AA.* | Unsigned multiply | RRR | MAC16 |
| LDINC | Load and increment | RRR | MAC16 |
| LDDEC | Load and decrement | RRR | MAC16 |

**Note:** * may be LL, LH, HL, or HH (indicating high/low half-word combinations)

### 5.7 Load Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| L8UI | Load unsigned 8-bit | RRI8 | Core |
| L16UI | Load unsigned 16-bit | RRI8 | Core |
| L16SI | Load signed 16-bit | RRI8 | Core |
| L32I | Load 32-bit | RRI8 | Core |
| L32R | Load from literal pool | RI16 | Core |
| L32E | Load 32-bit (exception handling) | RRI4 | Windowed Register |
| L32AI | Atomic load 32-bit | RRI8 | Multiprocessor Synchronization |

### 5.8 Store Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| S8I | Store 8-bit | RRI8 | Core |
| S16I | Store 16-bit | RRI8 | Core |
| S32I | Store 32-bit | RRI8 | Core |
| S32E | Store 32-bit (exception handling) | RRI4 | Windowed Register |
| S32RI | Atomic store 32-bit | RRI8 | Multiprocessor Synchronization |
| S32C1I | Conditional store 32-bit | RRI8 | Conditional Store |

### 5.9 Floating-Point Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| ADD.S | Floating-point add | RRR | Floating-Point |
| SUB.S | Floating-point subtract | RRR | Floating-Point |
| MUL.S | Floating-point multiply | RRR | Floating-Point |
| MADD.S | Floating-point multiply-add | RRR | Floating-Point |
| MSUB.S | Floating-point multiply-subtract | RRR | Floating-Point |
| ABS.S | Floating-point absolute value | RRR | Floating-Point |
| NEG.S | Floating-point negate | RRR | Floating-Point |
| MOV.S | Floating-point move | RRR | Floating-Point |
| RFR | Read from AR into FR | RRR | Floating-Point |
| WFR | Write from FR into AR | RRR | Floating-Point |
| FLOAT.S | Convert integer to float | RRR | Floating-Point |
| UFLOAT.S | Convert unsigned integer to float | RRR | Floating-Point |
| TRUNC.S | Truncate float to integer | RRR | Floating-Point |
| UTRUNC.S | Truncate float to unsigned integer | RRR | Floating-Point |
| ROUND.S | Round float | RRR | Floating-Point |
| CEIL.S | Round float up (ceiling) | RRR | Floating-Point |
| FLOOR.S | Round float down (floor) | RRR | Floating-Point |

### 5.10 Floating-Point Comparison Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| OEQ.S | Ordered equal compare | RRR | Floating-Point |
| OLD.S | Ordered less-than compare | RRR | Floating-Point |
| OLE.S | Ordered less-than-or-equal compare | RRR | Floating-Point |
| UEQ.S | Unordered-or-equal compare | RRR | Floating-Point |
| ULT.S | Unordered-or-less-than compare | RRR | Floating-Point |
| ULE.S | Unordered-or-less-than-or-equal compare | RRR | Floating-Point |
| UN.S | Unordered compare (NaN check) | RRR | Floating-Point |

### 5.11 Branch Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| J | Unconditional jump | CALL | Core |
| JX | Register jump | CALLX | Core |
| BEQ | Branch if equal | RRI8 | Core |
| BNE | Branch if not equal | RRI8 | Core |
| BLT | Branch if less than (signed) | RRI8 | Core |
| BGE | Branch if greater-or-equal (signed) | RRI8 | Core |
| BLTU | Branch if less than (unsigned) | RRI8 | Core |
| BGEU | Branch if greater-or-equal (unsigned) | RRI8 | Core |
| BEQI | Branch if equal to immediate | BRI8 | Core |
| BNEI | Branch if not equal to immediate | BRI8 | Core |
| BLTI | Branch if less than immediate | BRI8 | Core |
| BGEI | Branch if greater-or-equal to immediate | BRI8 | Core |
| BLTUI | Branch if less than immediate (unsigned) | BRI8 | Core |
| BGEUI | Branch if greater-or-equal to immediate (unsigned) | BRI8 | Core |
| BEQZ | Branch if equal to zero | BRI12 | Core |
| BNEZ | Branch if not equal to zero | BRI12 | Core |
| BLTZ | Branch if less than zero | BRI12 | Core |
| BGEZ | Branch if greater-or-equal to zero | BRI12 | Core |
| BNONE | Branch if no bits set | RRI8 | Core |
| BALL | Branch if all bits set | RRI8 | Core |
| BANY | Branch if any bit set | RRI8 | Core |
| BNALL | Branch if not all bits set | RRI8 | Core |
| BBC | Branch if bit clear | RRI8 | Core |
| BBS | Branch if bit set | RRI8 | Core |
| BBCI | Branch if immediate-selected bit clear | RRI8 | Core |
| BBSI | Branch if immediate-selected bit set | RRI8 | Core |

### 5.12 Call Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| CALL0 | Call (no window) | CALL | Core |
| CALL4 | Call (4-register window) | CALL | Windowed Register |
| CALL8 | Call (8-register window) | CALL | Windowed Register |
| CALL12 | Call (12-register window) | CALL | Windowed Register |
| CALLX0 | Register call (no window) | CALLX | Core |
| CALLX4 | Register call (4-register window) | CALLX | Windowed Register |
| CALLX8 | Register call (8-register window) | CALLX | Windowed Register |
| CALLX12 | Register call (12-register window) | CALLX | Windowed Register |

### 5.13 Return Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| RET | Return | CALLX | Core |
| RETW | Windowed return | CALLX | Windowed Register |
| RETW.N | Windowed return (narrow) | RRRN | Windowed Register + Code Density |

### 5.14 Loop Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| LOOP | Zero-overhead loop | BRI8 | Loop |
| LOOPGTZ | Loop if greater than zero | BRI8 | Loop |
| LOOPNEZ | Loop if not equal to zero | BRI8 | Loop |

### 5.15 Special Register Access Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| RSR.* | Read special register | RSR | Core |
| WSR.* | Write special register | RSR | Core |
| XSR.* | Exchange special register | RSR | Core |

### 5.16 User Register Access Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| RUR.* | Read user register | RSR | Coprocessor |
| WUR.* | Write user register | RSR | Coprocessor |

### 5.17 Synchronization Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| ISYNC | Instruction sync | RRR | Core |
| DSYNC | Data sync | RRR | Core |
| RSYNC | Register sync | RRR | Core |
| ESYNC | Execute sync | RRR | Core |
| MEMW | Memory wait | RRR | Core |
| EXTW | Extended wait | RRR | Core |
| EXCW | Exception wait | RRR | Core |

### 5.18 Cache Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| IHI | Instruction cache hit invalidate | RRI4 | Instruction Cache |
| III | Instruction cache index invalidate | RRI4 | Instruction Cache |
| IPF | Instruction cache prefetch | RRI4 | Instruction Cache |
| IPFL | Instruction cache prefetch and lock | RRI4 | Instruction Cache |
| DHI | Data cache hit invalidate | RRI4 | Data Cache |
| DII | Data cache index invalidate | RRI4 | Data Cache |
| DPF | Data cache prefetch | RRI4 | Data Cache |
| DPFL | Data cache prefetch and lock | RRI4 | Data Cache |
| DWB | Data cache writeback | RRI4 | Data Cache |
| DWBI | Data cache writeback and invalidate | RRI4 | Data Cache |
| DIWB | Data cache index writeback | RRI4 | Data Cache |
| DIWBI | Data cache index writeback and invalidate | RRI4 | Data Cache |
| DHU | Data cache hit unlock | RRI4 | Data Cache |
| DIU | Data cache index unlock | RRI4 | Data Cache |
| IHUL | Instruction cache hit unlock | RRI4 | Instruction Cache |
| IIUL | Instruction cache index unlock | RRI4 | Instruction Cache |
| LICW | Lock instruction cache way | RRR | Instruction Cache Index Lock |
| SICW | Unlock instruction cache way | RRR | Instruction Cache Index Lock |
| LDCW | Lock data cache way | RRR | Data Cache Index Lock |
| SDCW | Unlock data cache way | RRR | Data Cache Index Lock |

### 5.19 TLB Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| RITLB0 | Read instruction TLB entry 0 | RRR | Region Translation/MMU |
| RITLB1 | Read instruction TLB entry 1 | RRR | Region Translation/MMU |
| RDTLB0 | Read data TLB entry 0 | RRR | Region Translation/MMU |
| RDTLB1 | Read data TLB entry 1 | RRR | Region Translation/MMU |
| WITLB | Write instruction TLB entry | RRR | Region Translation/MMU |
| WDTLB | Write data TLB entry | RRR | Region Translation/MMU |
| IITLB | Invalidate instruction TLB entry | RRR | Region Translation/MMU |
| IDTLB | Invalidate data TLB entry | RRR | Region Translation/MMU |
| PITLB | Probe instruction TLB | RRR | Region Translation/MMU |
| PDTLB | Probe data TLB | RRR | Region Translation/MMU |

### 5.20 Exception and Interrupt Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| RFE | Return from exception | RRR | Exception |
| RFUE | Return from user exception | RRR | Exception (XEA1) |
| RFI | Return from interrupt | RRR | Interrupt |
| RFME | Return from memory error | RRR | Exception |
| RFDE | Return from double exception | RRR | Exception |
| RFWO | Return from window overflow | RRR | Windowed Register |
| RFWU | Return from window underflow | RRR | Windowed Register |
| SYSCALL | System call | RRR | Exception |
| WAITI | Wait for interrupt | RRR | Interrupt |
| RSIL | Read and set interrupt level | RRR | Interrupt |
| BREAK | Breakpoint | RRR | Debug |
| BREAK.N | Breakpoint (narrow) | RRRN | Debug + Code Density |

### 5.21 Other Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| MOV | Move | RRR | Core |
| MOVI | Immediate move | RRI8 | Core |
| MOV.N | Move (narrow) | RRRN | Code Density |
| MOVI.N | Immediate move (narrow) | RI7 | Code Density |
| MOVEQZ | Move if equal to zero | RRR | Miscellaneous Operations |
| MOVNEZ | Move if not equal to zero | RRR | Miscellaneous Operations |
| MOVLTZ | Move if less than zero | RRR | Miscellaneous Operations |
| MOVGEZ | Move if greater-or-equal to zero | RRR | Miscellaneous Operations |
| MOVFP | Move if floating-point false | RRR | Floating-Point |
| MOVTP | Move if floating-point true | RRR | Floating-Point |
| ENTRY | Function entry | BRI8 | Windowed Register |
| ILL | Illegal instruction | CALLX | Core |
| ILL.N | Illegal instruction (narrow) | RRRN | Code Density |
| NOP | No operation | RRR | Core |
| NOP.N | No operation (narrow) | RRRN | Code Density |
| RER | Read external register | RRR | Core |
| WER | Write external register | RRR | Core |

### 5.22 Miscellaneous Operation Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| MIN | Minimum | RRR | Miscellaneous Operations |
| MAX | Maximum | RRR | Miscellaneous Operations |
| MINU | Unsigned minimum | RRR | Miscellaneous Operations |
| MAXU | Unsigned maximum | RRR | Miscellaneous Operations |
| CLAMPS | Signed clamp | RRR | Miscellaneous Operations |
| NSA | Count leading sign bits | RRR | Miscellaneous Operations |
| NSAU | Count leading zero bits (unsigned) | RRR | Miscellaneous Operations |
| SEXT | Sign extend | RRR | Miscellaneous Operations |

### 5.23 Boolean Operation Instructions

| Instruction | Operation | Format | Option |
|------|------|------|------|
| ANDB | Boolean AND | RRR | Boolean |
| ORB | Boolean OR | RRR | Boolean |
| XORB | Boolean XOR | RRR | Boolean |
| ANDBC | Boolean AND-complement | RRR | Boolean |
| ORBC | Boolean OR-complement | RRR | Boolean |
| ANY4 | Any of 4 booleans true | RRR | Boolean |
| ALL4 | All of 4 booleans true | RRR | Boolean |
| ANY8 | Any of 8 booleans true | RRR | Boolean |
| ALL8 | All of 8 booleans true | RRR | Boolean |

---

## 6. Instruction Details

### 6.1 Arithmetic Instructions

#### ADD - Addition

**Syntax:** `ADD ar, as, at`

**Operation:** `AR[r] ← AR[s] + AR[t]`

**Description:** Adds the contents of address registers as and at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### ADDI - Immediate Addition

**Syntax:** `ADDI ar, as, -128..127`

**Operation:** `AR[r] ← AR[s] + sign_extend(imm8)`

**Description:** Adds the contents of address register as to an 8-bit signed immediate, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### ADDMI - Immediate Addition (16-bit)

**Syntax:** `ADDMI ar, as, -32768..32512` (multiple of 256)

**Operation:** `AR[r] ← AR[s] + sign_extend(imm8 << 8)`

**Description:** Adds the contents of address register as to a 16-bit signed immediate (a multiple of 256), and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### ADDX2 - Shift-Left-1 Addition

**Syntax:** `ADDX2 ar, as, at`

**Operation:** `AR[r] ← (AR[s] << 1) + AR[t]`

**Description:** Shifts address register as left by 1 bit, adds at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### ADDX4 - Shift-Left-2 Addition

**Syntax:** `ADDX4 ar, as, at`

**Operation:** `AR[r] ← (AR[s] << 2) + AR[t]`

**Description:** Shifts address register as left by 2 bits, adds at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### ADDX8 - Shift-Left-3 Addition

**Syntax:** `ADDX8 ar, as, at`

**Operation:** `AR[r] ← (AR[s] << 3) + AR[t]`

**Description:** Shifts address register as left by 3 bits, adds at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SUB - Subtraction

**Syntax:** `SUB ar, as, at`

**Operation:** `AR[r] ← AR[s] - AR[t]`

**Description:** Subtracts the contents of at from address register as, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SUBX2 - Shift-Left-1 Subtraction

**Syntax:** `SUBX2 ar, as, at`

**Operation:** `AR[r] ← (AR[s] << 1) - AR[t]`

**Description:** Shifts address register as left by 1 bit, subtracts at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SUBX4 - Shift-Left-2 Subtraction

**Syntax:** `SUBX4 ar, as, at`

**Operation:** `AR[r] ← (AR[s] << 2) - AR[t]`

**Description:** Shifts address register as left by 2 bits, subtracts at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SUBX8 - Shift-Left-3 Subtraction

**Syntax:** `SUBX8 ar, as, at`

**Operation:** `AR[r] ← (AR[s] << 3) - AR[t]`

**Description:** Shifts address register as left by 3 bits, subtracts at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### NEG - Negate

**Syntax:** `NEG ar, at`

**Operation:** `AR[r] ← 0 - AR[t]`

**Description:** Negates the contents of address register at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### ABS - Absolute Value

**Syntax:** `ABS ar, at`

**Operation:** `AR[r] ← |AR[t]|`

**Description:** Takes the absolute value of the contents of address register at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

### 6.2 Logical Instructions

#### AND - Bitwise AND

**Syntax:** `AND ar, as, at`

**Operation:** `AR[r] ← AR[s] and AR[t]`

**Description:** Performs a bitwise AND of the contents of address registers as and at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### OR - Bitwise OR

**Syntax:** `OR ar, as, at`

**Operation:** `AR[r] ← AR[s] or AR[t]`

**Description:** Performs a bitwise OR of the contents of address registers as and at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### XOR - Bitwise XOR

**Syntax:** `XOR ar, as, at`

**Operation:** `AR[r] ← AR[s] xor AR[t]`

**Description:** Performs a bitwise XOR of the contents of address registers as and at, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

### 6.3 Shift Instructions

#### SLL - Logical Shift Left

**Syntax:** `SLL ar, as`

**Operation:** `AR[r] ← AR[s] << SAR`

**Description:** Logically shifts the contents of address register as left by SAR bits, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SRL - Logical Shift Right

**Syntax:** `SRL ar, at`

**Operation:** `AR[r] ← AR[t] >> SAR` (logical shift right)

**Description:** Logically shifts the contents of address register at right by SAR bits, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SRA - Arithmetic Shift Right

**Syntax:** `SRA ar, at`

**Operation:** `AR[r] ← AR[t] >> SAR` (arithmetic shift right)

**Description:** Arithmetically shifts the contents of address register at right by SAR bits, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SLLI - Immediate Logical Shift Left

**Syntax:** `SLLI ar, as, 0..31`

**Operation:** `AR[r] ← AR[s] << imm5`

**Description:** Logically shifts the contents of address register as left by imm5 bits, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SRLI - Immediate Logical Shift Right

**Syntax:** `SRLI ar, as, 0..31`

**Operation:** `AR[r] ← AR[s] >> imm5` (logical shift right)

**Description:** Logically shifts the contents of address register as right by imm5 bits, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SRAI - Immediate Arithmetic Shift Right

**Syntax:** `SRAI ar, as, 0..31`

**Operation:** `AR[r] ← AR[s] >> imm5` (arithmetic shift right)

**Description:** Arithmetically shifts the contents of address register as right by imm5 bits, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SRC - Funnel (Rotate) Shift

**Syntax:** `SRC ar, as, at`

**Operation:** `AR[r] ← (AR[s] << (32-SAR)) or (AR[t] >> SAR)`

**Description:** Concatenates AR[s] and AR[t] and shifts the result right by SAR bits, writing it to ar. Used for multi-word shifts.

**Exceptions:** EveryInstR Group

---

#### SSR - Set Shift Amount for Right Shift

**Syntax:** `SSR as`

**Operation:** `SAR ← AR[s]4..0`

**Description:** Writes the low 5 bits of address register as into SAR, for use by a subsequent right-shift operation.

**Exceptions:** EveryInstR Group

---

#### SSL - Set Shift Amount for Left Shift

**Syntax:** `SSL as`

**Operation:** `SAR ← 32 - AR[s]4..0`

**Description:** Writes 32 minus the low 5 bits of address register as into SAR, for use by a subsequent left-shift operation.

**Exceptions:** EveryInstR Group

---

#### SSA8L - Set 8-bit Left-Shift Alignment

**Syntax:** `SSA8L as`

**Operation:** `SAR ← (AR[s] and 3) << 3`

**Description:** Writes the low 2 bits of address register as, multiplied by 8, into SAR, for byte-aligned left shifts.

**Exceptions:** EveryInstR Group

---

#### SSA8B - Set 8-bit Right-Shift Alignment

**Syntax:** `SSA8B as`

**Operation:** `SAR ← 32 - ((AR[s] and 3) << 3)`

**Description:** Writes 32 minus the low 2 bits of address register as, multiplied by 8, into SAR, for byte-aligned right shifts.

**Exceptions:** EveryInstR Group

---

#### SSAI - Set Immediate Shift Alignment

**Syntax:** `SSAI 0..31`

**Operation:** `SAR ← 32 - imm5`

**Description:** Writes 32 minus a 5-bit immediate into SAR.

**Exceptions:** EveryInstR Group

---

### 6.4 Multiply Instructions

#### MULL - 32-bit Multiply (Low 32 Bits)

**Syntax:** `MULL ar, as, at`

**Operation:** `AR[r] ← (AR[s] × AR[t])31..0`

**Description:** Multiplies the contents of address registers as and at, and writes the low 32 bits of the result to ar.

**Exceptions:** EveryInstR Group

---

#### MULUH - Unsigned 32-bit Multiply (High 32 Bits)

**Syntax:** `MULUH ar, as, at`

**Operation:** `AR[r] ← (AR[s] × AR[t])63..32` (unsigned)

**Description:** Multiplies the contents of address registers as and at as unsigned values, and writes the high 32 bits of the result to ar.

**Exceptions:** EveryInstR Group

---

#### MULSH - Signed 32-bit Multiply (High 32 Bits)

**Syntax:** `MULSH ar, as, at`

**Operation:** `AR[r] ← (AR[s] × AR[t])63..32` (signed)

**Description:** Multiplies the contents of address registers as and at as signed values, and writes the high 32 bits of the result to ar.

**Exceptions:** EveryInstR Group

---

#### MUL16U - Unsigned 16-bit Multiply

**Syntax:** `MUL16U ar, as, at`

**Operation:** `AR[r] ← (AR[s]15..0 × AR[t]15..0)`

**Description:** Multiplies the low 16 bits of address registers as and at as unsigned values, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### MUL16S - Signed 16-bit Multiply

**Syntax:** `MUL16S ar, as, at`

**Operation:** `AR[r] ← (AR[s]15..0 × AR[t]15..0)` (signed)

**Description:** Multiplies the low 16 bits of address registers as and at as signed values, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

### 6.5 Divide Instructions

#### QUOU - Unsigned Divide

**Syntax:** `QUOU ar, as, at`

**Operation:** `AR[r] ← AR[s] ÷ AR[t]` (unsigned)

**Description:** Divides the contents of address register as by at (unsigned), and writes the quotient to ar.

**Exceptions:**
- EveryInstR Group
- IntegerDivideByZeroCause (divisor is zero)

---

#### QUOS - Signed Divide

**Syntax:** `QUOS ar, as, at`

**Operation:** `AR[r] ← AR[s] ÷ AR[t]` (signed)

**Description:** Divides the contents of address register as by at (signed), and writes the quotient to ar.

**Exceptions:**
- EveryInstR Group
- IntegerDivideByZeroCause (divisor is zero)

---

#### REMU - Unsigned Remainder

**Syntax:** `REMU ar, as, at`

**Operation:** `AR[r] ← AR[s] mod AR[t]` (unsigned)

**Description:** Divides the contents of address register as by at (unsigned), and writes the remainder to ar.

**Exceptions:**
- EveryInstR Group
- IntegerDivideByZeroCause (divisor is zero)

---

#### REMS - Signed Remainder

**Syntax:** `REMS ar, as, at`

**Operation:** `AR[r] ← AR[s] mod AR[t]` (signed)

**Description:** Divides the contents of address register as by at (signed), and writes the remainder to ar.

**Exceptions:**
- EveryInstR Group
- IntegerDivideByZeroCause (divisor is zero)

---

### 6.6 Load Instructions

#### L8UI - Load Unsigned 8-bit

**Syntax:** `L8UI ar, as, 0..255`

**Operation:** `AR[r] ← zero_extend(Mem[AR[s] + imm8]7..0)`

**Description:** Loads an unsigned 8-bit value from memory address as+imm8, zero-extends it to 32 bits, and writes it to ar.

**Exceptions:**
- EveryInstR Group
- GenExcep(LoadStoreErrorCause)
- GenExcep(LoadStoreAlignmentCause)

---

#### L16UI - Load Unsigned 16-bit

**Syntax:** `L16UI ar, as, 0..510` (2-byte aligned)

**Operation:** `AR[r] ← zero_extend(Mem[AR[s] + (imm8 << 1)]15..0)`

**Description:** Loads an unsigned 16-bit value from memory address as+(imm8<<1), zero-extends it to 32 bits, and writes it to ar.

**Exceptions:**
- EveryInstR Group
- GenExcep(LoadStoreErrorCause)
- GenExcep(LoadStoreAlignmentCause)

---

#### L16SI - Load Signed 16-bit

**Syntax:** `L16SI ar, as, 0..510` (2-byte aligned)

**Operation:** `AR[r] ← sign_extend(Mem[AR[s] + (imm8 << 1)]15..0)`

**Description:** Loads a signed 16-bit value from memory address as+(imm8<<1), sign-extends it to 32 bits, and writes it to ar.

**Exceptions:**
- EveryInstR Group
- GenExcep(LoadStoreErrorCause)
- GenExcep(LoadStoreAlignmentCause)

---

#### L32I - Load 32-bit

**Syntax:** `L32I ar, as, 0..1020` (4-byte aligned)

**Operation:** `AR[r] ← Mem[AR[s] + (imm8 << 2)]`

**Description:** Loads a 32-bit value from memory address as+(imm8<<2), and writes it to ar.

**Exceptions:**
- EveryInstR Group
- GenExcep(LoadStoreErrorCause)
- GenExcep(LoadStoreAlignmentCause)

---

#### L32R - Load from Literal Pool

**Syntax:** `L32R ar, label`

**Operation:** `AR[r] ← Mem[(PC + 3)4..2 || 00 + imm16]`

**Description:** Loads a 32-bit constant from the PC-relative literal pool, and writes it to ar.

**Exceptions:** EveryInstR Group

---

### 6.7 Store Instructions

#### S8I - Store 8-bit

**Syntax:** `S8I ar, as, 0..255`

**Operation:** `Mem[AR[s] + imm8]7..0 ← AR[r]7..0`

**Description:** Stores the low 8 bits of address register ar to memory address as+imm8.

**Exceptions:**
- EveryInstR Group
- GenExcep(LoadStoreErrorCause)

---

#### S16I - Store 16-bit

**Syntax:** `S16I ar, as, 0..510` (2-byte aligned)

**Operation:** `Mem[AR[s] + (imm8 << 1)]15..0 ← AR[r]15..0`

**Description:** Stores the low 16 bits of address register ar to memory address as+(imm8<<1).

**Exceptions:**
- EveryInstR Group
- GenExcep(LoadStoreErrorCause)
- GenExcep(LoadStoreAlignmentCause)

---

#### S32I - Store 32-bit

**Syntax:** `S32I ar, as, 0..1020` (4-byte aligned)

**Operation:** `Mem[AR[s] + (imm8 << 2)] ← AR[r]`

**Description:** Stores the contents of address register ar to memory address as+(imm8<<2).

**Exceptions:**
- EveryInstR Group
- GenExcep(LoadStoreErrorCause)
- GenExcep(LoadStoreAlignmentCause)

---

### 6.8 Branch Instructions

#### J - Unconditional Jump

**Syntax:** `J target`

**Operation:** `PC ← PC + sign_extend(imm18) + 4`

**Description:** Jumps to the PC-relative address target.

**Exceptions:** EveryInst Group

---

#### BEQ - Branch if Equal

**Syntax:** `BEQ as, at, label`

**Operation:** `if AR[s] == AR[t] then PC ← PC + sign_extend(imm8) + 4`

**Description:** Branches to label if as equals at.

**Exceptions:** EveryInst Group

---

#### BNE - Branch if Not Equal

**Syntax:** `BNE as, at, label`

**Operation:** `if AR[s] != AR[t] then PC ← PC + sign_extend(imm8) + 4`

**Description:** Branches to label if as does not equal at.

**Exceptions:** EveryInst Group

---

#### BLT - Branch if Less Than (Signed)

**Syntax:** `BLT as, at, label`

**Operation:** `if AR[s] < AR[t] (signed) then PC ← PC + sign_extend(imm8) + 4`

**Description:** Branches to label if as is less than at (signed comparison).

**Exceptions:** EveryInst Group

---

#### BGE - Branch if Greater-or-Equal (Signed)

**Syntax:** `BGE as, at, label`

**Operation:** `if AR[s] >= AR[t] (signed) then PC ← PC + sign_extend(imm8) + 4`

**Description:** Branches to label if as is greater than or equal to at (signed comparison).

**Exceptions:** EveryInst Group

---

#### BLTU - Branch if Less Than (Unsigned)

**Syntax:** `BLTU as, at, label`

**Operation:** `if AR[s] < AR[t] (unsigned) then PC ← PC + sign_extend(imm8) + 4`

**Description:** Branches to label if as is less than at (unsigned comparison).

**Exceptions:** EveryInst Group

---

#### BGEU - Branch if Greater-or-Equal (Unsigned)

**Syntax:** `BGEU as, at, label`

**Operation:** `if AR[s] >= AR[t] (unsigned) then PC ← PC + sign_extend(imm8) + 4`

**Description:** Branches to label if as is greater than or equal to at (unsigned comparison).

**Exceptions:** EveryInst Group

---

#### BEQZ - Branch if Equal to Zero

**Syntax:** `BEQZ as, label`

**Operation:** `if AR[s] == 0 then PC ← PC + sign_extend(imm12) + 4`

**Description:** Branches to label if as equals 0.

**Exceptions:** EveryInst Group

---

#### BNEZ - Branch if Not Equal to Zero

**Syntax:** `BNEZ as, label`

**Operation:** `if AR[s] != 0 then PC ← PC + sign_extend(imm12) + 4`

**Description:** Branches to label if as does not equal 0.

**Exceptions:** EveryInst Group

---

### 6.9 Call and Return Instructions

#### CALL0 - Call (No Window)

**Syntax:** `CALL0 target`

**Operation:**
```
AR[0] ← PC + 3
PC ← PC + sign_extend(imm18) + 4
```

**Description:** Saves the return address to a0, and jumps to target.

**Exceptions:** EveryInst Group

---

#### CALL4 - Call (4-Register Window)

**Syntax:** `CALL4 target`

**Operation:**
```
AR[0] ← (PC + 3) and 0x3FFFFFFF or 0x01
PS.CALLINC ← 1
WindowCheck(1, 1, 1)
PC ← PC + sign_extend(imm18) + 4
```

**Description:** Saves the return address to a0 (encoding a window size of 4), checks for window overflow, and jumps to target.

**Exceptions:**
- EveryInst Group
- GenExcep(WindowOverflowCause)

---

#### CALL8 - Call (8-Register Window)

**Syntax:** `CALL8 target`

**Operation:**
```
AR[0] ← (PC + 3) and 0x3FFFFFFF or 0x02
PS.CALLINC ← 2
WindowCheck(2, 2, 2)
PC ← PC + sign_extend(imm18) + 4
```

**Description:** Saves the return address to a0 (encoding a window size of 8), checks for window overflow, and jumps to target.

**Exceptions:**
- EveryInst Group
- GenExcep(WindowOverflowCause)

---

#### CALL12 - Call (12-Register Window)

**Syntax:** `CALL12 target`

**Operation:**
```
AR[0] ← (PC + 3) and 0x3FFFFFFF or 0x03
PS.CALLINC ← 3
WindowCheck(3, 3, 3)
PC ← PC + sign_extend(imm18) + 4
```

**Description:** Saves the return address to a0 (encoding a window size of 12), checks for window overflow, and jumps to target.

**Exceptions:**
- EveryInst Group
- GenExcep(WindowOverflowCause)

---

#### RET - Return

**Syntax:** `RET`

**Operation:** `PC ← AR[0]`

**Description:** Returns to the address saved in a0.

**Exceptions:** EveryInst Group

---

#### RETW - Windowed Return

**Syntax:** `RETW`

**Operation:**
```
if (AR[0]1..0 == 0) then
  PC ← AR[0]
else
  PS.CALLINC ← AR[0]1..0
  WindowUnderflow()
  PC ← AR[0] and 0x3FFFFFFF
endif
```

**Description:** Returns based on the window information encoded in a0, potentially handling a window underflow.

**Exceptions:**
- EveryInst Group
- GenExcep(WindowUnderflowCause)

---

### 6.10 Loop Instructions

#### LOOP - Zero-Overhead Loop

**Syntax:** `LOOP as, label`

**Operation:**
```
LBEG ← PC + 3
LEND ← PC + 4 + zero_extend(imm8)
LCOUNT ← AR[s] - 1
```
When LCOUNT=0, the loop iterates 2^32 times.

**Description:** Sets up a zero-overhead loop; the iteration count is specified by as, and the loop body runs from just after this instruction to label.

**Exceptions:** EveryInst Group

**Restrictions:**
- The loop body may be at most 256 bytes (imm8 is an 8-bit unsigned offset)
- LEND must be greater than LBEG
- The loop body may not contain jump instructions

---

#### LOOPGTZ - Loop if Greater Than Zero

**Syntax:** `LOOPGTZ as, label`

**Operation:**
```
LBEG ← PC + 3
LEND ← PC + 4 + zero_extend(imm8)
LCOUNT ← AR[s] - 1
if AR[s] <= 0 then
  PC ← PC + 4 + zero_extend(imm8)
endif
```

**Description:** If as is greater than 0, sets up a zero-overhead loop; otherwise skips the loop body. Note that LBEG/LEND/LCOUNT are still set even when the loop is skipped (they simply won't be executed).

**Exceptions:** EveryInst Group

---

#### LOOPNEZ - Loop if Not Equal to Zero

**Syntax:** `LOOPNEZ as, label`

**Operation:**
```
LBEG ← PC + 3
LEND ← PC + 4 + zero_extend(imm8)
LCOUNT ← AR[s] - 1
if AR[s] == 0 then
  PC ← PC + 4 + zero_extend(imm8)
endif
```

**Description:** If as is not equal to 0, sets up a zero-overhead loop; otherwise skips the loop body. Note that LBEG/LEND/LCOUNT are still set even when the loop is skipped (they simply won't be executed).

**Exceptions:** EveryInst Group

---

### 6.11 Special Register Access Instructions

#### RSR.* - Read Special Register

**Syntax:** `RSR.* at` or `RSR at, *`

**Operation:** `AR[t] ← SpecialRegister[*]`

**Description:** Reads the contents of the specified special register into address register at.

**Exceptions:**
- EveryInstR Group
- GenExcep(IllegalInstructionCause) (if the register is not configured)
- GenExcep(PrivilegedCause) (if sr >= 64 and not in privileged mode)

---

#### WSR.* - Write Special Register

**Syntax:** `WSR.* at` or `WSR at, *`

**Operation:** `SpecialRegister[*] ← AR[t]`

**Description:** Writes the contents of address register at into the specified special register.

**Exceptions:**
- EveryInstR Group
- GenExcep(IllegalInstructionCause) (if the register is not configured)
- GenExcep(PrivilegedCause) (if sr >= 64 and not in privileged mode)

---

#### XSR.* - Exchange Special Register

**Syntax:** `XSR.* at` or `XSR at, *`

**Operation:**
```
temp ← SpecialRegister[*]
SpecialRegister[*] ← AR[t]
AR[t] ← temp
```

**Description:** Exchanges the contents of address register at with the specified special register.

**Exceptions:**
- EveryInstR Group
- GenExcep(IllegalInstructionCause) (if the register is not configured)
- GenExcep(PrivilegedCause) (if sr >= 64 and not in privileged mode)

---

### 6.12 Synchronization Instructions

#### ISYNC - Instruction Sync

**Syntax:** `ISYNC`

**Operation:** Synchronizes the instruction stream

**Description:** Ensures prior instructions are visible to subsequent instructions; used for self-modifying code and after special-register modifications.

**Exceptions:** EveryInst Group

---

#### DSYNC - Data Sync

**Syntax:** `DSYNC`

**Operation:** Synchronizes the data stream

**Description:** Ensures prior data operations are visible to subsequent data operations; used for memory-mapped I/O and after TLB modifications.

**Exceptions:** EveryInst Group

---

#### RSYNC - Register Sync

**Syntax:** `RSYNC`

**Operation:** Synchronizes register access

**Description:** Ensures prior special-register writes are visible to subsequent special-register reads.

**Exceptions:** EveryInst Group

---

#### ESYNC - Execute Sync

**Syntax:** `ESYNC`

**Operation:** Execute synchronization

**Description:** Ensures prior special-register writes are visible to all subsequent operations; the strongest synchronization instruction.

**Exceptions:** EveryInst Group

---

#### MEMW - Memory Wait

**Syntax:** `MEMW`

**Operation:** Waits for memory operations to complete

**Description:** Waits for all outstanding memory operations to complete; used for multiprocessor synchronization.

**Exceptions:** EveryInst Group

---

### 6.13 Floating-Point Instructions

#### ADD.S - Floating-Point Add

**Syntax:** `ADD.S fr, fs, ft`

**Operation:** `FR[r] ← FR[s] +s FR[t]`

**Description:** Adds the contents of floating-point registers fs and ft, and writes the result to fr.

**Exceptions:**
- EveryInstR Group
- GenExcep(Coprocessor0Disabled)

---

#### SUB.S - Floating-Point Subtract

**Syntax:** `SUB.S fr, fs, ft`

**Operation:** `FR[r] ← FR[s] -s FR[t]`

**Description:** Subtracts the contents of ft from floating-point register fs, and writes the result to fr.

**Exceptions:**
- EveryInstR Group
- GenExcep(Coprocessor0Disabled)

---

#### MUL.S - Floating-Point Multiply

**Syntax:** `MUL.S fr, fs, ft`

**Operation:** `FR[r] ← FR[s] ×s FR[t]`

**Description:** Multiplies the contents of floating-point registers fs and ft, and writes the result to fr.

**Exceptions:**
- EveryInstR Group
- GenExcep(Coprocessor0Disabled)

---

#### RFR - Read from AR into FR

**Syntax:** `RFR fr, as`

**Operation:** `FR[r] ← AR[s]`

**Description:** Moves the contents of address register as into floating-point register fr; not an arithmetic operation.

**Exceptions:**
- EveryInstR Group
- GenExcep(Coprocessor0Disabled)

---

#### WFR - Write from FR into AR

**Syntax:** `WFR ar, fs`

**Operation:** `AR[r] ← FR[s]`

**Description:** Moves the contents of floating-point register fs into address register ar; not an arithmetic operation.

**Exceptions:**
- EveryInstR Group
- GenExcep(Coprocessor0Disabled)

---

### 6.14 Exception and Interrupt Instructions

#### RFE - Return from Exception

**Syntax:** `RFE`

**Operation:**
```
PS ← EPS[1]
PC ← EPC[1]
```

**Description:** Returns from an exception, restoring the program state and program counter.

**Exceptions:**
- EveryInst Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### RFI - Return from Interrupt

**Syntax:** `RFI level` (level = 2..7)

**Operation:**
```
PS ← EPS[level]
PC ← EPC[level]
```

**Description:** Returns from an interrupt, restoring the program state and program counter.

**Exceptions:**
- EveryInst Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### SYSCALL - System Call

**Syntax:** `SYSCALL`

**Operation:**
```
EPC[1] ← PC
EXCCAUSE ← SyscallCause
PC ← UserExceptionVector
```

**Description:** Triggers a system-call exception.

**Exceptions:**
- EveryInst Group
- GenExcep(SyscallCause)

---

#### WAITI - Wait for Interrupt

**Syntax:** `WAITI 0..15`

**Operation:**
```
PS.INTLEVEL ← imm4
// May enter a low-power mode while waiting for an interrupt
```

**Description:** Sets the interrupt level and waits for an interrupt; typically used in idle loops to reduce power consumption.

**Exceptions:**
- EveryInst Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### RSIL - Read and Set Interrupt Level

**Syntax:** `RSIL ar, 0..15`

**Operation:**
```
AR[r] ← PS
PS.INTLEVEL ← imm4
```

**Description:** Reads the current program state into ar, and sets a new interrupt level.

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### BREAK - Breakpoint

**Syntax:** `BREAK 0..15, 0..15`

**Operation:** Triggers a debug exception

**Description:** Triggers a debug exception, used for breakpoint debugging.

**Exceptions:**
- EveryInst Group
- GenExcep(DebugCause)

---

### 6.15 Cache Instructions

#### IHI - Instruction Cache Hit Invalidate

**Syntax:** `IHI as, 0..240` (16-byte aligned)

**Operation:** Invalidates the line that hits in the instruction cache

**Description:** Invalidates the instruction-cache line matching address as+(imm4<<4).

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### III - Instruction Cache Index Invalidate

**Syntax:** `III as, 0..240` (16-byte aligned)

**Operation:** Invalidates the line at the specified index in the instruction cache

**Description:** Invalidates the instruction-cache line at index as+(imm4<<4).

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### DHI - Data Cache Hit Invalidate

**Syntax:** `DHI as, 0..240` (16-byte aligned)

**Operation:** Invalidates the line that hits in the data cache

**Description:** Invalidates the data-cache line matching address as+(imm4<<4).

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### DII - Data Cache Index Invalidate

**Syntax:** `DII as, 0..240` (16-byte aligned)

**Operation:** Invalidates the line at the specified index in the data cache

**Description:** Invalidates the data-cache line at index as+(imm4<<4).

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### DWB - Data Cache Writeback

**Syntax:** `DWB as, 0..240` (16-byte aligned)

**Operation:** Writes back the line that hits in the data cache

**Description:** Writes the dirty data-cache line matching address as+(imm4<<4) back to memory.

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### DWBI - Data Cache Writeback and Invalidate

**Syntax:** `DWBI as, 0..240` (16-byte aligned)

**Operation:** Writes back and invalidates the line that hits in the data cache

**Description:** Writes the dirty data-cache line matching address as+(imm4<<4) back to memory, then invalidates that line.

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

### 6.16 TLB Instructions

#### RITLB0 - Read Instruction TLB Entry 0

**Syntax:** `RITLB0 at, as`

**Operation:** `AR[t] ← InstTLB[0][AR[s]]`

**Description:** Reads the entry specified by as from instruction TLB way 0 into at.

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### RITLB1 - Read Instruction TLB Entry 1

**Syntax:** `RITLB1 at, as`

**Operation:** `AR[t] ← InstTLB[1][AR[s]]`

**Description:** Reads the entry specified by as from instruction TLB way 1 into at.

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### RDTLB0 - Read Data TLB Entry 0

**Syntax:** `RDTLB0 at, as`

**Operation:** `AR[t] ← DataTLB[0][AR[s]]`

**Description:** Reads the entry specified by as from data TLB way 0 into at.

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### RDTLB1 - Read Data TLB Entry 1

**Syntax:** `RDTLB1 at, as`

**Operation:** `AR[t] ← DataTLB[1][AR[s]]`

**Description:** Reads the entry specified by as from data TLB way 1 into at.

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### WITLB - Write Instruction TLB Entry

**Syntax:** `WITLB at, as`

**Operation:** `InstTLB[way][index] ← AR[t]`

**Description:** Writes the contents of at into the instruction-TLB entry specified by as.

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

#### WDTLB - Write Data TLB Entry

**Syntax:** `WDTLB at, as`

**Operation:** `DataTLB[way][index] ← AR[t]`

**Description:** Writes the contents of at into the data-TLB entry specified by as.

**Exceptions:**
- EveryInstR Group
- GenExcep(PrivilegedCause) (if not in privileged mode)

---

### 6.17 Miscellaneous Instructions

#### MOV - Move

**Syntax:** `MOV ar, as`

**Operation:** `AR[r] ← AR[s]`

**Description:** Copies the contents of address register as to ar.

**Exceptions:** EveryInstR Group

---

#### MOVI - Immediate Move

**Syntax:** `MOVI ar, -2048..2047`

**Operation:** `AR[r] ← sign_extend(imm12)`

**Description:** Sign-extends a 12-bit signed immediate to 32 bits and writes it to ar.

**Exceptions:** EveryInstR Group

---

#### MOVEQZ - Move if Equal to Zero

**Syntax:** `MOVEQZ ar, as, at`

**Operation:** `if AR[t] == 0 then AR[r] ← AR[s]`

**Description:** If at equals 0, copies the contents of as to ar.

**Exceptions:** EveryInstR Group

---

#### MOVNEZ - Move if Not Equal to Zero

**Syntax:** `MOVNEZ ar, as, at`

**Operation:** `if AR[t] != 0 then AR[r] ← AR[s]`

**Description:** If at does not equal 0, copies the contents of as to ar.

**Exceptions:** EveryInstR Group

---

#### MOVLTZ - Move if Less Than Zero

**Syntax:** `MOVLTZ ar, as, at`

**Operation:** `if AR[t] < 0 then AR[r] ← AR[s]`

**Description:** If at is less than 0, copies the contents of as to ar.

**Exceptions:** EveryInstR Group

---

#### MOVGEZ - Move if Greater-or-Equal to Zero

**Syntax:** `MOVGEZ ar, as, at`

**Operation:** `if AR[t] >= 0 then AR[r] ← AR[s]`

**Description:** If at is greater than or equal to 0, copies the contents of as to ar.

**Exceptions:** EveryInstR Group

---

#### ENTRY - Function Entry

**Syntax:** `ENTRY as, 0..32767` (8-byte aligned)

**Operation:**
```
AR[PS.CALLINC||s1..0] ← AR[s] - (imm12 << 3)
WindowBase ← WindowBase + PS.CALLINC
WindowStart[WindowBase] ← 1
```

**Description:** Rotates the register window and allocates stack space, used at function entry. After rotation, the callee's sp (a1) points to the new stack frame, and the caller's a0-a3 are preserved in a register region not visible to the callee.

**Exceptions:**
- EveryInst Group
- GenExcep(WindowOverflowCause)

---

#### NOP - No Operation

**Syntax:** `NOP`

**Operation:** None

**Description:** No operation; produces no effect.

**Exceptions:** EveryInst Group

---

### 6.18 Miscellaneous Operation Instructions

#### MIN - Minimum

**Syntax:** `MIN ar, as, at`

**Operation:** `AR[r] ← min(AR[s], AR[t])` (signed)

**Description:** Takes the smaller of as and at (signed comparison), and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### MAX - Maximum

**Syntax:** `MAX ar, as, at`

**Operation:** `AR[r] ← max(AR[s], AR[t])` (signed)

**Description:** Takes the larger of as and at (signed comparison), and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### MINU - Unsigned Minimum

**Syntax:** `MINU ar, as, at`

**Operation:** `AR[r] ← min(AR[s], AR[t])` (unsigned)

**Description:** Takes the smaller of as and at (unsigned comparison), and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### MAXU - Unsigned Maximum

**Syntax:** `MAXU ar, as, at`

**Operation:** `AR[r] ← max(AR[s], AR[t])` (unsigned)

**Description:** Takes the larger of as and at (unsigned comparison), and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### NSA - Count Leading Sign Bits

**Syntax:** `NSA ar, as`

**Operation:** `AR[r] ← count_leading_zeros(AR[s])`

**Description:** Counts the number of leading sign bits in as (from the most significant bit), and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### NSAU - Count Leading Zero Bits (Unsigned)

**Syntax:** `NSAU ar, as`

**Operation:** `AR[r] ← count_leading_zeros(AR[s])` (unsigned)

**Description:** Counts the number of leading zero bits in as (unsigned), and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### CLAMPS - Signed Clamp

**Syntax:** `CLAMPS ar, as, 0..31`

**Operation:** `AR[r] ← clamp(AR[s], -(1 << imm4), (1 << imm4) - 1)`

**Description:** Clamps as to the specified range, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

#### SEXT - Sign Extend

**Syntax:** `SEXT ar, as, 7..22`

**Operation:** `AR[r] ← sign_extend(AR[s]imm4..0)`

**Description:** Sign-extends the specified bit of as to 32 bits, and writes the result to ar.

**Exceptions:** EveryInstR Group

---

## 7. Exceptions and Interrupts

### 7.1 Exception Vectors

> **Note:** The exact vector address offsets are determined by the processor configuration; the table below shows the offsets for a typical Xtensa configuration. Under the Relocatable Vector Option, the static group (ResetVector/MemoryErrorVector) uses a fixed absolute address, while the dynamic group (all other vectors) is based at VECBASE with a configurable offset.

| Vector | Address (typical offset) | Description |
|------|------|------|
| ResetVector | (fixed at configuration) + 0x00 | Reset vector |
| MemoryErrorVector | (fixed at configuration) + 0x60 | Memory error vector |
| UserExceptionVector | VECBASE + 0x40 | User exception vector |
| KernelExceptionVector | VECBASE + 0x50 | Kernel exception vector |
| DoubleExceptionVector | VECBASE + 0x70 | Double exception vector |
| WindowOverflow4 | VECBASE + 0x80 | 4-register window overflow |
| WindowUnderflow4 | VECBASE + 0x90 | 4-register window underflow |
| WindowOverflow8 | VECBASE + 0xA0 | 8-register window overflow |
| WindowUnderflow8 | VECBASE + 0xB0 | 8-register window underflow |
| WindowOverflow12 | VECBASE + 0xC0 | 12-register window overflow |
| WindowUnderflow12 | VECBASE + 0xD0 | 12-register window underflow |
| InterruptVector2..7 | VECBASE + 0x100 + (level-2)×0x10 | Interrupt vectors |

### 7.2 Exception Causes (EXCCAUSE)

| Value | Name | Description |
|----|------|------|
| 0 | IllegalInstructionCause | Illegal instruction |
| 1 | SyscallCause | System call |
| 2 | InstructionFetchErrorCause | Instruction fetch error |
| 3 | LoadStoreErrorCause | Load/store error |
| 4 | Level1InterruptCause | Level-1 interrupt |
| 5 | AllocaCause | Stack allocation error |
| 6 | IntegerDivideByZeroCause | Integer divide by zero |
| 8 | PrivilegedCause | Privilege violation |
| 9 | LoadStoreAlignmentCause | Load/store alignment error |
| 12 | InstrPIFDataErrorCause | Instruction PIF data error |
| 13 | LoadStorePIFDataErrorCause | Load/store PIF data error |
| 14 | InstrPIFAddrErrorCause | Instruction PIF address error |
| 15 | LoadStorePIFAddrErrorCause | Load/store PIF address error |
| 16 | InstTLBMissCause | Instruction TLB miss |
| 17 | InstTLBMultiHitCause | Instruction TLB multi-hit |
| 18 | InstFetchPrivilegeCause | Instruction fetch privilege error |
| 20 | InstFetchProhibitedCause | Instruction fetch prohibited |
| 24 | LoadStoreTLBMissCause | Load/store TLB miss |
| 25 | LoadStoreTLBMultiHitCause | Load/store TLB multi-hit |
| 26 | LoadStorePrivilegeCause | Load/store privilege error |
| 28 | LoadProhibitedCause | Load prohibited |
| 29 | StoreProhibitedCause | Store prohibited |
| 32 | Coprocessor0DisabledCause | Coprocessor 0 disabled |
| 33 | Coprocessor1DisabledCause | Coprocessor 1 disabled |
| 34 | Coprocessor2DisabledCause | Coprocessor 2 disabled |
| 35 | Coprocessor3DisabledCause | Coprocessor 3 disabled |
| 36 | Coprocessor4DisabledCause | Coprocessor 4 disabled |
| 37 | Coprocessor5DisabledCause | Coprocessor 5 disabled |
| 38 | Coprocessor6DisabledCause | Coprocessor 6 disabled |
| 39 | Coprocessor7DisabledCause | Coprocessor 7 disabled |

---

## 8. Memory Management

### 8.1 TLB Entry Formats

#### Instruction TLB Read Format (RITLB)

| Bits | Field | Description |
|----|------|------|
| 31-12 | VPN | Virtual page number |
| 11-0 | - | Reserved |

#### Instruction TLB Write Format (WITLB)

| Bits | Field | Description |
|----|------|------|
| 31-12 | PPN | Physical page number |
| 11-8 | CA | Cache attribute |
| 7-4 | SR | Status/permissions |
| 3-0 | RING | Ring number |

#### Data TLB Read Format (RDTLB)

| Bits | Field | Description |
|----|------|------|
| 31-12 | VPN | Virtual page number |
| 11-0 | ASID | Address space identifier |

#### Data TLB Write Format (WDTLB)

| Bits | Field | Description |
|----|------|------|
| 31-12 | PPN | Physical page number |
| 11-8 | CA | Cache attribute |
| 7-4 | SR | Status/permissions |
| 3-0 | RING | Ring number |

### 8.2 Cache Attributes (CA)

| Value | Name | Description |
|----|------|------|
| 0 | - | Reserved |
| 1 | Cached | Cached (write-back) |
| 2 | Bypass | Bypass (uncached) |
| 3 | Cached(WriteThrough) | Cached (write-through) |
| 4 | - | Reserved |
| 5-13 | - | Reserved |
| 14 | Isolate | Isolated |
| 15 | - | Reserved |

### 8.3 Memory Attributes

| Attribute | Description |
|------|------|
| Bypass | Uncached, direct memory access |
| Cached | Cached access |
| WriteThrough | Write-through cache |
| WriteBack | Write-back cache |
| Isolate | Isolated access |
| Guarded | Guarded access |

---

## Appendix A: Instruction Opcode Summary

### A.1 Major Opcode (op0)

| op0 | Description |
|-----|------|
| 00xx | QRST - Fast operations |
| 01xx | L32R - Load from literal pool |
| 10xx | LSAI - Load/Store/Immediate |
| 11xx | Reserved/Density instructions |

### A.2 QRST Sub-opcode (op1)

| op1 | Description |
|-----|------|
| 00xx | RST0 - Register/Shift/Test 0 |
| 01xx | RST1 - Register/Shift/Test 1 |
| 10xx | RST2 - Register/Shift/Test 2 |
| 11xx | RST3 - Register/Shift/Test 3 |

### A.3 Architecture Option Abbreviations

| Abbreviation | Option |
|------|------|
| C | Instruction Cache or Data Cache |
| D | MAC16 |
| F | Floating-Point Coprocessor |
| I | 32-Bit Integer Multiply/Divide |
| L | Cache Index Lock |
| M | MMU |
| N | Code Density (Narrow instructions) |
| P | Coprocessor |
| S | Speculation |
| U | Miscellaneous Operations |
| W | Windowed Registers |
| X | Exception or Interrupt |
| Y | Multiprocessor Synchronization |

---

**Document Version:** 1.0
**Last Updated:** 2026-04-03
**Source:** Xtensa Instruction Set Architecture (ISA) Reference Manual
