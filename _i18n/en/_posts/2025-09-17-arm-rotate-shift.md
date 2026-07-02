---
layout: post
lang: en
ref: "arm-rotate-shift"
title: "[ARM32] How to Rotate Bits with ROR and RRX"
date: 2025-09-17 22:40:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["ror", "rrx", "arm", "assembly", "rotate shift"]
---
In this post, we’ll explore the final topic in ARM's shift operations — **Rotate Shift**.  
The ARMv4 architecture provides two rotate instructions: `ror` (Rotate Right) and `rrx` (Rotate Right with eXtend).

Unlike logical or arithmetic shifts, rotate instructions **wrap the shifted-out bits around and insert them back** on the other side.

## Rotate Shift Instructions in ARM

| Instruction | Description |
|:---:|:---:|
| `ror` | Rotate Right |
| `rrx` | Rotate Right with eXtend |

Just like other shift operators, `ror` accepts either an immediate or a register for the shift amount:
- `ror #<imm>`: rotates right by an immediate value
- `ror <Rs>`: rotates right by the value in a register

On the other hand, `rrx` does **not** take any operand and **always rotates by 1 bit**, using the **Carry flag in CPSR** to extend the operation to 33 bits.

## ROR: Rotate Right
The `ror` instruction shifts all bits to the right, and the bits shifted out from the LSB side are **re-inserted into the MSB side**.  
In other words, it's a **bit-preserving, circular move**.

**ror.s**
```text
  .text
  .global _start
_start:
  mov r0, #0x96
  mov r1, r0, ror #4
```

Here, `R0 = 0x96` → `0b0000...1001_0110`
After `ror #4`, the lower 4 bits `0110` are wrapped around and placed at the MSB:

```
Before:   0b0000_0000_0000_0000_0000_0000_1001_0110 = 0x96
After:    0b0110_0000_0000_0000_0000_0000_0000_1001 = 0x60000009
```

## RRX: Rotate Right with Extend
`rrx` behaves similarly to `ror #1`, but with one major difference — it includes the **CPSR's Carry flag** and performs a **33-bit rotation**:

- The LSB is shifted into the **Carry flag**
- The current Carry flag value is shifted into the **MSB**

**rrx.s**
```text
  .text
  .global _start
_start:
  mov r0, #0x69
  movs r1, r0, rrx
  movs r2, r1, rrx
  b .
```

```
Initial:   0b0000_0000_0000_0000_0000_0000_0110_1001 = 0x69

[rrx/C=0]: 0b0000_0000_0000_0000_0000_0000_0011_0100 -> R1=0x34, C=1
[rrx/C=1]: 0b1000_0000_0000_0000_0000_0000_0001_1010 -> R2=0x8000001a, C=0
```

### Checking Carry Flag in GDB
```bash
$ arm-none-eabi-gcc -Ttext=0x10000 -nostdlib rrx.s -o rrx.elf
$ qemu-system-arm -machine versatilepb -nographic -S -s -kernel rrx.elf
$ gdb-multiarch rrx.elf
```

{% highlight bash mark_lines="11 17" %}
(gdb) target remote :1234
(gdb) si                          # Executes: mov r0, #0x69
0x00010004 in _start ()
(gdb) i r r0
r0             0x69     105
(gdb) p ($cpsr >> 29) & 1         # Check initial Carry Flag
$3 = 0
(gdb) si                          # Executes: movs r1, r0, rrx
0x00010008 in _start ()
(gdb) p ($cpsr >> 29) & 1
$4 = 1
(gdb) i r r1
r1             0x34     52
(gdb) si                          # Executes: movs r2, r1, rrx
0x0001000c in _start ()
(gdb) p ($cpsr >> 29) & 1
$5 = 0
(gdb) i r r0 r1 r2
r0             0x69     105
r1             0x34     52
r2             0x8000001a       -2147483622
{% endhighlight %}

## Why is there no Rotate Left (ROL)?
ARMv4T provides ROR (rotate right) but not ROL (rotate left), for two reasons.

First, in a 32-bit rotation, rotating left and rotating right are equivalent.
Rotating left by `n` bits produces exactly the same result as rotating right by `32 - n` bits.
A single ROR can therefore express left rotation as well, leaving no reason for a separate ROL.

Second, ARM does not treat shifts as standalone instructions.
They are folded into the second operand of data-processing instructions via the barrel shifter,
and the field that selects the shift type is only 2 bits wide—enough for just four options (LSL, LSR, ASR, ROR).
The left direction is already taken by LSL, and since rotation is direction-agnostic, ROR alone is chosen to keep the limited encoding space orthogonal.

```text
ror r0, r0, #27   @ equivalent to rotating left by 5 (32 - 5 = 27)
```

## Summary
In this post, we’ve covered ARM’s rotate shift instructions:

- `ror`: circular right shift that preserves all bits
- `rrx`: 1-bit circular shift with carry extension
- `rol` is not an instruction but can be emulated using `lsl` + `ror`

In the next post, we’ll explore how these shift operations integrate with real arithmetic instructions (`add`, `sub`) and how they affect **Carry and Overflow** flags.
