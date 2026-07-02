---
layout: post
lang: en
ref: "arm-cpsr-condition-flags"
title: "[ARM32] Conditional Execution Using CPSR and Condition Flags"
date: 2025-11-27 20:20:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["condition flags", "arm cpsr"]
---
Until now, every instruction we examined executed unconditionally once its turn came.
However, the ARM architecture provides a unique feature: it can **execute an instruction only when a specific condition is met**, using the **condition flags** stored in the `CPSR` register.

This allows ARM to implement conditional logic without using branch instructions, which is one of the architecture’s defining characteristics.
Reducing branches also helps maintain pipeline efficiency.

## CPSR (Current Program Status Register)
The `CPSR` register stores the CPU’s program state and contains four condition flags in its upper 4 bits:

| Bit | Flag | Meaning         |
| --- | ---- | --------------- |
| 31  | N    | Negative result |
| 30  | Z    | Zero result     |
| 29  | C    | Carry           |
| 28  | V    | Overflow        |

To update these flags, the instruction must include the `s` suffix, such as `adds`, `subs`, or `movs`.

## Condition Flags
Almost every ARM instruction can be made conditional.
By appending a two-letter condition code (e.g., `EQ`, `NE`, `MI`), the instruction will execute only when the corresponding `CPSR` flags match the condition.

<figure style="text-align: center;"> 
  <img src="/assets/img/condition-code-table-in-arm-document.png" 
    alt="ARM condition code table" 
    style="display: block; margin: auto;" /> 
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;"> 
  Figure 1. Condition code summary from the ARM Architecture Reference Manual
  </figcaption> 
</figure>

## Instruction + Condition
```
{instruction}{condition}
```

To execute an instruction only **when the previous result is negative**:
```armsm
  movmi r0, r1
```
`MI`(Minus) executes only when the `N` flag is `1`, meaning the previous result was negative.

## Example Code
```text
  .text
  .global _start
_start:
  movs r0, #-1
  addmi r0, r0, #5
  addpl r0, r0, #1
  b .
```

This code performs:
- Add `5` when the value is negative
- Add `1` when the value is zero or positive

Equivalent C code:
```c
int r0 = -1;
r0 += (r0 < 0) ? 5 : 1;
```

## Debugging CPSR Flags
Use GDB to inspect `CPSR`:

```bash
(gdb) p/x $cpsr
```

Extract each flag:
```bash
p ($cpsr >> 31) & 1   # N
p ($cpsr >> 30) & 1   # Z
p ($cpsr >> 29) & 1   # C
p ($cpsr >> 28) & 1   # V
```

## Conclusion
In this post, we explored conditional execution using `CPSR` flags and condition codes.
In the next article, we will cover conditional branches using `CMP`/`CMN` and discuss how to choose between conditional execution and branching.