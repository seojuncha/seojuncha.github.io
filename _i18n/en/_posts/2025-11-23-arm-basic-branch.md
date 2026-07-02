---
layout: post
lang: en
ref: "arm-basic-branch"
title: "[ARM32] Branching Execution Flow with the B Instruction"
date: 2025-11-23 10:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["arm b", "arm branch instruction"]
---
In this post, we will explore how the `B` instruction is used for branching, along with practical examples.
The `B` instruction unconditionally moves the execution flow to a specified location, forming the foundation for conditionals, loops, and broader program structure.

[Sample Code in GitHub](https://github.com/seojuncha/arm-assembly-tutorial/blob/main/05_branching/branch.s)

## Why Do We Use the B Instruction?
The `B` instruction simply moves the execution flow to another position.
It does not return like a function call; its behavior is similar to the `goto` statement in C.

Typical uses include:
- Implementing loops
- Skipping specific code blocks
- Simple control flow switching

## How Execution Flow Branching Works
The CPU stores the address of the next instruction in the Program Counter (PC).
A branch instruction changes the PC to another address, altering where the CPU fetches the next instruction.

> Note: In ARMv4’s pipeline, the PC appears as current instruction address + 8 bytes.
> However, the assembler handles branch offset calculation, so this does not affect typical coding.

## Branch Instruction: B

```armasm
b label
```

`b label` simply jumps to the address indicated by the label.
Internally, this is equivalent to updating the PC with the label's address.

## Understanding Labels
A label gives a name to a specific instruction address.

```
  label:
    instructions
```

A colon (:) must be placed after the label name.
The label’s address is the address of the first instruction below it.

Example:
```text
  foo:
    mov r0, #1   @ address = 0x10000
    mov r0, #2   @ address = 0x10004
```
Here, the address of foo is `0x10000`.

## Example Code
```text
  .text
  .global _start
_start:
  mov r0, #2
  b   foo
  mov r1, r0     @ never executed
foo:
  add r0, r0, #3
  b   _start
```

**Execution Flow**
1. `mov r0, #2`
2. `b foo` → jumps to foo
3. `add r0, r0, #3`
4. `b _start` → jumps back to _start
5. executes again

Thus, the value of `R0` repeats as 2 → 5 → 2 → 5 indefinitely.
The instruction `mov r1, r0` is never executed due to the branch.

## Debugging: Observing PC Changes with GDB
```bash
(gdb) target remote :1234
(gdb) b _start
(gdb) b foo
(gdb) display/i $pc
(gdb) c
(gdb) si
```
`display/i $pc` is useful for watching how the PC changes step-by-step.
You can clearly see the PC jump directly to the label when executing `b foo`.

## Conclusion
In this post, we explored the basic unconditional branch instruction `B` in ARMv4.
In the next post, we will look at `BL`, the instruction used for function calls.
BL automatically stores the return address in the Link Register (`LR`), making it essential for implementing function calls.