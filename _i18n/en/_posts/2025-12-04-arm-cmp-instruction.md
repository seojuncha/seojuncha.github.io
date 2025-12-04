---
layout: post
lang: en 
ref: "arm-cmp-instruction"
title: "How Does the CMP Instruction Work in ARM Assembly?"
date: 2025-12-04 18:00:00 +0900
categories: ["arm", "assembly"]
tags: ["arm cmp", "arm cpsr"]
---

The `CMP` instruction in ARM is used to compare two values.
**Comparison is performed by subtracting** one value from the other and checking the resulting condition flags.
For example, `5 - 3 = 2` is positive, which means `5` is greater than `3`.
`3 - 5 = -2` is negative, so `3` is less than `5`.
And `5 - 5 = 0` tells us the two values are equal.

The `CMP` instruction uses this arithmetic principle but stores only the status of the result in the `CPSR`, not the actual subtraction result.

[**GitHub Example Code**](https://github.com/seojuncha/arm-assembly-tutorial/tree/main/06_comparing-values)

## CPU Operations and the CPSR
In ARM, the `CPSR` stores the status of all arithmetic and logical operations.
Its **upper four bits** hold the `N`, `Z`, `C`, and `V` flags, which summarize the outcome of an operation.

> The actual computation result is stored in a register, while the `CPSR` records only the status of that result.

For more details on the `CPSR`, refer to [this post]({% post_url 2025-11-27-arm-cpsr-condition-flags %}).

## The CMP Instruction
```armasm
  cmp Rn, Operand2
```

Here, `Rn` must be a register, and `Operand2` is a shifter operand.
As described earlier, `CMP` effectively performs a subtraction (`Rn - Operand2`).

However, it does not store the result back into `Rn` and instead updates only the `CPSR` flags.
In other words, `cmp Rn, Operand2` behaves like `subs Rn, Rn, Operand2`, **except the register value is not modified**.

> `CMP` and other comparison instructions **automatically** update the `CPSR` **without** requiring the `s` suffix.

## Example Code
```armasm
  .text
  .global _start
_start:
    mov r0, #5
    mov r1, #5
    cmp r0, r1
    beq equal

equal:
    mov r2, #0xff
```

After executing `cmp r0, r1`, the `CPSR` flags will be as follows:

| CPSR Flag | Value | Reason      |
| --------- | ----- | ----------- |
| N         | 0     | 5 - 5 ≥ 0   |
| Z         | 1     | 5 - 5 = 0   |
| C         | 1     | No borrow   |
| V         | 0     | No overflow |

> In `CMP`, the `C` flag(Bit[29]) is used for unsigned comparisons.  
> `C=1` when no borrow occurs, and `C=0` when a borrow does occur.

## Debugging
```bash
(gdb) i r cpsr
0x60000010
```
Running `i r cpsr` in GDB shows the updated `CPSR` value after executing `CMP`.
Although GDB prints all 32 bits, we are mainly interested in the upper four bits (`N`, `Z`, `C`, `V`).

You can extract these bits easily using Python:
```python
cpsr = 0x60000010
flags = (cpsr >> 28) & 0xF
print("{:04b}".format(flags))
```

## References
- [ARM Architecture Reference Manual](https://github.com/seojuncha/arm-assembly-tutorial/blob/main/doc/arm-architecture-reference-manual_DDI0100.pdf)
- [Python3 Input and Output](https://docs.python.org/3/tutorial/inputoutput.html)
- [Python3 Format Specification](https://docs.python.org/3/library/string.html#formatspec)