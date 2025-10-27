---
layout: post
lang: en
ref: "arm-arithmetic-operations"
title: "ARM Assembly #6 - Arithmetic Operations"
date: 2025-10-27 20:20:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["add", "sub", "adc", "sbc", "rsb", "rsc"]
---

In this post, we’ll explore the **basic arithmetic operations** in ARM Assembly —  
addition, subtraction, and their variants that include **carry (C flag)**.

So far, we’ve learned how to move values into registers and how to shift them.  
Now it’s time to perform actual **arithmetic operations** with those values.

## Addition and Subtraction in ARMv4
ARM provides the following arithmetic instructions:

| Instructions | Descriptions |
| :-: | :-: |
| `add` | Add |
| `adc` | Add with carry |
| `sub` | Subtract  |
| `sbc` | Subtract with carry |
| `rsb` | Reverse subtract |
| `rsc` | Reverse subtract with carry |

> From now on, we’ll refer specifically to **ARMv4**, as this series is based on that architecture.

All these instructions are **binary operations** that take two operands:

```
result = first-operand op second-operand
<opcode> = <Rd>, <Rn>, <shifter-operand>
```

| Field | Meaning |
| :--: | :--: |
| `<opcode>` | Operations (e.g., add, sub) |
| `<Rd>` | Destination register (result) |
| `<Rn>` | First operand |
| `<shifter-operand>` | Second operand (immediate or shifted register) |


Just like other **data processing instructions**, these also use  
the [shifter operand]({% post_url 2025-09-25-arm-sfhiter-operand %}) mechanism introduced earlier.

## 1. ADD : Addition
```
Assembly Syntax: add <Rd>, <Rn>, <Shifter-Operand>

Methemetical form: C = A + B
  C: <Rd>
  A: <Rn>
  B: <Shifter-Operand>
```

Unlike mathematics, where `C = A + B` and `C = B + A` are equivalent,  
ARM assembly requires the **first operand (`<Rn>`) to be a register**,  
so the order **cannot be swapped**.

**Valid**
```armasm
add r0, r1, #2
```
**Invalid**
```armasm
add r0, #2, r1   @ The first operand must be a register.
```

### Example: ADD
**add.s**
{% highlight armasm mark_lines="5" %}
  .text
  .global _start:
_start:
  mov r0, #3
  add r1, r0, #3
  add r2, r0, lsl #2
  b .
{% endhighlight %}

Explanation:
- `mov r0, #3`:
  - `R0 = 3`
- `add r1, r0, #3`:
  - `R1 = R0 + 3`
  - `R1= 3 + 3`
  - `R1 = 6`
- `add r2, r0, r0 lsl #2`:
  - `R2 = R0 + (3 << 2)`
  - `R2 = 3 + (3 << 2)`
  - `R2 = 3 + 12`
  - `R2 = 15`

Here, the second operand `r0 lsl #2` is a **shifter operand**,
so ARM shifts `r0` left by 2 bits before performing the addition.

## 2. SUB : Subtraction
```
Assembly Syntax: sub <Rd>, <Rn>, <Shifter-Operand>

Methemetical form: C = A - B
  C: <Rd>
  A: <Rn>
  B: <Shifter-Operand>
```
`sub` subtracts the second operand from the first operand.

### Example: SUB
**sub.s**
{% highlight armasm mark_lines="5" %}
  .text
  .global _start:
_start:
  mov r0, #4
  sub r1, r0, #0x1
  b .
{% endhighlight %}

Explanation:
- `mov r0, #4`:
  - `R0 = 4`
- `sub r1, r0, #0x1`:
  - `R1 = R0 - 0x1`
  - `R1 = 4 - 1`
  - `R1 = 3`

## 3. RSB : Reverse Subtract
```
Assembly Syntax: rsb <Rd>, <Rn>, <Shifter-Operand>

Mathmetical form: C = B - A
  C: <Rd>
  A: <Rn>
  B: <Shifter-Operand>
```
`rsb` performs subtraction in the **reverse direction**:
it subtracts the first operand from the second one.
RSB stands for *Reverse Subtract*.

### Example: RSB
**rsb.s**
{% highlight armasm mark_lines="6" %}
  .text
  .global _start:
_start:
  mov r0, #5
  mov r1, #2
  rsb r2, r1, r0
  b .
{% endhighlight %}

Explanation:
- `mov r0, #5`:
  - `R0 = 5`
- `mov r1, #2`:
  - `R1 = 2`
- `rsb r2, r1, r0`:
  - `R2 = R0 - R1`
  - `R2 = 5 - 2`
  - `R2 = 3`

If `sub r2, r1, r0` were used instead, the result would be `-3`.

## 4. ADC/SBC/RBC : Operations with Carry

So far, we’ve handled simple addition and subtraction.
However, computers must also handle carry (C flag) —
the overflow or borrow that occurs when results exceed bit limits.

ARM provides separate instructions for these cases:
| Instruction | Desciption |
| :-: | :-: |
| `adc` | Add with carry (includes carry flag) |
| `sbc` | Subtract with carry (includes borrow) |
| `rsc` | Reverse subtract with carry |

### Example: ADC (Add with Carry)
**adc.s**
```armasm
  .text
  .global _start
_start:
  mov r0, #0x3
  movs r0, r0, lsr #1
  mov r1, #2
  adc r2, r0, r1
  b .
```
 
Explanation:
- `mov r0, #0x3`
  - `R0 = 3`
- `movs r0, r0, lsr #1`
  - `R0 = R0 >> 1`
  - `R0 = 3 >> 1`
  - `R0 = 1`
  - because of the `S` suffix, the **bit shifted out** (1) sets the **Carry flag**.
- `mov r1, #2`
  - `R1 = 2`
- `adc r2, r0, r1`
  - `R2 = R0 + R1 + Carry`
  - `R2 = 1 + 2 + 1`
  - `R2 = 4`
 
#### GDB Debugging Output
```
(gdb) x/5i 0x10000
=> 0x10000 <_start>:    mov     r0, #3
   0x10004 <_start+4>:  lsrs    r0, r0, #1
   0x10008 <_start+8>:  mov     r1, #2
   0x1000c <_start+12>: adc     r2, r0, r1
   0x10010 <_start+16>: b       0x10010 <_start+16>
(gdb) p ($cpsr >> 29) & 1
$1 = 0
(gdb) i r r0 r1 r2
r0             0x0      0
r1             0x0      0
r2             0x0      0
(gdb) si
0x00010004 in _start ()
(gdb) i r r0 r1 r2
r0             0x3      3
r1             0x0      0
r2             0x0      0
(gdb) p ($cpsr >> 29) & 1
$2 = 0       @ Carry flag = 0
(gdb) si
0x00010008 in _start ()
(gdb) p ($cpsr >> 29) & 1
$3 = 1
(gdb) i r r0 r1 r2
r0             0x1      1
r1             0x0      0
r2             0x0      0
(gdb) si
0x0001000c in _start ()
(gdb) i r r0 r1 r2
r0             0x1      1
r1             0x2      2
r2             0x0      0
(gdb) si
0x00010010 in _start ()
(gdb) p ($cpsr >> 29) & 1
$4 = 1        @ Carry flag = 1
(gdb) i r r0 r1 r2
r0             0x1      1
r1             0x2      2
r2             0x4      4
(gdb)
```

As shown, the Carry flag was set during the shift,
and `adc` correctly included it in the addition.

## Summary
Here’s a quick summary of what we covered:

| Type | Instuctions | Description | 
|:-:|:-:|:-:|
| Addition |`add`, `adc` | `adc` adds with the carry flag |
| Subtraction |`sub`, `sbc` | `sbc` subtracts with borrow | 
| Reverse Subtraction | `rsb`, `rsc` | Performs subtraction in reverse order |