---
layout: post
lang: en
ref: arm-mov-instruction
title: ARM Assembly (1) – mov; Storing Values in Registers 
date: 2025-09-08 22:24:00 +0900
categories: ARM Assembly Tutorial
tags: [assembly, embedded, low-level, ARM, QEMU, GDB]
---
Computers are machines designed to perform *calculations*.  
These calculations are ultimately performed by the **CPU**, and a typical operation requires two things:
**operands** and an **operator**.  
Operands are usually stored in **registers**, which are small, fast storage locations inside the CPU.

Assembly language gives us direct access to CPU registers. In this post, we’ll explore how to store values in these registers using the fundamental `mov` instruction.

> You can find the source code used in this tutorial on [GitHub](https://github.com/seojuncha/arm-assembly-tutorial/tree/main/02_moving-data-around)!

## Using mov to Store a Value in a Register

### Storing Immediate Values

**store-imm-to-reg.s**
{% highlight armasm linenos mark_lines="4" %}
  .text
  .global _start
_start:
  mov r0, #2
  b .
{% endhighlight %}

`mov r0, #2`: stores the number 3 in register R0.

The `mov` instruction follows a simple syntax:
```
mov destination_register, value
```
When a number is prefixed with `#` (like `#2`), it is called an ***immediate value***.
You can also use hexadecimal notation: e.g., `#0x2`.

> For compiling, running on QEMU, and setting up GDB, refer to [this post](../../../2025/09/03/hello-arm-assembly).

### Compile 
```bash
$ arm-none-eabi-gcc \
  -nostdlib \
  -Ttext=0x10000 \
  store-imm-to-reg.s \
  -o store-imm-to-reg.elf
```

### Run on QEMU & Debug with GDB 
#### Start QEMU

```bash
$ qemu-system-arm \
  -machine versatilepb \
  -nographic \
  -S -s \
  -kernel store-imm-to-reg.elf
```
Here’s a quick explanation of each option:

| Option | Description |
|:--|:--|
| `-machine versatilepb` | Use the VersatilePB ARM development board |
| `-nographic` | Disable GUI; run in console |
| `-S` | Halt the CPU until GDB connects | 
| `-s` | Start GDB server on port 1234 | 
| `-kernel` | Load the specified ELF file | 

> QEMU may seem unresponsive at first — this is expected. It’s waiting for GDB to connect.

#### Connect with GDB
```bash
$ gdb-multiarch store-imm-to-reg.elf
```

<figure style="text-align: center;">
  <img src="/assets/img/run-gdb.png" alt="Launching GDB with the ELF binary" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">Launching GDB with the compiled ELF binary</figcaption>
</figure>

Inside GDB, connect to QEMU’s GDB server:
```gdb
(gdb) target retmote localhost:1234
```

<figure style="text-align: center;">
  <img src="/assets/img/connect-gdb-server.png" alt="Connecting GDB to QEMU's GDB server on port 1234" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">Connecting GDB to QEMU’s GDB server at port 1234</figcaption>
</figure>

Once connected, we can inspect whether our program is properly loaded into memory.
To check the contents at a specific memory address, GDB uses the following command format: 
```
x/{N}{format} {address}
```
- `{N}`: number of units to display
- `{format}`: output type (e.g., `i` for instructions, `x` for hexadecimal, `s` for string, `u` for unsigned decimal)
- `{address}`: memory address to inspect

For example, to disassemble 10 instructions starting at address `0x10000`, use:
```gdb
(gdb) x/10i 0x10000
```
This helps you verify that the machine code was loaded properly and matches what you wrote in your assembly source file.

<figure style="text-align: center;">
  <img src="/assets/img/show-10-instructions.png" alt="Disassembling 10 instructions starting at address 0x10000" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">Disassembling 10 instructions from address 0x10000</figcaption>
</figure>

To check the current register values:

```gdb
(gdb) info registers   # or shorthand: i r 
(gdb) i r pc r0        # check just PC and R0
```
<figure style="text-align: center;">
  <img src="/assets/img/print-pc-and-r0.png" alt="Printing the values of PC and R0 registers in GDB" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">Checking the values of PC and R0 in GDB</figcaption>
</figure>

At this point, PC (program counter) should be pointing at `0x10000`, which is our `mov r0, #3` instruction.

#### Execute Step-by-Step
To execute a single instruction:
```gdb
(gdb) stepi   # or shorthand: si
```
Then check the register values again:
```gdb
(gdb) i r pc r0
```
<figure style="text-align: center;">
  <img src="/assets/img/next-step-pc-and-r0.png" alt="Stepping through one instruction and inspecting updated PC and R0" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">PC moves forward, and R0 is now updated after executing `mov`</figcaption>
</figure>


You’ll see:
-	PC has moved from `0x10000` to `0x10004`
-	R0 now contains `2`

Run `stepi` one more time to execute `b .`, which creates an infinite loop by jumping to the current PC.

<figure style="text-align: center;">
  <img src="/assets/img/next-step-to-branch.png" alt="PC value remains the same after executing a self-branching instruction" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">PC value stays the same after `b .` (infinite loop)</figcaption>
</figure>

### Copying Between Registers
Let’s now copy a value from one register to another.

{% highlight armasm linenos mark_lines="5" %}
  .text
  .global _start
_start:
  mov r0, #2
  mov r1, r0
  b .
{% endhighlight %}

Here, `mov r1, r0` copies the value of R0 into R1.

> You can repeat the GDB steps from above to verify the register contents after each instruction.

## What’s the Difference Between mov and mvn?
The `mvn` instruction is similar to `mov`, but it stores the bitwise NOT ([1’s complement](https://en.wikipedia.org/wiki/Ones%27_complement)) of the value.

|mvn|mov|
|:--|:--|
|mvn r0, #0|mov r0, #-1|
|mvn r0, #-1|mov r0, #0|
|mvn r0, #0xF|mov r0, #-16|

> Pro tip: Use Python REPL to test bitwise negation: 
> ```python
> >>> ~0xf
> -16
> ```


## Coming Up Next: Shift Operands
Both `mov` and `mvn` belong to the **Data Processing Instructions** category in the ARM architecture.

These instructions can use a **shifted operand** as input, which allows you to shift bits while copying or computing values.

This concept applies to many arithmetic operations (like `add`, `sub`, etc.), and we’ll cover it in the next post.

<figure style="text-align: center;">
  <img src="/assets/img/shifter-operand-in-arm-manual.png" alt="Excerpt from ARM manual showing shifted operand encoding" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">ARM manual: encoding format for shifted operand</figcaption>
</figure>

## Summary 
In this post, we explored how to:
-	Use `mov` to assign immediate values to registers
-	Copy values between registers
-	Understand the difference between `mov` and `mvn`
-	Use QEMU and GDB to debug and inspect registers

We confirmed our code works in a simulated ARM environment using QEMU and GDB — a key part of your low-level dev workflow.

Don’t worry if GDB feels unfamiliar right now. We’ll continue using this setup in future posts, so it will become second nature with practice.

Up next: we’ll dive into shift operands and how they enhance the expressiveness of ARM instructions.
