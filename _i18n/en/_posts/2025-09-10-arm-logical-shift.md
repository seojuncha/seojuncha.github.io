---
layout: post
lang: en
ref: arm-logical-shift
title: "[ARM32] Understanding Logical Shifts (LSL, LSR)"
date: 2025-09-10 00:13:00 +0900
categories: [arm,assembly,tutorial]
tags: [Assembly, Embedded, low-level programming, ARM, QEMU, GDB]
---

Computers frequently perform bit-level operations to manipulate data, and **shift operations** are one of the most fundamental among them. In this post, we’ll focus on **Logical Shifts** in ARM assembly.

Rather than going deep into abstract theory, we’ll walk through simple examples that show how logical shift works in practice. We’ll also look at how the **Carry Flag** (C flag) is affected by these operations.

> You can find the example code on the [GitHub repository](https://github.com/seojuncha/arm-assembly-tutorial/tree/main/02_moving-data-around).


## What is a shift operand?

In ARM assembly, instructions like `mov`, `add`, and `sub` allow one of the operands to be shifted on the fly before being used.  

This is called a **shift operand**.

Thanks to this feature, you can perform shift and assignment in a single instruction, making the code more concise.


<figure style="text-align: center;">
  <img src="/assets/img/alut-shifter-usage.png" alt="Explanation of how ARM instructions control both the ALU and the shifter to maximize efficiency" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">ARM data-processing instructions allow simultaneous use of the Arithmetic Logic Unit (ALU) and the shifter, enabling efficient execution of complex operations in a single instruction cycle.</figcaption>
</figure>

Example:
```armasm
  mov r1, r0, lsl #2   @ Shift R0 left by 2 bits and store in R1
```

## What is a logical shift?
A **logical shift** simply moves the bits to the left or right, filling empty bits with 0.

For example, using 4-bit data `0110`:
```
Left shift:     0110 << 1 = 1100
Right shift:    0110 >> 1 = 0011
```

ARM provides two logical shift operations:

| Instruction | Description |
|:--:|:--:|
|`lsl`| Logical Shift Left |
|`lsr`| Logical Shift Right |

The amount of shift can be specified either by an immediate value or by a register.

- `lsl #n`, `lsr #n` : Immediate shift
- `lsl rN`, `lsr rN` : Register shift

### Example Code: Logical Shift in Action

**logical-shift.s**
```text
  .text
  .global _start
_start:
  mov r0, #3
  mov r1, r0, lsl r0
  mov r2, r1, lsr #1
  b .
```

- `mov r0, #3` stores 3 in `R0`
- `mov r1, r0, lsl r0` shifts `R0` left by 3 bits and stores in `R1`
- `mov r2, r1, lsr #1` shifts `R1` right by 1 bit and stores in `R2`

> For an introduction to the `mov` instruction, please refer to [the previous post]({% post_url 2025-09-08-arm-mov-instruction %})

Example flow:
```
r0 = 0x3 = 0b0...000011

# r1 = r0 << 3
r1 = 0x18 = 0b0..011000

# r2 = 0x18 >> 1
r2= 0xC = 0b0...001100
```

### Debugging with GDB
Let’s confirm the behavior using QEMU and GDB.

#### 1. Run QEMU 
```bash
$ qemu-system-arm \
    -machine versatilepb \
    -nographic \
    -S \
    -s \
    -kernel logical-shift.elf
```
#### 2. Connect with GDB
```bash
$ gdb-multiarch logical-shift.elf
(gdb) target remote localhost:1234
```
#### 3. Check instructions
```bash
(gdb) x/10i 0x10000
```
#### 4. Check registers 
```bash
(gdb) info registers
(gdb) i r r0 r1 r2
```
#### 5. Step Through
```bash
(gdb) stepi
```
After each step, check register values again to confirm the effect.

## When is the Carry Flag set? 
Shift operations push out bits that no longer fit in the range.
One of those bits may be captured in the **Carry Flag (C)** in the **CPSR** register.

<figure style="text-align: center;">
  <img src="/assets/img/cpsr-condition-flags.png" alt="Description of the CPSR register and the four condition flags: Negative, Zero, Carry, and Overflow." style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">The Current Program Status Register (CPSR) holds the processor's condition flags—Negative (N), Zero (Z), Carry (C), and Overflow (V)—which reflect the results of arithmetic and logical operations.</figcaption>
</figure>

By default, instructions like `mov` do not update the carry flag.
To update it, use the `s` suffix: `movs`, `adds`, `subs`, etc.

### Example Code: Checking the Carry Flag 
**logical-right-shift-carry.s**
```
  .text
  .global _start
_start:
  mov r0, 0x1
  mov r1, r0, lsr #1
  movs r2, r0, lsr #1
  b .
```

- `mov` does **not** affect the Carry flag
- `movs` updates the CPSR flags including Carry

To check the carry flag in GDB:

```bash
(gdb) target remote localhost:1234
(gdb) info registers cpsr
(gdb) p ($cpsr >> 29) & 1
```
This will show whether the Carry flag (bit 29) is set.

<figure style="text-align: center;">
  <img src="/assets/img/cpsr-layout.png" alt="CPSR bit layout showing N, Z, C, V flags and mode bits." style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">CPSR register layout with condition flags and mode bits.</figcaption>
</figure>

## Limitations and Use Cases
Logical shift always fills in `0`s. This works well for **unsigned integers**,
but can cause issues with **signed values** where the highest bit represents the sign.

Example:
```
  mov r0, #-4
  mov r1, r0, lsr #1
```

Shifting a signed number right with `lsr` can destroy the sign bit.
In these cases, use **ASR (Arithmetic Shift Right)** instead.

Despite this limitation, logical shift is extremely useful in:
- Array indexing
- Address calculations
- Bitmasking

## Conclusion
In this post, we covered:
- How to use `lsl` and `lsr` in ARM assembly
- What a shift operand is
- How to check the Carry flag using `movs`
- Why logical shifts are suitable for unsigned integers

In the next post, we'll explore other shift types such as `asr` and `ror`.
