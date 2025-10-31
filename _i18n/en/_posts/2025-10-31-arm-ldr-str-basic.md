---
layout: post
lang: en
ref: "arm-ldr-str-basic"
title: "ARM Assembly #7 - The Simplest Way to Access Memory with LDR and STR"
date: 2025-10-31 20:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["ldr", "str", "memory-access", "armv4"]
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


<figure>
<svg role="img" aria-labelledby="title desc" viewBox="0 0 800 240" width="100%" xmlns="http://www.w3.org/2000/svg">

  <title id="title">CPU ↔ Memory data flow with LDR/STR</title>
  <desc id="desc">Shows LDR reading from Memory to Register and STR writing from Register to Memory.</desc>
  <style>
    .fg { stroke:#E6E6E6; fill:#E6E6E6; }
    .box { fill:#0B1220; stroke:#A3B1C6; }
    .wire { stroke:#E6E6E6; }
    .muted { fill:#C8D0DB; }
    .marker path { fill:#E6E6E6; }  
    .label{font:16px sans-serif}
    .note{font:15px sans-serif}
    .mono{font:14px monospace}
  </style>
  <defs>
    <marker class="marker" id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" />
    </marker>
  </defs>

  <!-- CPU Registers -->
  <rect class="box" x="60" y="50" width="280" height="140" rx="8" />
  <text class="label fg" x="200" y="80" text-anchor="middle">CPU Registers</text>
  <!-- little register slots -->
  <g transform="translate(90,100)">
    <rect class="box" x="0" y="0" width="50" height="24" rx="4" />
    <rect class="box" x="60" y="0" width="50" height="24" rx="4" />
    <rect class="box" x="120" y="0" width="50" height="24" rx="4" />
    <rect class="box" x="180" y="0" width="50" height="24" rx="4" />
    <text class="note fg" x="25" y="17" text-anchor="middle">r0</text>
    <text class="note fg" x="85" y="17" text-anchor="middle">r1</text>
    <text class="note fg" x="145" y="17" text-anchor="middle">r2</text>
    <text class="note fg" x="205" y="17" text-anchor="middle">…</text>
  </g>

  <!-- Memory -->
  <rect class="box" x="460" y="30" width="280" height="180" rx="8" />
  <text class="label fg" x="600" y="60" text-anchor="middle">Memory</text>
  <g transform="translate(490,80)">
    <rect class="box" x="0" y="0" width="220" height="28" />
    <rect class="box" x="0" y="36" width="220" height="28" />
    <rect class="box" x="0" y="72" width="220" height="28" />
    <text class="note fg" x="8" y="19">0x1000</text>
    <text class="note fg" x="8" y="55">0x1004</text>
    <text class="note fg" x="8" y="91">0x1008</text>
  </g>

  <!-- Arrows -->
  <line class="wire" x1="340" y1="120" x2="460" y2="120" stroke="#000" stroke-width="2" marker-end="url(#arrow)"/>
  <text class="label fg" x="400" y="110" text-anchor="middle">STR</text>
  <line class="wire" x1="460" y1="160" x2="340" y2="160" stroke="#000" stroke-width="2" marker-end="url(#arrow)"/>
  <text class="label fg" x="400" y="150" text-anchor="middle">LDR</text>
</svg>
<figcaption>
LDR moves data Memory → Register; STR moves data Register → Memory.
</figcaption>
</figure>

## Why We Need to Access Memory

The ARM CPU’s data-processing instructions (`mov`, `add`, etc.) **never access memory directly** — they operate only on values stored in registers.
Therefore, any value stored in memory **must be loaded into a register before performing an operation**.


ARMv4 provides a total of **16** general-purpose registers (`r0`–`r15`).
It’s impossible to store all program data in these registers alone.
Thus, the CPU uses registers as temporary storage for computation, while the main data resides in memory.

<figure>
<svg role="img" aria-labelledby="title desc" viewBox="0 0 800 260" width="100%" xmlns="http://www.w3.org/2000/svg">

  <title id="title">Registers vs Memory roles</title>
  <desc id="desc">Shows 16 general-purpose registers contrasted with large memory as main storage.</desc>
  <style>
    .fg { stroke:#E6E6E6; fill:#E6E6E6; }
    .box { fill:#0B1220; stroke:#A3B1C6; }
    .wire { stroke:#E6E6E6; }
    .muted { fill:#C8D0DB; }
    .marker path { fill:#E6E6E6; }  
    .label{font:16px sans-serif}
    .note{font:13px sans-serif}
    .mono{font:14px monospace}
  </style>

  <!-- Registers grid -->
  <rect class="box" x="60" y="30" width="300" height="200" rx="8" />
  <text class="label fg" x="210" y="55" text-anchor="middle">Registers (16)</text>
  <g transform="translate(80,75)">
    <!-- 4 x 4 grid -->
    <g class="row" transform="translate(0,0)">
      <rect class="box" x="0" y="0" width="60" height="30" rx="4"/><text class="note fg" x="30" y="20" text-anchor="middle">r0</text>
      <rect class="box" x="70" y="0" width="60" height="30" rx="4"/><text class="note fg" x="100" y="20" text-anchor="middle">r1</text>
      <rect class="box" x="140" y="0" width="60" height="30" rx="4"/><text class="note fg" x="170" y="20" text-anchor="middle">r2</text>
      <rect class="box" x="210" y="0" width="60" height="30" rx="4"/><text class="note fg" x="240" y="20" text-anchor="middle">r3</text>
    </g>
    <g class="row" transform="translate(0,40)">
      <rect class="box" x="0" y="0" width="60" height="30" rx="4"/><text class="note fg" x="30" y="20" text-anchor="middle">r4</text>
      <rect class="box" x="70" y="0" width="60" height="30" rx="4"/><text class="note fg" x="100" y="20" text-anchor="middle">r5</text>
      <rect class="box" x="140" y="0" width="60" height="30" rx="4"/><text class="note fg" x="170" y="20" text-anchor="middle">r6</text>
      <rect class="box" x="210" y="0" width="60" height="30" rx="4"/><text class="note fg" x="240" y="20" text-anchor="middle">r7</text>
    </g>
    <g class="row" transform="translate(0,80)">
      <rect class="box" x="0" y="0" width="60" height="30" rx="4"/><text class="note fg" x="30" y="20" text-anchor="middle">r8</text>
      <rect class="box" x="70" y="0" width="60" height="30" rx="4"/><text class="note fg" x="100" y="20" text-anchor="middle">r9</text>
      <rect class="box" x="140" y="0" width="60" height="30" rx="4"/><text class="note fg" x="170" y="20" text-anchor="middle">r10</text>
      <rect class="box" x="210" y="0" width="60" height="30" rx="4"/><text class="note fg" x="240" y="20" text-anchor="middle">r11</text>
    </g>
    <g class="row" transform="translate(0,120)">
      <rect class="box" x="0" y="0" width="60" height="30" rx="4"/><text class="note fg" x="30" y="20" text-anchor="middle">r12</text>
      <rect class="box" x="70" y="0" width="60" height="30" rx="4"/><text class="note fg" x="100" y="20" text-anchor="middle">r13</text>
      <rect class="box" x="140" y="0" width="60" height="30" rx="4"/><text class="note fg" x="170" y="20" text-anchor="middle">r14</text>
      <rect class="box" x="210" y="0" width="60" height="30" rx="4"/><text class="note fg" x="240" y="20" text-anchor="middle">r15</text>
    </g>
  </g>

  <!-- Memory -->
  <rect class="box" x="440" y="30" width="300" height="200" rx="8" />
  <text class="label fg" x="590" y="55" text-anchor="middle">Main Memory (Data store)</text>
  <text class="note fg" x="590" y="90" text-anchor="middle">Large, holds program data</text>
  <text class="note fg" x="590" y="110" text-anchor="middle">Registers used for compute</text>
</svg>
<figcaption>
Registers are temporary for compute; memory is the main data store.
</figcaption>
</figure>


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

<figure>
<svg role="img" aria-labelledby="title desc" viewBox="0 0 800 220" width="100%" xmlns="http://www.w3.org/2000/svg">

  <title id="title">Base register addressing</title>

  <desc id="desc">Shows r1 holding an address 0x1000 and [r1] meaning value at memory[0x1000].</desc>

  <style>
    .fg { stroke:#E6E6E6; fill:#E6E6E6; }
    .box { fill:#0B1220; stroke:#A3B1C6; }
    .wire { stroke:#E6E6E6; }
    .muted { fill:#C8D0DB; }
    .marker path { fill:#E6E6E6; }  
    .label{font:16px sans-serif}
    .note{font:15px sans-serif}
    .mono{font:14px monospace}
  </style>

  <defs>
    <marker class="marker" id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" />
    </marker>
  </defs>

  <!-- r1 register -->
  <rect class="box" x="60" y="60" width="240" height="100" rx="8"/>
  <text class="label fg" x="180" y="88" text-anchor="middle">r1</text>
  <text class="label fg" x="180" y="120" text-anchor="middle">0x00001000</text>

  <!-- memory cell -->
  <rect class="box" x="480" y="40" width="240" height="140" rx="8"/>
  <text class="label fg" x="600" y="70" text-anchor="middle">Memory[0x1000]</text>
  <rect class="box" x="510" y="90" width="180" height="40"/>
  <text class="mono fg" x="600" y="115" text-anchor="middle">0x000004D2</text>

  <!-- arrow and labels -->
  <line x1="300" y1="110" x2="480" y2="110" stroke="#6f6c6cff" stroke-width="2" marker-end="url(#arrow)"/>
  <text class="note fg" x="390" y="100" text-anchor="middle">[r1] ≡ *(uint32_t*)0x1000</text>
</svg>
<figcaption>
If <code>r1</code> holds 0x1000, then <code>[r1]</code> means the value stored at that memory address.
</figcaption>
</figure>


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