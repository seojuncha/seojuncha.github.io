---
layout: post
lang: en
ref: what-is-low-level-programming
title: "What is Low-Level Programming?"
date: 2025-09-01 21:49:00 +0900
categories: programming low-level
tags: [low-level, assembly, C, ARM, Verilog, compiler, operating-system]
---

All the electronic devices we use every day—smartphones, laptops, and servers—are built on a complex combination of software and hardware.
Most people only interact with the visible parts, like websites or apps, but beneath the surface lies an unseen world: **low-level programming**.

Throughout my 10+ years of programming experience, I’ve worked with many languages and environments. But I’ve always been curious about how things actually work inside. Over time, I grew uncomfortable with code that I couldn’t clearly predict or fully understand. Even C++, the language I used most in industry, felt unsatisfying—despite being relatively close to the hardware, it still left me wondering what was truly happening underneath.

> Of course, I completely agree that scripting languages like Python are extremely useful—and I still use them all the time!

So even though my professional career hasn’t centered on low-level programming, my personal curiosity and research have led me down this path.
In this article, I’ll explore what low-level programming is, why it matters, and why we should care about it.


## What Is Low-Level Programming?

Low-level programming means working **as close to the hardware as possible**:

-	Directly manipulating CPU registers
-	Accessing specific memory addresses
-	Designing software based on hardware principles

High-level languages like Python, Java, and JavaScript abstract away these details so developers don’t need to think about hardware.
By contrast, languages like C and Assembly map almost directly to hardware operations, which is why they’re considered **low-level languages**.

Personally, I see C more as a high-level language than a low-level one. With C, you usually don’t have to rewrite code just because the hardware changes—thanks to its portability. Assembly, on the other hand, ties you directly to the CPU and memory architecture. That’s also where the compiler plays its role: bridging the gap between high-level abstractions and low-level execution.

## Key Characteristics of Low-Level Programming

1. Hardware Control
-	You can control I/O devices, memory, and CPU instructions directly.
-	Example: writing a value to a specific memory address or executing an instruction that manipulates a register.

2. Fine-Grained Optimization
-	Because code is closely tied to hardware, it can be optimized for speed and memory usage.
-	Essential for embedded systems, operating systems, and real-time control systems.

3. Complexity and Difficulty
-	With less abstraction, the learning curve is steeper.
-	But it also provides a deep understanding of how computers fundamentally work.

## Why Does It Matter?

Low-level programming isn’t just an “old-school” technique—it still plays a crucial role in many fields today:

-	**Operating Systems & Kernels:** The core of Linux, Windows, and macOS is written in C and Assembly.
-	**Drivers & Firmware:** Keyboards, mice, graphics cards, network cards—none of them work without low-level code.
-	**Security & Hacking:** Understanding vulnerabilities requires knowledge of memory structures and Assembly.
-	**Embedded Systems:** Cars, IoT devices, and appliances all rely on low-level programming.

In short: everything we use daily is powered by low-level code beneath the surface.

![github linux kernel language](/assets/img/github-linux-kernel-lanauge.png)

## Simple Examples

-	In ARM Assembly, the instruction `add r0, r1, r2` isn’t just “addition.” 
It represents the CPU’s Arithmetic Logic Unit (ALU) receiving register values, performing the operation, and storing the result.
-	In Verilog, the line `assign y = a & b;` corresponds to actual electrical signals flowing through a semiconductor circuit.
-	In C, the line `printf("Hello");` eventually invokes a system call, where the kernel manages the hardware I/O operation.

Every single line of code hides countless low-level operations.
Low-level programming isn’t just about learning difficult skills—it’s about developing a mindset of **understanding computers at their core**.

## Closing Thoughts

Low-level programming may not always be visible, but it forms the foundation of all modern technology.
My guiding framework will be George Hotz’s [From the Transistor project](https://github.com/seojuncha/fromthetransistor), 
which demonstrates how to directly build and observe low-level computer systems.

In future posts, I’ll dive into topics like:
-	**Binary Representation**
-	**ARM Assembly**
-	**Verilog**
-	**Compiler and OS Development**

Low-level programming is not just about writing code—it’s about exploring the essence of how computers work.