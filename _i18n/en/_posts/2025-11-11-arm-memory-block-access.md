---
layout: post
lang: en
ref: "arm-memory-block-access"
title: "ARM Assembly #11 - Memory Block Access(LDM, STM)"
date: 2025-11-11 00:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["memory block", "arm ldm", "arm stm"]
published: false
---

In this post, we’ll look at the ARM instructions `LDM` and `STM`, which allow reading or writing **multiple words at once** from memory.
In the previous post, we learned that `LDR` and `STR` can only handle **a single word** at a time.  
However, when working with **contiguous memory areas** such as arrays or structures, repeatedly using `LDR` or `STR` becomes inefficient.  
To solve this problem, ARM provides `LDM` (Load Multiple) and `STM` (Store Multiple), which can operate on memory blocks.

## Memory Block

In ARM programming, it’s very common to handle **a group of contiguous data** such as arrays or structures at once.  
Inside memory, these data are stored consecutively in **word-sized units**.  
This continuous region is called a **memory block**.

<figure style="text-align: center;">
  <img src="/assets/img/memory-block.png" alt="Memory block composed of consecutive words" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  A memory block consists of consecutive words, each mapped to sequential memory addresses.
  </figcaption>
</figure>

Memory is actually a **linear address space**, but for visualization, it’s often represented as a **2D grid** for clarity.  
Each cell represents one word (4 bytes) of storage.

As mentioned in the [previous post]({% post_url 2025-10-31-arm-ldr-str-basic %}), we can access memory word by word using hexadecimal addresses.

### Inefficiency of Using LDR/STR
Instructions like `STR` and `LDR` can only **load or store one word** at a time.  

```armasm
  mov r0, #0x8000
  mov r1, #0x1
  str r1, [r0]
```
<figure style="text-align: center;">
  <img src="/assets/img/ldr-str-single-access.png" alt="Single word access structure" style="display: block; margin: auto;" /> 
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  LDR/STR transfer data in a one-to-one relation between memory and register.
  </figcaption>
</figure>


In other words, the relationship between Mem[address] and a register is **one-to-one per word**.
Although you can manipulate addresses using offsets, you still need to repeat `LDR` and `STR` to handle multiple words.
Such repetition is inefficient when dealing with arrays or structures that occupy contiguous memory.

## LDM and STM
ARM provides `LDM` and `STM` instructions to handle such contiguous memory blocks efficiently.

- `LDM`: Loads multiple words from memory into registers.
- `STM`: Stores the values of multiple registers into memory.

```
  ldm Rn, {registers}
  stm Rn, {registers}
```


Here,  
`Rn` is the base address of the memory block to access.
`{registers}` is a list of registers, separated by commas or defined as a range using a dash (-).

> We call `Rn` the base address, **not** the start address,
> because the actual start address depends on the addressing mode, which we’ll discuss later.

### Working with Register Lists
Let’s look at three examples to see how register lists are used.

> Although only `LDM` is used here, `STM` works the same way.  
> The only difference is that `STM` stores register values into memory instead of loading them.

#### Example 1) Load two words from memory 0x8000 into R1 and R2
```armasm
  mov r0, #0x8000
  ldm r0, {r1, r2}
```
- `R1` ← Mem[`0x8000`] 
- `R2` ← Mem[`0x8004`] 
 
<figure style="text-align: center;">
  <img src="/assets/img/ldm-comma-seperated-registers.png" alt="Comma-separated register list in LDM" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  The instruction <code>ldm r0, {r1, r2}</code> loads two consecutive words into R1 and R2.
  </figcaption>
</figure>

#### Example 2) Load four words from memory 0x8000 into R1 to R4
```armasm
  mov r0, #0x8000
  ldm r0, {r1 - r4}
```
- `R1` ← Mem[`0x8000`] 
- `R2` ← Mem[`0x8004`] 
- `R3` ← Mem[`0x8008`] 
- `R4` ← Mem[`0x800C`] 

<figure style="text-align: center;">
  <img src="/assets/img/ldm-range-registers.png" alt="Range register list in LDM" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  The instruction <code>ldm r0, {r1-r4}</code> reads four consecutive words and stores them sequentially in R1 to R4.
  </figcaption>
</figure>

#### Example 3) Load from memory 0x8000 into R1, R2, and R7
```armasm
  mov r0, #0x8000
  ldm r0, {r1 - r2, r7}
```
- `R1` ← Mem[`0x8000`] 
- `R2` ← Mem[`0x8004`] 
- `R7` ← Mem[`0x8008`] 

<figure style="text-align: center;">
  <img src="/assets/img/ldm-mixed-registers.png" alt="Mixed register list in LDM" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  The instruction <code>ldm r0, {r1-r2, r7}</code> demonstrates that non-consecutive register lists can be specified.
  </figcaption>
</figure>


### Auto-Updating the Base Address
```
  ldm Rn!, {registers}
  stm Rn!, {registers}
```
Adding `!` after the base register `Rn` automatically updates its value to the next address after the operation.

```armasm
  mov r0, #0x8000
  mov r1, #0x1
  mov r2, #0x2
  mov r3, #0x3
 
  stm r0!, {r1 - r3}
  @ r0 = r0 + (3 * 4) = 0x8000 + 0xC = 0x800C
```
In other words, you can store multiple registers and automatically move to the next memory position in one instruction.

---
So far, in all our examples, the memory address has always **increased upward** (from low to high). 
However, depending on the situation, the address may **increase or decrease**,  
and we can also choose whether to access memory **before or after** updating the base register (`Rn`).  

This behavior is controlled by what’s called the **Addressing Mode**.
ARM’s `LDM` and `STM` instructions support the following four addressing modes:

### Four Addressing Modes: IA, IB, DA, DB

| Addressing Mode | Description |
|--|--|
| IA | Icrement After |
| IB | Icrement Before |
| DA | Decrement After |
| DB | Decrement Before |

<figure style="text-align: center;">
  <img src="/assets/img/multi-memory-inst-addr-mode.png" alt="Overview of LDM/STM addressing modes" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  LDM/STM instructions support four addressing modes — IA, IB, DA, and DB — each defining different address increments and starting positions.
  </figcaption>
</figure>


Note:
- Increment / Decrement is determined by the **U-bit**.
  - Increment (U=1): bottom → top (low → high address)
  - Decrement (U=0): top → bottom (high → low address)
- After / Before is determined by the **P-bit**.
  - After (P=0): included Rn
  - Before (P=1): excluded Rn

> As shown in the [stack post]({% post_url 2025-11-06-arm-stack-memory %}),
> it’s easier to visualize memory with higher addresses at the top and lower addresses at the bottom.

#### IA (default):
- start_address = Rn
- end_addrses = Rn + (# of registers * 4) - 4
- Rn = Rn + (# of registers * 4)

```
<bottom & included>
# of registers = 3, Rn = 0x1000
 
0x1008  : end address
0x1004
0x1000  : start address
 
Rn = 0x100C
```

<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png" alt="Memory layout in IA addressing mode" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  In Increment After mode, addresses increase starting from the base register.
  </figcaption>
</figure>
 
#### IB:
- start_address = Rn + 4
- end_address = Rn + (# of registers * 4)
- Rn = Rn + (# of registers * 4)

```
<bottom & excluded>
# of registers = 3, Rn = 0x1000
 
0x100C  : end address
0x1008
0x1004  : start address
0x1000
 
Rn = 0x100C
```

<figure style="text-align: center;">
  <img src="/assets/img/ib-addr-mode-in-memory.png" alt="Memory layout in IB addressing mode" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  In Increment Before mode, the memory range starts just after the base address and increases upward.
  </figcaption>
</figure>
 
#### DA
- start_address = Rn - (# of registers * 4) + 4
- end_address = Rn
- Rn = Rn - (# of registers * 4)

```
<top & included>
# of registers = 3, Rn = 0x1000
 
0x1000  : end address
0x0FFC
0x0FF8  : start address
0x0FF4
0x0FF0
 
Rn = 0xFFF4
```

<figure style="text-align: center;">
  <img src="/assets/img/da-addr-mode-in-memory.png" alt="Memory layout in DA addressing mode" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  In Decrement After mode, addresses decrease starting from the top address.
  </figcaption>
</figure>
 
#### DB:
- start_address = Rn - (# of registers * 4)
- end_address = Rn - 4
- Rn = Rn - (# of registers * 4)

```
<top & excluded>
# of registers = 3, Rn = 0x1000

0x1000
0x0FFC  : end address
0x0FF8
0x0FF4  : start address
 
Rn = 0xFFF4
```

<figure style="text-align: center;">
  <img src="/assets/img/db-addr-mode-in-memory.png" alt="Memory layout in DB addressing mode" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  In Decrement Before mode, the memory range starts before the base address and decreases downward.
  </figcaption>
</figure>
 
## Summary
In this post, we learned how to use `LDM` and `STM` to handle multiple words at once.
In the next post, we’ll see how `LDM` and `STM` are used in stack memory.
Because stacks often need to push or pop multiple registers at once, these instructions are extremely useful.