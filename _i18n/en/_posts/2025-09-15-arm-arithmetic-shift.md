---
layout: post
lang: en 
ref: "arm-arithmetic-shift"
title: "ARM Assembly #3 - Arithmetic Shift (ASR) and Sign Preservation"
date: 2025-09-15 21:53:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: [Assembly, ASR, ARM, QEMU, GDB]
---
One of the key components of the CPU is the ALU — the Arithmetic Logic Unit. As the name suggests, it handles both logical and arithmetic operations.

In arithmetic operations, it’s important to consider whether a number is signed or unsigned. In the [previous post]({% post_url 2025-09-10-arm-logical-shift %}), we explored logical shifts for bit-level manipulation.  
This time, we’ll look at arithmetic right shifts using the `asr` instruction and how they preserve the sign of a number.

## Arithmetic Shift in ARM

ARM provides a single instruction for arithmetic shift:

| Instruction | Description |
|:---|:---|
| `asr` | Arithmetic Shift Right |

You can use `asr` in two ways:
- `asr #<imm>`: Shift by an immediate value
- `asr <Rs>`: Shift by the value in a register

### How Sign Is Preserved
To preserve the sign of a number during an arithmetic shift,  
**“The shift-in bits are filled with the value of the original sign bit.”**

That means:
- If the most significant bit (MSB) is `0`, the shifted-in bits become `0`
- If the MSB is `1`, the shifted-in bits become `1`

**Case: MSB is 0**
```
Before (32-bit):  
0 1 0 1 1 1 0 0 ...  

ASR #3 →  

After (32-bit):  
0 0 0 0 1 0 1 1 ...
^ ^ ^  
| | |  
+---- Sign-preserving fill: 0
```

**Case: MSB is 1**
```
Before (32-bit):  
1 0 1 0 0 0 1 1 ...

ASR #3 →  

After (32-bit):  
1 1 1 1 0 1 0 ...
^ ^ ^  
| | |  
+---- Sign-preserving fill: 1
```

### ASR Instruction Example

**asr.s**
{% highlight armasm mark_lines="4 5" %}
  .text
  .global _start
_start:
  mov r0, #-32         @ -32 = 0xffffffe0
  mov r1, r0, asr #1   @ R1 = 0xfffffff0 = -16
  b .
{% endhighlight %}

`-32` is represented as `0xffffffe0` in hexadecimal.

After a 1-bit arithmetic shift right:
```
0xffffffe0 >> 1 = 0xfffffff0 (-16)
```
This is equivalent to dividing by 2, while preserving the sign:  
`-32 / 2 = -16`

## Why Only Right Arithmetic Shift?
There is no left arithmetic shift because **left shifting changes the MSB**, and that may be intentional.  
In other words, it doesn't make sense to “preserve the sign” when you're introducing new bits into the sign position.

**left-shift-for-negative.s**
{% highlight armasm mark_lines="4 5" %}
  .text
  .global _start
_start:
  mov r0, #0x40000000
  movs r1, r0, lsl #1
  b .
{% endhighlight %}

Shifting `0x40000000` left by 1 gives `0x80000000`. Here’s the breakdown:

| Binary | Hex | Decimal |
|:-:|:-:|:-:|
| 0b0100... | 0x40000000 | 1073741824 |
| 0b1000... | 0x80000000 | 2147483648 or -2147483648 |

This result can be interpreted as either positive or negative, depending on whether you're treating it as a signed or unsigned integer.

To help the programmer decide, the ARM CPU sets the **N (Negative)** flag in the CPSR register when the result has MSB = 1.  
This lets you check the CPSR and decide how to interpret the result.

> Later, we’ll also explore the **V (Overflow)** flag used in arithmetic operations like `add` and `sub`.
> But for now, since we used `lsl`, only the Negative flag is relevant.
> We’ll discuss flags in more detail in upcoming posts on arithmetic instructions.

<figure style="text-align: center;">
  <img src="/assets/img/non-add-sub-v-left-unchanged.png" alt="ARM manual: overflow flag not updated on logical shift" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">The ARM manual states that overflow flags are generally not updated for non-add/sub operations like logical shifts.</figcaption>
</figure>

In short: left shifts in ARM are always logical (`lsl`).  
There’s no such thing as an “arithmetic left shift” because it doesn’t make sense to preserve the sign when the sign bit itself is being shifted.

## Checking Flags in GDB
Let’s compile and run the `left-shift-for-negative.s` example and check the CPSR flags in GDB.

### 1. Compile
```bash
$ arm-none-eabi-gcc \
    -Ttext=0x10000 \
    -nostdlib \
    left-shift-for-negative.s \
    -o left-shift-for-negative.elf
```
#### 2. Ruin in QEMU
```bash
$ qemu-system-arm \
    -machine versatilepb \
    -nographic \
    -S \
    -s \
    -kernel left-shift-for-negative.elf
```
#### 3. Connect GDB
```
$ gdb-multiarch left-shift-for-negative.elf
```

{% highlight gdb mark_lines="7 9 10" %}
(gdb) target remote :1234
0x00010000 in _start ()    
(gdb) si                   # mov r0, #0x40000000 
0x00010004 in _start ()    
(gdb) p ($cpsr >> 31) & 1
$2 = 0
(gdb) si                   # movs r1, r0, lsl #1
0x00010008 in _start ()
(gdb) p ($cpsr >> 31) & 1
$2 = 1
{% endhighlight %}

The second instruction(`movs r1, r0, lsl #1`) set the N (Negative) flag because the result of the shift was interpreted as a negative number.

## Summary
In this post, we learned:

- How to use the `asr` instruction for arithmetic right shifts
- Why there is no `asl` (arithmetic left shift)
- How to check the Negative flag using GDB

In the next chapter, we’ll explore the final type of shift:
rotate right (`ror`) and rotate with extend (`rrx`), which are used to rotate bits circularly.

Let me know if you found any mistakes or suggestions for improvement!
