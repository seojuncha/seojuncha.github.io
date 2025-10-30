---
layout: post
lang: en
ref: "arm-ldr-str-basic"
title: "ARM Assembly #7 - The Simplest Way to Access Memory with LDR and STR"
date: 2025-10-30 22:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["ldr", "str", "memory-access", "armv4"]
published: false
---
This post is an introductory tutorial on **memory access** for beginners learning ARM assembly.
Using the ARMv4 architecture as a reference, we’ll explore how the CPU uses the `ldr` (load) and `str` (store) instructions to read and write data to memory.

In this post, you’ll learn:
- Why values must be loaded into registers before computation
- How `ldr` and `str` are structured and what they do
- What the square brackets (`[]`) mean in addressing mode
- How to inspect registers and memory states using **QEMU** and **GDB**

> **Key Takeaway:**   
> `ldr` and `str` are the foundation of all ARM programs.
> Through this tutorial, you’ll clearly see how the CPU actually reads and writes data.

## Why We Need to Access Memory

The ARM CPU’s data-processing instructions (`mov`, `add`, etc.) **never access memory directly** — they operate only on values stored in registers.
Therefore, any value stored in memory **must be loaded into a register before performing an operation**.

ARMv4 provides a total of **16** general-purpose registers (`r0`–`r15`).
It’s impossible to store all program data in these registers alone.
Thus, the CPU uses registers as temporary storage for computation, while the main data resides in memory.

## Assembly Instructions for Memory Access

There are two basic instructions for accessing memory:
- `ldr`: loads data **from memory into a register**.
- `str`: stores data **from a register into memory**.

Their basic syntax is:
```
ldr Rd, [Rn]
str Rd, [Rn]
```
Where:
- `Rd`: ***destination register*** to store or retrieve data.
- `Rn`: ***base register*** containing the memory address.

`ldr` reads the value stored at memory address `Rn` into register `Rd`,
and `str` writes the value in `Rd` to the memory location pointed to by `Rn`.

### Meaning of Brackets and Base Register

The value inside square brackets (`[]`) represents a memory address,
and the expression `[address]` refers to the value stored at that address.

For example:
```
ldr r0, [r1]
```

This means *“read the value stored at the **memory address pointed to by** `r1`, and put it into `r0`.”*

## How the CPU Reads and Writes Data via an Address

The CPU interprets the value stored in the base register (`Rn`) as a memory address.
It then locates the memory cell corresponding to that address and either reads or writes the value stored there.

In other words:
- `ldr` = read data from memory using the address.
- `str` = write data to memory using the address.

## Practice Example

The following examples demonstrate the basic behavior of `ldr` and `str`.
You can verify the results using **QEMU (versatilepb)** with `gdb-multiarch`.

> For compiling, running on QEMU, and setting up GDB, refer to [this post]({% post_url 2025-09-08-arm-mov-instruction %}).

```armasm
  .text
  .global _start
_start:
  mov r0, #0x3       @ store decimal 3 into r0
  ldr r1, =0x1000    @ store address 0x1000 into r1
  str r0, [r1]       @ [0x1000] = 3
  b .
```

Here, the CPU interprets the value in `r1` (`0x1000`) as a memory address
and stores the value of `r0` (`3`) into that memory location.

Next, we can load the value back from memory:
```armasm
  .text
  .global _start
_start:
  ldr r1, =0x1000    @ store address 0x1000 into r1
  ldr r0, [r1]       @ load value from [0x1000] into r0
  b .
```

If memory address `0x1000` previously held `3`,
then after execution, `r0` will contain `3`.

---
In this post, we explored **the most fundamental way the CPU interacts with memory using `ldr` and `str`**.
In the next post, we’ll cover more flexible addressing modes using offsets.