+++
date = '2025-01-22'
title = 'Carry & Overflow in ARM: Debugging with Assembly & QEMU'
tags = ["carry", "overflow", "arm", "arm assembly", "cpsr", "qemu", "gdb"]
draft = true
+++

Understanding the operation of the hardware ***Carry flag*** (C) and ***Overflow flag*** (V) in depth from a programmer's perspective offers significant advantages in various aspects. In particular, they play a crucial role in **low-level system programming**, **embedded systems development**, and **performance optimization and debugging**. For example,

1. Accurate numerical computation
2. High-performance low-level optimization
3. Debugging and troubleshooting
4. Error detection and improved stability
5. Implementation of multi-precision arithmetic
6. Conditional branching and code simplification

These advantages make mastering these concepts highly beneficial. In this post, we will explore how **[Carry and Overflow](/posts/carry-and-overflow/) are handled in ARM processors**, along with **a simple assembly example** that we will **debug using QEMU**.

Through this process, we aim to gain a deeper understanding of how arithmetic operations in our code function at the hardware level, empowering us to write better code and solve problems more effectively.

## 32-bit ARM Processor
This post is based on the 32-bit ARM processor used in the ongoing [From the Transistor](https://github.com/seojuncha/fromthetransistor-fork) project.

A 32-bit processor signifies that both the data size it can process at once and the memory address size are 32 bits.

Key characteristics of a 32-bit processor include:

- Register size: 32 bits
- Data bus and memory bus size: 32 bits
- [Memory alignment](https://en.wikipedia.org/wiki/Data_structure_alignment) unit: 32 bits
- Word size: 32 bits

## CPSR: Current Program Status Register
ARM processors have a special register called the ***CPSR \(Current Program Status Register\)***, which is used to **store the current state of the program**.

**CPSR Format**
```
32-bit CPSR
+---+---+---+---+---+----------+---+---+---+----+----+----+----+----+
| N | Z | C | V | Q | DNM(RAZ) | I | F | T | M4 | M3 | M2 | M1 | M0 |
+---+---+---+---+---+----------+---+---+---+----+----+----+----+----+
```

### Condition Code Flags
The upper 4 bits [31:28] of the CPSR are designated as the ***Condition Code Flags*** and are used to evaluate the program's status. These bits include the following fields:

- N (***N***egative)
- Z (***Z***ero)
- C (***C***arry)
- V (o***V***erflow)

The Condition Code Flags are set to 1 when a specific condition is met and cleared to 0 when it is not. Each flag occupies a single bit in the CPSR.

> The term *flag* typically refers to controlling two states: true (1) or false (0).  
> Setting a flag to true is referred to as set, while changing it to false is called clear.

#### N Flag
- Set: When the result is negative
- Clear: When the result is zero or positive

#### Z Flag
- Set: When the result is zero
- Clear: When the result is not zero

#### C Flag
- Set: When a carry occurs
- Clear: When no carry occurs

#### V Flag
- Set: When an overflow occurs
- Clear: When no overflow occurs

> **What conditions are being checked?**  
> The CPU (e.g., ARM) is essentially a machine that executes *binary-encoded instructions*. The four flags set in the CPSR are all used to **examine the result of executing an instruction**.  
> 
> Example: If the result of an instruction's operation is 0, Z Flag is Set

## Carry Flag & Overflow Flag
The behavior of the Carry Flag and Overflow Flag varies slightly depending on the type of operation and CPU instruction. 

ARM instructions can be broadly classified into the following three types:
- Data-Processing Instructions
  - Perform operations such as arithmetic, logical, and shift operations.
  - Examples: `ADD`, `MOV`, `AND`, etc.
- Load and Store Instructions
  - Transfer data between memory and registers.
  - Examples: `LDR`, `STR`, etc.
- Branch Instructions
  - Control the program's flow of execution.
  - Examples: `B`, `BL`, etc.

> Additional instruction types exist, but these three are sufficient to understand for now.  
> A deeper dive into ARM assembly will be covered in future posts.

### ARM Assembly with suffix 's'
However, **not all data-processing instructions update the status flags**. This design choice provides programmers with greater efficiency and flexibility. Updating the status flags involves hardware operations, and applying this to every instruction would lead to unnecessary power consumption and performance degradation.

```armasm
ADD r0, r1, r2   @ Not update the status flags
ADDS r0, r1, r2  @ Update the status flags
```

## Assembly Example: Compare "ADD" and "ADDS"
Let’s explore the concepts discussed so far with an actual ARM assembly code example.

> The reason for not using a C language example is that the generation of assembly code is compiler-dependent.  
> Converting C code into assembly is entirely the compiler’s responsibility.  
> Depending on the logic structure and optimization, the compiler may or may not use the S suffix.
> 
> Therefore, to simplify the example, we will use assembly directly.

### Environment
- Ubuntu 22.04
- [arm-none-eabi-gcc](https://launchpad.net/ubuntu/jammy/+package/gcc-arm-none-eabi) 10.3.1
- [gdb-multiarch](https://installati.one/install-gdb-multiarch-ubuntu-22-04/) 12.1

### Code Examples
Here are two assembly codes, `add.s` and `adds.s`, along with the following test scenario: 

**add.s**
```armasm{hl_lines=[6]}
.section .text
.global _start

_start:
  mov r0, #0xffffffff
  add r1, r0, #2
```

**adds.s**
```armasm{hl_lines=[6]}
.section .text
.global _start

_start:
  mov r0, #0xffffffff
  adds r1, r0, #2
```
**Compile**:
```shell
$ arm-none-eabi-gcc -g -o add.elf add.s -nostdlib -specs=nosys.specs
$ arm-none-eabi-gcc -g -o adds.elf adds.s -nostdlib -specs=nosys.specs
```

#### Test Scenario
1. Assign the 32-bit maximum value (0xFFFF_FFFF) to the r0 register.
2. Add 2 to the value in the r0 register and store the result in the r1 register.
   - Check the changes in the CPSR when using add.
   - Check the changes in the CPSR when using adds.

By comparing these two cases, we can observe the impact of the S suffix on the `CPSR` flags.

### Run on QEMU
We will use the QEMU emulator to execute each file on a 32-bit ARM processor. This setup allows us to analyze how the instructions (`add` and `adds`) behave and how they affect the `CPSR` flags during execution.

```shell
$ qemu-system-arm -M versatilepb -nographic -s -S -kernel add.elf
$ qemu-system-arm -M versatilepb -nographic -s -S -kernel adds.elf
```

> A separate post on how to use QEMU will be created in the future.  
> If you have any questions, please refer to the [QEMU official documentation](https://www.qemu.org/docs/master/).

### Debug with GDB
We will use [GDB](https://man7.org/linux/man-pages/man1/gdb.1.html) to execute each instruction step-by-step and inspect the `CPSR` values after each operation.

#### [GDB Commands](https://ftp.gnu.org/old-gnu/Manuals/gdb/html_chapter/gdb_4.html#SEC12) Used
- `target remote localhost:1234` 
  - Connects to the local GDB server running on QEMU.
- `si`
  - Executes a single CPU instruction step.
- `i r r0 r1 cpsr`
  - Outputs the values of specified CPU registers:
  - `i r` : Displays the values of all CPU registers.
  - `r0 r1 cpsr` : Displays the values of `r0`, `r1`, and `cpsr` registers specifically.

#### Use "ADD" instruction

**1. Start GDB**
```shell
$ gdb-multiarch add.elf
```

**2. Connect to the local GDB Server**
```shell
(gdb) target remote localhost:1234
_start () at add.s:5
5         mov r0, #0xffffffff
```
The next instruction to be executed is displayed as `mov r0, #0xffffffff`

**3. Check Initial CPSR**
```shell
(gdb) i r r0 r1 cpsr
r0             0x0                 0
r1             0x0                 0
cpsr           0x400001d3          1073742291
```
The CPSR value, `0x400001d3` in binary is `0100 0000 0000 0000 0000 0001 1101 0011`.  
Looking at the upper 4 bits:

- N : 0
- Z : 1
- C : 0
- V : 0

This indicates that only **Z (Zero)** flag is set due to register initialization.

**4. Exectute the `mov` instruction**
```shell
(gdb) si
6         add r1, r0, #2
```
The `si` command executes the current CPU instruction.  
The `mov r0, #0xffffffff` instruction instruction is executed, and then the next instruction to be executed is `add r1, r0, #2`.

**5. Verify `CPSR` After `mov` Execution**
```shell
(gdb) i r r0 r1 cpsr
r0             0xffffffff          -1
r1             0x0                 0
cpsr           0x400001d3          1073742291
```
The `mov` instruction assisngs `0xffffffff` to the `r0` register. However, since the `s` suffix is not used, the `CPSR` flags remain unchanged, even though the Z flag should have been cleared.

**6. Execute the `add` instruction**
```shell
(gdb) si
0x00008008 in ?? ()
```
The `add r1, r0, #2` instruction is executed.

**7. Verify `CPSR` After `add` Execution**
```shell
(gdb) i r r0 r1 cpsr
r0             0xffffffff          -1
r1             0x1                 1
cpsr           0x400001d3          1073742291
```
When `add r1, r0, #2` is executed, the `CPSR` flags remain unchanged, as the `s` suffix is not used.


#### Use "ADDS" instruction
Following the same process, we will now examine the changes in register values when using `ADDS`.
```shell{hl_lines=[11,15,21]}
$ gdb-multiarch adds.elf
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
_start () at adds.s:5
5         mov r0, #0xffffffff
(gdb) i r r0 r1 cpsr
r0             0x0                 0
r1             0x0                 0
cpsr           0x400001d3          1073742291
(gdb) si
6         adds r1, r0, #2
(gdb) i r r0 r1 cpsr
r0             0xffffffff          -1
r1             0x0                 0
cpsr           0x400001d3          1073742291
(gdb) si
0x00008008 in ?? ()
(gdb) i r r0 r1 cpsr
r0             0xffffffff          -1
r1             0x1                 1
cpsr           0x200001d3          536871379
```
After following the same debugging process, the fianl `CPSR` flags is `0x200001d3`.  
In binary, `0x200001d3` is `0010 0000 0000 0000 0000 0001 1101 0011`, and the upper 4 bits are as follows:

- N : 0
- Z : 0
- **C : 1**
- V : 0

Since the result of `add` operation exceeds the 32-bit maximum value (`0xffffffff`), a **Carry occurs**. However, the addition result, `1` does not exceed the range of signed integers, so **Overflow does not occur**. As a result, the CPSR **Carry Flag is set**, while the **Overflow Flag remains clear**.

## Conclusion
In this post, we explored how **Carry** and **Overflow** are handled in ARM processors, using **assembly code examples** and **verifying their behavior with QEMU and GDB**. At the hardware level, the primary goal is to detect Carry and Overflow conditions, while handling them is the responsibility of the software. Therefore, understanding the detection conditions and methods is essential for programmers to write more efficient and robust code.

**Upcoming Posts**
- How are arithmetic operations processed in hardware?
- How are Carry and Overflow detection implemented?
- Utilizing ARM assembly with QEMU

**Feel free to share your questions or suggestions in the comments for more detailed explanations!**