+++
date = '2025-02-18'
title = 'Introduction to the From the Transistor Project and Progress'
tags = ["from the transistor", "geohot", "george hotz"]
draft = true
+++

The **From the Transistor** project was initiated with the goal of understanding all technological stages from hardware to software. In this article, I would like to share **the purpose of the project, its current progress, and future plans**.

I have been working as a programmer for about 10 years, primarily developing software in the application domain using C/C++. However, I have always been curious about the operations and principles of low-level systems. Additionally, programmers fundamentally write software that operates on computer systems. Thus, I have often thought that a true programmer—or even a hacker—should have a solid understanding of fundamental principles starting from the low-level hardware level.

> Aren't you curious about what a programmer who understands both hardware and software can achieve?!

Following this curiosity, about 1–2 years ago, I came across a project called [From the Transistor to the Web Browser](https://github.com/geohot/fromthetransistor) introduced by George Hotz. I had been interested in this project for a while and had attempted it multiple times, only to give up before completion. However, I recently felt a strong urge to take on the challenge again. Moreover, I could not find a properly completed version of this project online, even years after its introduction, which further fueled my determination.

> **Who is George Hotz?**
>
> [George Hotz](https://en.wikipedia.org/wiki/George_Hotz) is an American programmer, well known as a hacker for jailbreaking Apple's iPhone and hacking Sony's PS3. He is also known by the alias **geohot**, and I primarily follow his work through [his YouTube channel](https://www.youtube.com/@geohotarchive). Since his YouTube channel uploads recordings of Twitch livestreams, most videos are quite long, typically 3–5 hours. However, watching these videos provides insight into how he approaches and solves problems, which is highly motivational.

### Two Key Failure Factors
The two main reasons why I could not complete the **From the Transistor** project in the past were:

1. **Abstract and vague explanations**
2. **High project difficulty**

First, each section of the project lacks detailed explanations, making it difficult to define the project scope without prior knowledge or experience. There are many unfamiliar terms, and it is unclear how each concept should be applied in practical exercises.

![geohot's building a UART comment](img/building-uart-comments-geohot.png)

*A project for writing a UART module in Verilog. The application of MMIO concepts varies depending on implementation and verification methods. But what exactly is semihosting?*

Second, while difficulty is subjective, I believe most would agree that this project is challenging. Even an embedded programmer working in a related field would find it difficult to build a compiler, an OS, and a TCP stack from scratch.

> Not to mention that Verilog is not commonly used in software development.

Additionally, the project timeline makes it hard to complete. Even **geohot, one of the most renowned hackers in the world, planned a full-time 12-week schedule**. That’s approximately three months, and given the difficulty level, a full-time employee or freelancer attempting to complete it in that time frame is likely to become exhausted and give up.

### Staying on Track
Having experienced previous failures, my **top priority this time is to complete the project without stopping, even if I feel it is not perfect**.

However, I will ensure that I do not deviate from the original project scope. Since there is no precise project interpretation, I will define the scope myself.

For example, while implementing an assembler and CPU, I could define my own **Instruction Set Architecture (ISA)** to reduce data size. However, this would make proper verification difficult. Thus, while simplifications for functionality are acceptable, I will avoid deviating too much from the intended scope. The same applies to implementing an **ARM7-based CPU**. I will exclude debugging-related ports and user-mode controls found in actual ARM CPUs, focusing instead on pipeline operation.

Since I am not an FPGA programmer, I will not be concerned about whether the Verilog program **synthesizes correctly, runs on an FPGA board, or meets performance requirements** at this stage. Once the project is more developed and I have a clearer understanding, I may attempt these aspects in the future.

## Project Overview
This project aims to **understand and practice the link between hardware and software step by step**. Verilog will be used to understand hardware operation, and software simulators will be used to test the hardware. By gradually implementing software on top of the hardware stack, I aim to gain a comprehensive understanding of how hardware and software interact.

While specific tools and programs are suggested, I will not strictly adhere to them as long as they serve the project's purpose. For instance, I have chosen **Icarus Verilog** instead of **Verilator** for Verilog simulation in **cocotb**, as Verilator has some stability issues when working with cocotb, and high performance is not a priority at this stage.

Each section of the project requires a fully functional program implementation. Since each stage builds upon the previous one, skipping steps is not an option.

### Project Goals
Rather than striving for perfect implementation and validation of each session, I plan to move on to the next section once the basics are complete, allowing me to grasp the overall structure first. For example, verifying a **UART controller** written in Verilog with a simulator like cocotb or Verilator is challenging. Instead, I will focus on hacking and leveraging existing systems to validate my own implementations.

### Major Technical Stacks by Section and Expected Outcomes
Each project section must produce a fully functional program or code.

Verilog modules will be simulated using cocotb, meaning testbenches will be written in Python. From the **third section onward, once the compiler is implemented, programs written in C will be executed using cocotb or QEMU**.

Peripheral modules such as **UART, Ethernet, and SD card controllers** can be written in Verilog and simulated with cocotb, but accurately simulating their hardware behavior is difficult. The best approach would be to add **a custom UART device in QEMU**, but since that is not critical to project completion, I will simplify the implementation.

**Note!**
> The overall design and I/O operations are still unclear.  
> I will provide more precise explanations once the project is complete.  
> Please stay tuned!
