---
layout: post
lang: en
ref: "c-build-step"
title: "The Four Stages of C Compilation"
date: 2026-07-01 21:00:00 +0900
categories: ["programming"]
tags: ["c compile", "c programming", "c build"]
---

Using a simple sample program, we take a close look at how C code is transformed into an executable file.

It's not that I don't already understand this process; the point is to grow more comfortable with the concepts and analysis methods of each stage by explaining it myself.

The lab environment is AMD64/Linux based. Note that I mix GNU and LLVM programs throughout.
Unless you're using a very old toolchain, you should be able to follow along without trouble in most environments.

## Key Terms
**Portable**  
Means that code can run on a different platform without modification.
**C code itself is portable**. The same source code can be used on both the AMD64 and AArch64 architectures.
However, **the compiled executable is not portable**. An executable compiled for AMD64 cannot run on AArch64.

**Translation Unit**  
Refers to a single source file as the compiler interprets it. Each source file has its own Translation Unit (TU).
As the name implies, it is the **unit of translation** from the compiler's point of view.

**Instruction Set**  
The set of instructions a CPU processes. **A CPU interprets various arithmetic/logic operations in binary-code form to determine what work it should perform**.
For example, it's like pre-defining that the binary code `1111` means an addition operation and `1100` means a subtraction operation.
The catch is that this mapping differs for every kind of CPU.

**Object File Format**  
Regardless of its kind, an object file is ultimately a collection of binary-code instructions or data that the hardware can execute.
Modern Linux uses **ELF (Executable and Linkable Format)** as its default object file format.

## The Compilation (Build) Pipeline
The process by which a C source file is transformed into an executable object file takes the form of a kind of pipeline.
It can be broadly divided into four stages, and because each stage uses the previous stage's output as its input, it can be defined as a pipeline.

**The four stages of the compilation pipeline:**
- Preprocessing
- Compilation
- Assembly
- Linking

Let's use the simplified sample code below to see how the whole compilation process works.

**sum.h**
```c
#ifndef SUM_H
#define SUM_H

int sum(int a, int b);

#endif
```

**sum.c**
```c
#include "sum.h"

#define WORK 1
#define PLUS +

typedef int i32;

/* block comment */
i32       sum(i32 a,    i32 b) {   // line comment
#if WORK
    return a PLUS b;
#else
    return 0;
#endif
}
```

**main.c**
```c
#include "sum.h"

int main(void) {
  return sum(3, 5);
}
```

## Stage 1. Preprocessing
- **Input**: C source file
- **Output**: Preprocessed source file
- **Preprocessor**: `cpp`, `clang-cpp`

The first stage is preprocessing. This is a distinctive feature of C/C++ that most other languages don't provide.

The preprocessing stage performs four main jobs:
1. Header file inclusion
2. Macro expansion
3. Conditional compilation
4. Source code cleanup

When you use a tool like `gcc`, you usually don't have to think much about this, but `gcc` is more of an integrated tool for conveniently handling each stage of the compilation pipeline; in reality a separate tool is used for each stage. Among these, the one used in the preprocessing stage is `cpp`.

```bash
$ gcc -E sum.c -o sum.i   # with gcc, the -E option lets you inspect the preprocessed result
$ cpp sum.c -o sum.i
```

Let's look at `sum.i`, generated as the result of preprocessing `sum.c`, and see what the preprocessing stage handled.

**sum.i**
```c
# 0 "sum.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 0 "<command-line>" 2
# 1 "sum.c"
# 1 "sum.h" 1



int sum(int a, int b);
# 2 "sum.c" 2




typedef int i32;


i32 sum(i32 a, i32 b) {

    return a + b;



}
```

**Header file inclusion**
- The `#include "sum.h"` statement was replaced with the contents of that header file.

**Macro expansion**
- Because of `#define PLUS +`, the statement `return a PLUS b;` was expanded to `return a + b;`.

**Conditional compilation**
- Because of `#define WORK 1` and the `#if WORK, #else, #endif` conditions, `return 0;` was removed.

**Source code cleanup**
- The unnecessary whitespace between tokens in `i32       sum(i32 a,    i32 b)` was removed.
- Comments were also removed. Comments are for the programmer and have nothing to do with execution, so they are stripped out beforehand.

For reference, lines such as `# 2 "sum.c" 2` are called *linemarkers*.

A point to keep in mind: **the preprocessing stage does not include C-grammar-based parsing.**

### Evidence that preprocessing does no grammar-based parsing
If you examine the result of preprocessing the code below, you can see that the preprocessor doesn't look at C grammar but does check preprocessing-directive syntax.

**pp.h**
```c
#include <stdio.h>

#define text 100

This is my text.

```

```bash
$ cpp pp.h -o pp.i
```

The `pp.h` header file contains an `#include` statement, a `#define`, and one ordinary sentence.
A lone sentence like `This is my text.` is syntax that wouldn't parse, but because the preprocessing stage doesn't parse C grammar, it produces no particular error message.

That said, you can confirm at the very bottom of the `pp.i` file that, per the preprocessing behavior, `text` is replaced with `100`.

## Stage 2. Compilation
- **Input**: Preprocessed source file
- **Output**: Assembly file
- **Compiler**: `gcc`, `clang`

The compilation stage converts the preprocessed source file into an assembly file matching the host architecture.
This means that a different assembly file is generated depending on the hardware the final executable runs on.

Why must assembly files be separated by architecture?

A program written in C is portable. Being portable means it can run on different architectures (e.g., AMD64, AArch64, etc.) without changing the source code.
This portability is precisely thanks to the assembly files generated separately for each architecture.

```
 C Source   ────┬─  Assembly (AMD64)    ─────   Machine Code (AMD64)
                └─  Assembly (AArch64)  ─────   Machine Code (AArch64)
```

Setting aside directives, an assembly instruction generally corresponds to a single CPU machine instruction.
Each platform (architecture) has a different ***instruction set*** and provides different assembly instructions.
Not only that, the CPU register structure and names differ as well.

The upshot is that, for the sake of portability, different assembly code must be generated for each platform.
The details on this are covered below in "Stage 3. Assembly."

To generate assembly code, you can use the `gcc -S` option.

```bash
$ gcc -S sum.i -o sum.s
$ gcc -S main.i -o main.s
```

To express the pipeline's input/output connections I used the `.i` files as input, but it's more common to pass `.c` files as arguments to generate assembly files.

The compiler treats each preprocessed source file as an individual ***TU (Translation Unit)***.
You can think of a TU as the single unit the compiler processes.
In other words, each source file the compiler compiles becomes one TU.

**sum.s**
```
        .file	"sum.c"
        .text
        .globl	sum
        .type	sum, @function
sum:
.LFB0:
        .cfi_startproc
        endbr64
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        movl    %edi, -4(%rbp)
        movl    %esi, -8(%rbp)
        movl    -4(%rbp), %edx
        movl    -8(%rbp), %eax
        addl    %edx, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   sum, .-sum
        .ident  "GCC: (Ubuntu 11.4.0-1ubuntu1~22.04.3) 11.4.0"
        .section        .note.GNU-stack,"",@progbits
        .section        .note.gnu.property,"a"
        .align 8
        .long   1f - 0f
        .long   4f - 1f
        .long   5
0:
        .string "GNU"
1:
        .align 8
        .long   0xc0000002
        .long   3f - 2f
2:
        .long   0x3
3:
        .align 8
4:

```

The generated assembly file describes the behavior of the `sum()` function in AMD64 assembly.
- `.text`
  - Located in the text (code) section; that is, executable code rather than static data.
- `.globl sum`
  - The linkable global symbol sum.
- `pushq %rbp`, `popq %rbp`
  - Saving and restoring the `rbp` register for the function call and return.
- `movq %rsp, %rbp`
  - Creates the stack frame.
- `movl %edi, -4(%rbp)`, `movl %esi, -8(%rbp)`
  - Pushes the two function arguments held in the `edi` and `esi` registers onto the stack.
- `movl -4(%rbp), %edx`, `movl -8(%rbp), %eax`
  - Reads the function arguments stored on the stack into the `edx` and `eax` registers.
- `addl %edx, %eax`
  - Adds the values of the two registers.
- `ret`
  - Returns to the main function.

So how was the compiler able to convert C source code into assembly code?

This process is the very heart of compiler theory. However, it's beyond the scope of this post, so I'll cover it in more detail in another post.

**How does it know whether the code follows C grammar?**  
Back when I built a [C compiler](https://github.com/seojuncha/ftt-c-compiler), setting aside the theoretical parts, the first thing I was curious about was how to split and classify tokens according to C grammar.

We learn C grammar through lectures or textbooks and write C programs, but that wasn't enough to actually build a compiler.
The C language's grammar is described in detail in the [ISO document](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf).

### Platform-dependent assembly code
You can confirm that even with the same C code and the same compiler, different assembly code can be generated.

For example, look at the assembly code targeting the AArch64 architecture.

```bash
$ aarch64-none-linux-gnu-gcc -S sum.c -o sum_arm64.s

$ cat sum_arm64.s

        .arch armv8-a
        .file   "sum.c"
        .text
        .align  2
        .global sum
        .type   sum, %function
sum:
.LFB0:
        .cfi_startproc
        sub     sp, sp, #16
        .cfi_def_cfa_offset 16
        str     w0, [sp, 12]
        str     w1, [sp, 8]
        ldr     w1, [sp, 12]
        ldr     w0, [sp, 8]
        add     w0, w1, w0
        add     sp, sp, 16
        .cfi_def_cfa_offset 0
        ret
        .cfi_endproc
.LFE0:
        .size   sum, .-sum
        .ident  "GCC: (Arm GNU Toolchain 15.2.Rel1 (Build arm-15.86)) 15.2.1 20251203"
        .section        .note.GNU-stack,"",@progbits
```

Even using the same `sum.c` file, it outputs different assembly code.

## Stage 3. Assembly
- **Input**: Assembly file
- **Output**: Relocatable Object File
- **Assembler**: `as`, `llvm-as`

The assembly file generated by the compiler backend's CodeGen module is translated into machine code by the assembler (`as`) matching the platform.
Machine code expresses a different instruction set for each platform.

```bash
$ as sum.s -o sum.o
$ as main.s -o main.o
```

The command that can do everything up to this point — preprocessing, compilation, and assembly — in one go is `gcc -c`.
```bash
$ gcc -c sum.c -o sum.o
$ gcc -c main.c -o main.o

$ file main.o

main.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

The generated ***relocatable object file*** contains binary code compatible with the target architecture, per the assembly instructions.

Let's check the disassembly result of the `sum.o` and `main.o` object files with the `objdump -d` command.

```bash
$ objdump -d sum.o

sum.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <sum>:
   0:   f3 0f 1e fa             endbr64
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   89 7d fc                mov    %edi,-0x4(%rbp)
   b:   89 75 f8                mov    %esi,-0x8(%rbp)
   e:   8b 55 fc                mov    -0x4(%rbp),%edx
  11:   8b 45 f8                mov    -0x8(%rbp),%eax
  14:   01 d0                   add    %edx,%eax
  16:   5d                      pop    %rbp
  17:   c3                      ret
```

As shown in the table below, each assembly line maps to a single binary executable code.

| Assembly | Binary Code |
| - | - |
| `endbr64` | `0xF30F1EFA` |
| `push %rbp` | `0x55` |
| `mov %rsp, %rbp` | `0x4889E5` |
| `mov %edi,-0x4(%rbp)` | `0x897DFC` |
| `mov %esi, -0x8(%rbp)` | `0x8975F8` |
| `mov -0x4(%rbp), %edx` | `0x8B55FC` |
| `mov -0x8(%rbp), %eax` | `0x8B45F8` |
| `add %edx, %eax` | `0x01D0` |
| `pop %rbp` | `0x5D` |
| `ret` | `0xC3` |


```bash
$ objdump -d main.o

main.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:   f3 0f 1e fa             endbr64
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   be 05 00 00 00          mov    $0x5,%esi
   d:   bf 03 00 00 00          mov    $0x3,%edi
  12:   e8 00 00 00 00          call   17 <main+0x17>
  17:   5d                      pop    %rbp
  18:   c3                      ret
```

You can see that binary code for main's assembly instructions like `endbr64` or `push %rbp` exactly matches the binary code we saw in the sum function.

Let's also look briefly at main's behavior. (Parts identical to the sum function are omitted.)
- `mov    $0x5,%esi`
  - Stores the integer 5 in the `esi` register.
- `mov    $0x3,%edi`
  - Stores the integer 3 in the `edi` register.
- `call   17 <main+0x17>`
  - The `call` instruction calls the function at the address that follows, but since it hasn't been relocated yet, it points to the address of the next instruction.

Let's also disassemble `sum_arm64.o`, compiled for AArch64. At a glance you can see it's different from the AMD64 object file.

```bash
$ aarch64-none-linux-gnu-objdump -d sum_arm64.o

sum_arm64.o:     file format elf64-littleaarch64


Disassembly of section .text:

0000000000000000 <sum>:
   0:   d10043ff        sub     sp, sp, #0x10
   4:   b9000fe0        str     w0, [sp, #12]
   8:   b9000be1        str     w1, [sp, #8]
   c:   b9400fe1        ldr     w1, [sp, #12]
  10:   b9400be0        ldr     w0, [sp, #8]
  14:   0b000020        add     w0, w1, w0
  18:   910043ff        add     sp, sp, #0x10
  1c:   d65f03c0        ret
```

This proves that, even for identical hardware behavior, the compatible binary executable code and assembly code differ depending on the architecture.

For example,
AMD64's addition operation was `add %edx, %eax`, whereas AArch64's addition operation takes the form `add w0, w1, w0`.
Their respective binary executable codes likewise differ, as you can confirm from `0x01D0` versus `0x0B000020`.

## Stage 4. Linking
- **Input**: Relocatable object file
- **Output**: Executable Object File, Shared Object File
- **Linker**: `ld`, `lld`

So far the compiler has processed each source file as one TU, and the assembler has generated the respective relocatable object files.
These object files, generated this way, cannot be executed on their own.
These object files merely describe what machine code each source file was translated into.
Bundling each object file into one so it can be loaded into process memory is linking, and that is the linker's role.

So how can data defined across multiple object files be gathered into one?

The linker uses a kind of label called a ***symbol***, which each object file carries, to connect the actual binary code.

Beyond this, the linker does various things during the linking process, so directly using `ld` to link is something I'll cover in another post.

For now, let's link simply using only `gcc`.

```bash
$ gcc main.o sum.o   # after linking, generates a.out

$ file a.out
a.out: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),  .. <omitted>

$ ./a.out || echo $?
8                       # returns 8, the result of the sum call
```

```bash
$ objdump -d a.out

...<omitted>...

0000000000001129 <sum>:
    1129:       f3 0f 1e fa             endbr64
    112d:       55                      push   %rbp
    112e:       48 89 e5                mov    %rsp,%rbp
    1131:       89 7d fc                mov    %edi,-0x4(%rbp)
    1134:       89 75 f8                mov    %esi,-0x8(%rbp)
    1137:       8b 55 fc                mov    -0x4(%rbp),%edx
    113a:       8b 45 f8                mov    -0x8(%rbp),%eax
    113d:       01 d0                   add    %edx,%eax
    113f:       5d                      pop    %rbp
    1140:       c3                      ret

0000000000001141 <main>:
    1141:       f3 0f 1e fa             endbr64
    1145:       55                      push   %rbp
    1146:       48 89 e5                mov    %rsp,%rbp
    1149:       be 05 00 00 00          mov    $0x5,%esi
    114e:       bf 03 00 00 00          mov    $0x3,%edi
    1153:       e8 d1 ff ff ff          call   1129 <sum>
    1158:       5d                      pop    %rbp
    1159:       c3                      ret
```

You can see that the binary executable code defined in both `sum` and `main` has all been inserted into a single `a.out` file.

### Object file symbols
Let's check the symbols defined in the relocatable object files (`sum.o`, `main.o`) generated before linking, using the `nm` command.

> `nm` is a GNU utility program that shows the symbols defined in an object file. For the meaning of the letter to the left of each function name below, see `man nm`.

```bash
$ nm sum.o main.o

sum.o:
0000000000000000 T sum

main.o:
0000000000000000 T main
                 U sum
```

`main.o` (the object file compiled from `main.c`) has two symbols, `main` and `sum`.
However, the `sum` symbol is `U`, meaning it's in an *Undefined* state. It means the symbol is not defined.

This is because the binary executable code labeled (symbolized) with the name **sum** — that is, the function's definition — is defined in the `sum.o` file, not in the `main.o` file.

So the linker has to gather the symbol information in order to merge binary executable code located in different places.
Let's check the symbols again in the final generated executable object file.

```bash
$ nm a.out

0000000000001141 T main
0000000000001129 T sum
```

The `sum` symbol has changed to `T` (executable code in the `.text` section).
