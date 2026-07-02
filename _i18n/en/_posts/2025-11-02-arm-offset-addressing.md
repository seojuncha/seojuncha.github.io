---
layout: post
lang: en
ref: "arm-offset-addressing"
title: "[ARM32] Calculating Memory Addresses with Offsets"
date: 2025-11-02 13:40:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["ldr", "str", "offset", "addressing"]
---

In this post, we’ll explore how **offset-based addressing** works in ARM assembly.  
You’ll learn how to calculate memory addresses efficiently [using `LDR` and `STR`]({% post_url 2025-10-31-arm-ldr-str-basic %}) without needing separate [`ADD` instructions]({% post_url 2025-10-27-arm-arithmetic-operations %}) each time.

## Why We Need Offsets  
In assembly, a simple instruction like `ldr r0, [r1]` always accesses the same memory address.
But when working with arrays or structures **stored in contiguous memory**, we’d have to use extra `add` instructions each time to move to the next element.

For example:
```
ldr r0, [r1]       @ Read first data
add r1, r1, #4     @ Move to next element
ldr r2, [r1]       @ Read second data
```
That works, but it’s repetitive and inefficient.
Instead, ARM lets us **combine address calculation and memory access**:
```
ldr r0, [r1, #4]   @ Reads directly from r1 + 4
```
This reduces instruction count and CPU cycles, improving performance.


<figure style="text-align: center;">
  <img src="/assets/img/arm-offset-memory.png" alt="Byte-addressed memory with int array and base+offset" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">In byte-addressed memory, an <code>int</code> array advances by 4 bytes per index.</figcaption>
</figure>


## How Offset Address Calculation Works
An offset represents the distance from a base register address.
ARM calculates the final memory address by adding or subtracting this offset from the base:

```
Address = Base Register + Offset
```

Offsets can be:
- Immediate – a constant value like `#4` or `#8`
- Register – the value of another register

Example:
```
ldr r0, [r1, #8]    @ address = r1 + 8
ldr r0, [r1, r2]    @ address = r1 + r2
```
Both addition (+) and subtraction (–) are supported,
and immediate offsets are limited to 12-bit unsigned values (`0`–`4095`).

## Three Offset Forms: Immediate, Register, and Shifted Register
ARM provides three main ways to express an offset:

<figure style="text-align: center;">
  <img src="/assets/img/arm-offset-forms.png" alt="Three offset forms: immediate, register, shifted register" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">Offsets can be expressed as immediate, register, or shifted-register forms.</figcaption>
</figure>

### 1. Immediate Offset – Add a constant directly.

```
[Rn, #imm]
```
Example:

```
ldr r0, [r1, #12]     @ Load data from (r1 + 12)
```

### 2. Register Offset – Add the value of another register.
```
[Rn, Rm]
```
Example:
```
ldr r0, [r1, r2]      @ Load data from (r1 + r2)
```

### 3. Shifted Register Offset – Add a shifted version of another register.
```
[Rn, Rm, Shift #n]
```

Example:
```
ldr r0, [r1, r2, lsl #2]  @ address = r1 + (r2 << 2)
```

Here, `lsl #2` means “×4”.
Since each int element is 4 bytes, this perfectly aligns with array indexing.

## Example Code
Here’s how offset-based address calculation works in assembly:
```
ldr r1, =0x1000            @ Base address (start of the array)
ldr r2, =3                 @ Index i = 3
ldr r0, [r1, r2, lsl #2]   @ address = 0x1000 + (3 << 2) = 0x100C
                           @ r0 = value of arr[3]
```

This is equivalent to accessing `arr[i]` in C.
The `lsl #2` part represents `i * 4`, perfectly matching a 4-byte int element.

<figure style="text-align: center;">
  <img src="/assets/img/arm-offset-forms.png" alt="Scaling i by 4 using lsl #2" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;"><code>lsl #2</code> scales the index <code>i</code> by 4 to match the 4-byte size of <code>int</code></figcaption>
</figure>


## C to Assembly Conversion
In C, when accessing an array element, the compiler automatically calculates the memory address.

For example:
```c
int arr[4] = {10, 20, 30, 40};
int x = arr[2];
```

The compiler might generate:
```
ldr r1, =arr                 @ r1 = &arr[0]
ldr r0, [r1, #8]             @ arr[2] → base + (2 * 4) = +8
```

If the index is stored in a variable:
```
ldr r1, =arr
mov r2, r0                   @ r2 = i
ldr r0, [r1, r2, lsl #2]     @ arr[i] = *(base + i*4)
```

In short:
- Each int element is 4 bytes.
- Memory is byte-addressed, so the address increases by 4 for each index.
- ARM uses `lsl #2` to efficiently represent this `×4` scaling.

## Conclusion
In this post, we learned how to use offsets in `ldr` and `str ` to calculate memory addresses efficiently.

Using offsets lets you perform both address calculation and memory access in one instruction.
This makes it easier and faster to work with arrays or structures stored in consecutive memory.

In the next post, we’ll explore how to use **pre-index** and **post-index addressing**
to perform automatic address updates after each memory access.