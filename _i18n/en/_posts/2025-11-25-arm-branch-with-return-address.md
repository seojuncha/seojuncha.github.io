---
layout: post
lang: en
ref: "arm-branch-with-return-address"
title: "ARM Assembly #14 - Using the BL Instruction That Automatically Saves the Return Address" 
date: 2025-11-23 10:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["arm bl", "arm branch instruction", "link register", "arm lr"]
published: false
---

The `BL` instruction is an extended version of the `B` instruction described in the [previous post]({% post_url 2025-11-23-arm-basic-branch %}), adding function-call capability.
While the `B` instruction simply jumps to a target label, the `BL` instruction **automatically stores the return address in the `LR`** (Register `R14`) so execution can return after the branch.
For this reason, `BL` is a core mechanism for implementing function calls in ARM and forms the foundation for concepts like the stack, calling conventions, and frame pointers.

<figure style="text-align: center;">
  <a href="https://youtu.be/BZamp0G0viA" target="_blank">
  <img src="/assets/img/"
    alt="Youtube Video to explain ARM BL instruction"
    style="display: block; margin: auto;" /> 
  </a>
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  <a href="https://www.youtube.com/@seojuncha" target="_blank">youtube.com/@seojuncha</a>
  </figcaption>
</figure>

[**GitHub Sample Code**](https://github.com/seojuncha/arm-assembly-tutorial/blob/main/05_branching/branch-with-link.s)

## Why We Need to Store the Return Address
When a function finishes executing, it must return to the point where it was called.
If the return address is not stored, the CPU won't know which instruction to execute next, so the address must be saved somewhere.

## The BL Instruction
```armasm
  bl label
```
The syntax looks identical to `b label`, but the internal behavior is different.
When executing `bl label`, the CPU jumps to the label and simultaneously stores the address of the next instruction (the return address) into `LR`.
After the subroutine finishes, the value stored in `LR` is loaded back into `PC` to restore the original control flow.

<figure style="text-align: center;">
  <img src="/assets/img/bl-with-lr-and-pc.png"
    alt="Diagram of PC and LR behavior during BL instruction execution"
    style="display: block; margin: auto;" /> 
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  Figure 1. When the BL instruction executes, the PC jumps to the subroutine while the LR stores the return address for later execution.
  </figcaption>
</figure>

## The LR (Link Register)
ARM uses the `LR` as a dedicated register to temporarily store the return address when making function calls.
In ARMv4, register `R14` is used as the `LR`.
```armasm
  bl foo
  mov r0, #2   @ LR has this instruction address.
```

When the subroutine finishes, execution must return using the value stored in `LR` by writing it into `PC`.
```armasm
  mov pc, lr
```

> Since ARMv4 does not support the `bx` instruction, returning is done using `mov pc, lr`.
> Starting from ARMv5, `bx lr` became the standard return instruction.

## Example Code
```armasm
  .text
  .global _start
_start:
  mov r0, #2
  bl foo
  mov r1, r0
  b .
foo:
  add r0, r0, #3
  mov pc, lr
```

### Execution Flow
1. `mov r0, #2` → R0 = 2
2. `bl foo` → PC = foo, LR = address of `mov r1, r0`
3. `add r0, r0, #3` → R0 = 5
4. `mov pc, lr` → PC jumps to `mov r1, r0`
5. run `mov r1, r0`

This example shows a simple flow where the subroutine foo adds `3` to `R0` and returns the result to be stored in `R1`.
Without `mov pc, lr`, the result would never be written to `R1` and the program flow would not continue properly.

<figure style="text-align: center;">
  <img src="/assets/img/control-flow-with-pc-and-lr.png"
    alt="Control-flow diagram showing return mechanism using BL and mov pc, lr"
    style="display: block; margin: auto;" /> 
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  Figure 2. This diagram illustrates how BL stores the return address in LR and how <code>mov pc, lr</code> restores execution flow after the subroutine completes.
  </figcaption>
</figure>

### Debugging
```bash
(gdb) target remote :1234
(gdb) display/i $pc
(gdb) display/i $lr
```

By inspecting `PC` and `LR` right after `bl foo`, you can clearly see that BL stores the next instruction’s address in `LR`.
In particular, if `LR` matches the address of `mov r1, r0`, the behavior of `BL` becomes very intuitive.

## Conclusion
In the next post, we’ll examine condition codes for conditional execution.
Until now, every instruction executed unconditionally, but ARM allows adding two-letter condition codes so instructions run only under specific conditions.