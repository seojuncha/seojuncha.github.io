---
layout: post
lang: en
ref: "stack-memory-and-addressing-modes"
title: "ARM Assembly #12 - Stack Memory and Addressing Mdoes"
date: 2025-11-11 00:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["memory block", "arm ldm", "arm stm", "addressing modes", "stack memory"]
published: false
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

#### IA(Increment After) - default
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

**Example: ldmia.s**
```armasm
  .text
  .global _start
_start:
  ldr r0, =arr
  ldmia r0, {r1-r3}
  b .
 
arr:
  .word 0x1
  .word 0x2
  .word 0x3
```

**Example: stmia.s**
```armasm
  .text
  .global _start
_start:
  mov r0, #0x8000
  mov r1, #0x1
  mov r2, #0x2
  mov r3, #0x3
 
  stmia r0, {r1-r3}
 
  b .
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
 