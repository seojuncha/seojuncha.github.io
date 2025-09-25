---
layout: post
lang: en
ref: "arm-shifter-operand"
title: "ARM Assembly #5 - A Complete Guide to Shifter Operands"
date: 2025-09-25 23:10:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["shifter-operand"]
---
In this post, we’ll take a complete look at **Shifter Operands** used in ARM's **Data Processing Instructions**. 
Shifter operands are not optional—they are **fundamental components** of all data processing instructions.

We’ll focus on three key aspects:

1. Types and syntax of shifter operands  
2. How immediate values are encoded and what limitations they have  
3. Why ARM supports shifter operands at all  

## How Shifter Operands Are Used

Instructions like `mov`, `add`, and `sub` always use at least one **shifter operand**.  
A typical data processing instruction includes:

- Destination register (`Rd`)
- Input register (`Rn`)
- Shifter operand (`Shifter Operand`)

<figure>
  <img src="/assets/img/arm-data-processing-table.png" style="display: block; margin: auto;" alt="ARM Data Processing Instruction Table">
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    A summary of ARM's <b>Data Processing Instructions</b>.  
    All of them use a <code>shifter_operand</code> and may combine it with <code>Rn</code> to produce a result in <code>Rd</code>, or simply update condition flags.
  </figcaption>
</figure>

### Instruction Format

- Unary instructions (`mov`, `mvn`):

```
<instruction> <Rd>, <Shifter Operand>
```

- Binary instructions (`add`, `sub`, `and`, etc.):

```
<instruction> <Rd>, <Rn>, <Shifter Operand>
```

In other words:
- `mov` and `mvn` store the shifter operand’s value (or its bitwise negation) into `Rd`
- Other instructions compute `Rn op ShifterOperand`, storing the result in `Rd`

## Types of Shifter Operands

Shifter operands can take one of the following forms:

- **Immediate constant**  
  - `#<immediate>`
- **Register value**  
  - `<Rm>`
- **Shifted register**  
  - Logical shift left: `<Rm>, lsl #<shift_imm>` or `<Rm>, lsl <Rs>`  
  - Logical shift right: `<Rm>, lsr #<shift_imm>` or `<Rm>, lsr <Rs>`  
  - Arithmetic shift right: `<Rm>, asr #<shift_imm>` or `<Rm>, asr <Rs>`  
  - Rotate: `<Rm>, ror #<shift_imm>` / `<Rm>, ror <Rs>` / `<Rm>, rrx`  

Here, `<Rm>` is the operand register used in the operation.

You can plug these into the `<Shifter Operand>` position in any data-processing instruction.

Example:
```
# mov r0, r1
  <instruction> <Rd>, <Shifter Operand>
  <instruction> <Rd>, <Rm>
       mov       r0    r1
 
# add r0, r1, r0, lsl #2
  <instruction> <Rd>, <Rn>, <Shifter Operand>
  <instruction> <Rd>, <Rn>, <Rm> lsl #<shift_imm>
       add       r0    r1    r0         #2
```

### Valid and Invalid Syntax Examples

```armasm
mov r0, r1           @ Valid (<Rd> = r0, <Rm> = r1)
mov r0, #0xff        @ Valid (<Rd> = r0, <Shifter Operand> = 0xff)
mov r0, r0, lsl #2   @ Valid (<Rd> = r0, <Sfhiter Operand> = r0, lsl #2)
add r0, #1           @ Invalid (<Rd> = r0, missing <Rn>)
```

## Immediate Value Limitations
When using an immediate (#<immediate>) as a shifter operand, there’s an important limitation.
You can’t represent all 32-bit values—even though ARM is a 32-bit architecture.

Example:
```armasm
  .text
  .global _start
_start:
  mov r0, #0x101
  b .
```

This causes an error:
```
imm.s: Assembler messages:
imm.s:4: Error: invalid constant (101) after fixup
```

### Immediate Value Encoding
Immediate values in shifter operands are encoded in **12 bits** total:
- 8-bit constant (imm8)
- 4-bit rotation (rotate_imm)

Actual rotation is `rotate_imm * 2` bits (even numbers only)

The final value is computed like this:
```
#<immediate> = #<imm8> ROR (2 * #<rotate_imm>)
```

That means **you can only represent values that can be formed by rotating an 8-bit number.**
> Constraints:
> - 0x00 <= imm8 <= 0xff
> - 0 <= rotate_imm <= 15
> - Effective rotation = 2 * rotate_imm

### Example: Valid Immediate
```armasm
mov r0, #0x104
```
- `0x104` = `0b0000_0000_0000_0000_0000_0001_0000_0100`
- Can’t be stored directly in 8 bits, but:
- If you rotate it **left by 2** (== right by 30), the resulting 8-bit pattern(`0100_0001`) is valid
- Therefore, `imm8=0x41`, `rotate_imm=15` → valid encoding

### Example: Invalid Immediate
```armasm
mov r0, #0x101
```
- `0x101`=`0b0000_0000_0000_0000_0000_0001_0000_0001`
- No rotation of 8-bit values can produce this → invalid

### What to Do with Unrepresentable Constants?
Use a **literal pool**.

**literal-pool.s**
```armasm
  .text
  .global _start
_start:
  ldr r0, =0x101   @ The assembler places 0x101 nearby and loads it
  b .
```
Compile:
```bash
$ arm-none-eabi-gcc \
    -Ttext=0x10000 \
    -nostdlib \
    literal-pool.s \
    -o literal-pool.elf
```
Disassembled:
```bash
$ arm-none-eabi-objdump -D literal-pool.elf
00010000 <_start>:
   10000:       e51f0000        ldr     r0, [pc, #-0]   @ 10008 <_start+0x8>
```
> We’ll explore literal pools in more detail in a future post.

#### Example in C: When Constants Can’t Be Encoded
 
**invalid-imm.c**
```c
int main(void) {
  int x = 0x101;
  return x;
}
```

Compile:
```bash
$ arm-none-eabi-gcc -fomit-frame-pointer -nostdlib -S invalid-imm.c
```

Assembly output:
```armasm
main:
  sub sp, sp, #8
  ldr r3, .L3
  @...
.L3:
  .word 257
```
The constant is stored in a literal pool and loaded with `ldr`.

## Why Use This Encoding?
If ARM allowed 32-bit immediate constants in every instruction, the instruction size would need to increase to 64 bits.
That would violate the compact RISC design of ARM.

Most common constants in real code are:

- Small values (0, 1, 255)
- Bitmasks (0xff00, 0x8000)
- Address offsets

These are often representable via rotate encoding.
The result is **compact, expressive, and efficient**.

## Why Do Shifter Operands Exist?
ARM instructions pass the second operand through a **Barrel Shifter** before going to the ALU.

<figure>
  <img src="/assets/img/arm-barrel-shifter.png" style="display: block; margin: auto;" alt="ARM Barrel Shifter Diagram">
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
A partial block diagram of the ARM7TDMI. The second operand is always passed through a <b>Barrel Shifter</b> before entering the ALU. This allows shift and arithmetic/logic operations to happen in a single instruction.
  </figcaption>
</figure>

This hardware design allows:

- Fewer instructions
- Higher code density
- Better performance

---
This concludes our deep dive into shifter operands in ARM assembly.
Let me know in the comments if you want a breakdown of shift operation opcodes or internal instruction encoding!
