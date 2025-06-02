+++
date = '2025-01-24'
title = "Verilog for Programmers: An Introduction"
tags = ["verilog", "hdl"]
draft = true
+++

To work on [From the Transistor](https://github.com/seojuncha/fromthetransistor-fork) project, I used a tool called ***Verilog***. 
Verilog is one of the [HDLs(Hardware Description Languages)](https://en.wikipedia.org/wiki/Hardware_description_language) designed for programming [FPGAs(Field Programmable Gate Arrays)](https://en.wikipedia.org/wiki/Field-programmable_gate_array). Simply put, it's a language used for designing hardware circuits.

In this post, we'll explore **how Verilog differs from other programming language** from the perspective of a software programmer and understand **how it can be helpful in various aspects**.

## What is Verilog?
If you're interesting in the history of Verilog, you can check out resources like the [Verilog wiki]() to avoid getting bogged down in details. 
From a programmer's perspective, we typically use software programming language like Python to develop software. 
Programs written in these languages are eventually translated into binary code that the computer can understand, and this binary code is then executed by the computer's hardware.

However, Verilog is a language designed for hardware, not software.
While programming language like Python are used to create programs that run on **exesting hardware**, Verilog allows you to design the **hardware itself** on which the software will eventually run.

## Why programmers have to know Verilog?
Not every programmers need to learn Verilog or other HDLs, especially those who primarily work with high-level languages. Many may never encounter the need for it.
However, at the end of the day, all programmers-regardless of their specialization-develop software that ultimately runs on computer hardware.

By learning Verilog, software programmers are
- **Understand the difference and connections between hardware and software.**
- **Design hardware optimized for a specific software needs.**
- **Expand their way of thinking by considering both hardware and software perspectives.**
- **Develop skills for integrated design, bridging the gap between software and hardware systems.**

## Difference with Software Programming Languages
Let's explore **three key differences** between Verilog and traditional programming languages. Keep in mind that all these differences stem from the fact that Verilog is designed specifically for modeling hardware.

### Sequential Logic vs Parallel Logic
Software programming languges operate sequentially, regardless of the language being used.
Programs are ultimately converted into CPU instructions, which are executed step by step in sequential manner.
In contrast, **Verilog inherently models the parallel nature of hardware operation**, and as a result, it fully supports parllelism at the language level.

> **"Isn't multithreaded programming parallel execution?"**  
> In multithreaded programming, each thread has its own program counter, which allows for independent instruction execution across threads.
> However, **individual threads still execute instructions sequentially**.

### Data type for Hardware Components
In software programming languages, data types are logical distinctions. For example:
- Integer
- String
- Object

These distinctions are **logical constructs for human understanding**, not something the computer inherently recognizes.

> Computers cannot differentiate between numbers, characters, or objects - they only understand 0s and 1s

However, since Verilog is a language designed to describe hardware behavior, its data types represent **hardware components**.

For instance, `reg` and `wire` represent basic hardware components, but their usage depends on the context. 

- `reg`: Models storage elements such as [flip-flops or latches](https://en.wikipedia.org/wiki/Flip-flop_(electronics)).
- `wire`: Models a single wire.

### Blocking vs Non-Blocking Assignment
In programming languages, variable assignment typically means storing a value in a specific memory location.
This process is sequential, so it's impossible to store two values in the same memory location simultaneously.

For example, in the following Python code:
```python
# Sequential assignment in Python
a = 3
b = a + 2
c = a * b
``` 
To compute the value of `c` the values of `a` and `b` must be determined first.
As a result, a variable must already have a value assigned to it before it can be used in an expression.

> **"What about multithreaded programming?"**  
> Even in multithreaded programming, the same principle applies.
> Even when dealing with critical sections, the issue lies in contention, not parallelism - each thread can only access memory one operation at a time.

However, Verilog supports **two types of assignment** mechanisms.

#### Sequential Assignment: Blocking
Sequential assignment using the `=` symbol behaves the same as in traditional programming languages.

```verilog
reg a, b, c;

always @(*) begin
  a = 3;
  b = a + 2;
  c = a * b;
end
```
Here, each variables -`a`, `b`, and `c`- is assigned sequentially, resulting in `c = 15`.
In hardware, this type of assignment is commonly used to design **combinational logic** circuits, such as AND/OR gates or multiplexers.

Combinational logic circuits require their outputs to change immediately based on the current inputs. They do not store any state; all outputs depend solely on the current inputs. Therefore, the order of operations must be guaranteed. Verilog naturally expresses this hardware behavior using **blocking assignments**.


#### Parallel Assignment: Non-Blocking
Non-blocking assignment using the `<=` symbol allows values to be assigned in parallel.

```verilog
reg a, b, c;

always @(posedge clk) begin
  a <= 5;
  b <= a + 2;
  c <= a * b;
end
```
In this case, the variables `a`, `b`, and `c` are **not dependent** on the values assigned within the same block. 
For example, when assigning `b`, it does not use the expected result of `a <= 5` (which is `5 + 2`), but rather the **previous state of `a`**.

In essence, non-blocking assignments are fundamentally **state-based operations**.

In hardware design, they are primarily used for creating **sequential logic** circuits, such as finite state machines (FSMs).

Sequential logic circuits operate based on both previous states and current inputs. Key components include flip-flops or latches for storing the previous state, and synchronous circuits that use a clock signal are a common example of sequential logic.

## Conclusion
From a software programmer's perspective, using Verilog can significantly aid in understanding hardware.

In this post, we’ve briefly introduced why Verilog can be useful for programmers and highlighted its key features. While the syntax itself isn’t too difficult to learn if you have programming experience, it’s crucial to **write code with an understanding of how the hardware operates**.

In the next post, we’ll explore various concepts by directly working with Verilog. Stay tuned!