+++
date = '2025-02-06'
title = "Verilog Simulation & Test Environment Setup Guide"
tags = ["verilog", "testbench", "cocotb", "icarus verilog", "gtkwave"]
draft = true
+++

Like other software programming languages, Verilog code also requires a verification process to ensure it operates as intended.

Verilog can be verified using two primary methods:

- **Hardware-based verification**
- **Software-based verification**

The hardware-based verification method involves uploading the Verilog code to an actual hardware device and checking the output signals based on the given input signals.

In contrast, the software-based verification method validates the functionality of the code purely through software, without requiring any hardware. Since hardware-based verification requires physical hardware and makes debugging signal transitions more challenging, software-based verification is typically performed first before uploading the code to hardware.

The code used to conduct this software-based verification process is called a ***Testbench***, and the environment in which the testbench is executed is referred to as a ***Simulator***.

In this post, we will explore **the concept of Verilog testbenches** and **how to set up a simulation** environment on Ubuntu and macOS.

Understanding why you need to configure a simulator suited to your development environment and write a testbench is essential. This knowledge will enable you to smoothly follow upcoming posts related to Verilog programming.

## Concept and Necessity of Testbenches: A Comparison with Software Testing Methods
A testbench is an environment that enables the verification of a Verilog-designed circuit without implementing it on actual hardware. It allows you to simulate circuit behavior, provide input signals, and observe the output.

As expected, writing a testbench follows the ***software-based verification approach***.

In software development, you can inspect output values or use debugging tools to analyze the program’s execution flow. However, in hardware design, you cannot simply print values or set breakpoints as you would in conventional programming.

Instead, a testbench provides a way to verify circuit behavior in a simulation environment. This approach is similar to how software developers perform unit testing or debugging, offering a structured way to validate Verilog code before implementing it on actual hardware.

Compared to software debugging and testing methods, Verilog testbenches have the following key differences:

|Comparison Item|C|Python|Verilog Testbench|
|:-:|:-:|:-:|:-:|
|Execution Environment|CPU|Interpreter|Simulator|
|Debugging Method|`printf`, `gdb`|`print`, `pdb`|`$display`, waveform|
|Output Inspection|Terminal output|Terminal output|Terminal output, waveform file|

## Introduction to Verilog Simulators

A Verilog testbench is essentially code written to simulate hardware behavior, and it requires an execution environment. This environment is known as a simulator. While the language used to write a testbench may vary depending on the simulator, the primary goal remains the same: ***to analyze input and output signals at specific points in time***.

> Note: A testbench is purely for testing purposes and is not synthesized into actual hardware.

Some of the most commonly used Verilog simulators include:

- `Icarus Verilog`
- `Verilator`
- `ModelSim`
- `Vendor-specific simulators` (e.g., Xilinx-provided tools for FPGA development)

For upcoming posts, we will be using `Icarus Verilog`, so I will walk you through the installation process for Icarus Verilog.

> **Do you always need to use a simulator to run a testbench?**
> 
> Not necessarily, but it is not a recommended approach.
> 
> As I will cover in a future post, it is possible to write and execute a testbench using only Verilog code without relying on a simulator. However, as designs become more complex, this approach quickly reaches its limitations.
> 
> ***The core of hardware verification lies in signal and timing analysis.***
> 
> In software, execution is sequential, so you can trace variable values and function calls to analyze changes over time. However, in hardware, execution can be parallel, meaning that rather than analyzing the execution sequence alone, you need to inspect how signals change at specific points in time.
> 
> Due to this characteristic, traditional software debugging methods are less effective for hardware verification. Instead, waveform analysis using a waveform viewer is often required.

### Simulation Installation Environment
The installation guide provided below has been tested on the following environments:
#### Ubuntu
- OS Version
  - 22.04
#### MacOS
- Processor
  - Apple Sillicon M2
- OS Version
  - Sequoia 15.1.1

### What is Icarus Verilog?
[Icarus Verilog](https://en.wikipedia.org/wiki/Icarus_Verilog) is an [open-source](https://github.com/steveicarus/iverilog) Verilog simulator that supports multiple operating systems. It is likely one of the most well-known and widely used Verilog simulators.

For detailed usage instructions, you can refer to the official [Icarus Verilog documentation](https://steveicarus.github.io/iverilog/).

#### Installing Icarus Verilog on Ubuntu
- **Installation**
```shell
$ sudo apt install iverilog
```
- **Verify Installation**
```shell
$ iverilog -v
$ vvp -v
```
If the installation was successful, you should see version information displayed in the terminal.

#### Installing Icarus Verilog on MacOS
- **Installation**
```shell
$ brew install icarus-verilog
```
- **Verify Installation**
```shell
$ iverilog -v
$ vvp -v
```
If the installation was successful, you should see version information displayed in the terminal.

### Verilog & VHDL Test Framework: cocotb
[cocotb](https://www.cocotb.org/) is a Python-based test framework that utilizes [various Verilog simulators](https://docs.cocotb.org/en/stable/simulator_support.html) as backends.

Since cocotb is written in Python, it allows you to write and execute testbenches in Python instead of Verilog, making the testing and simulation process more efficient and accessible.

#### Installing cocotb Using pip
Since cocotb is a Python-based framework, it can be installed using `pip`, making the installation process the same across all operating systems.
- **Installation**
```shell
$ pip3 install cocotb
```
- **Verify Installation**
```shell
$ pip3 list | grep cocotb
cocotb              1.9.2

$ python3 -c "import cocotb; print(cocotb.__version__)"
1.9.2

$ cocotb-config --help
```

## Waveform Analysis Tool: GTKWave
[GTKWave](https://gtkwave.sourceforge.net/) is a software tool that allows you to visually inspect hardware signal [waveforms](https://en.wikipedia.org/wiki/Waveform).

It reads waveform files (e.g., `FST`, `VCD`) generated by Verilog simulators and plots the signal transitions for each module along a timestamped timeline.

Since waveform analysis is essential for Verilog debugging, it is recommended to get familiar with its usage by referring to the [GTKWave User Manual](https://gtkwave.sourceforge.net/gtkwave.pdf).

### Building and Running GTKWave on MacOS
When installing GTKWave using brew on macOS, it may not function correctly. Therefore, it is recommended to build it manually using the following steps:
```shell
$ brew install desktop-file-utils shared-mime-info \
               gobject-introspection gtk-mac-integration \
               meson ninja pkg-config gtk+3 gtk4
$ git clone https://github.com/gtkwave/gtkwave.git
$ cd gtkwave
$ meson setup build
$ meson compile -C build
$ cd build/src
$ ./gtkwave
```
After successful compilation, you can run ./gtkwave to launch the waveform viewer.

### Installing GTKWave on Ubuntu
On Ubuntu, GTKWave can be installed easily using the package manager:

```shell
$ sudo apt install gtkwave
$ gtkwave
```

## Conclusion
In this post, we covered the verification and debugging of Verilog code using testbenches and introduced Verilog simulators. Just like in software development, **verification and debugging** are crucial steps in hardware design.

We have set up the test environment primarily using `Icarus Verilog`, but you can also use `Verilator` or `ModelSim` if needed. In future Verilog-related posts, we will gradually introduce how to utilize different simulators and test frameworks.

## Related Topics

- [**Why Software Developers Should Learn Verilog**](/posts/introduction-verilog-for-programmer/)  
  Why understanding hardware description languages can enhance a programmer’s skillset.

