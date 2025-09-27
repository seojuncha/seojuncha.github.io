---
layout: post
lang: en
ref: "signed-and-unsigned-binary"
date: 2025-09-27 22:00:00 +0900
title: "Binary Representation: Signed vs Unsigned"
tags: ["signed binary","unsigned binary", "binary representation"]
---

In this article, we will explore the **differences between Signed Binary and Unsigned Binary**, **their respective representation methods**, and **considerations when programming in C**.

From the user's perspective, data types handled by computers can be broadly categorized as either **characters** or **numbers**. However, from the computer's perspective, even characters are ultimately processed as numbers. Therefore, today, we will focus on the numeric data types, specifically integers, and how they are handled by computers.

<figure>
  <img src="https://upload.wikimedia.org/wikipedia/commons/b/b3/ASCII-Table.png" style="display: block; margin: auto;" alt="wiki-ascii-table">
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
An ASCII table is used to represent characters as numbers.
  </figcaption>
</figure>

## Concept of Integers
Numerical data in computers can be broadly divided into integers and real numbers. In mathematics, numbers are classified based on their characteristics as shown below.

### Integers and Real Numbers
<figure>
  <img src="https://upload.wikimedia.org/wikipedia/commons/4/43/Classification_of_Numbers.jpg" style="display: block; margin: auto;" alt="classification-of-real-number">
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
Definition of integers within the range of real numbers, which include positive numbers, negative numbers, and zero.
  </figcaption>
</figure>

### Meaning of Integers in Computers
All data is stored in a computer's memory as binary. This means that **storing an integer in a computer's memory occupies a specific memory space**. In daily life, we use decimal representation, where each digit (e.g., tens, hundreds) can range from 0 to 9, allowing for 10 different values per place (hence the term "decimal").

> __Memory Space?__  
> The term "memory space" is used because data is not always stored in main memory (RAM). For instance, data can also be stored in CPU registers. For convenience, we will use "memory" interchangeably in this article.

Computers process all data as binary (0s and 1s). Thus, to represent any data, including numbers, we need a binary representation. However, this is merely a difference in representation; the meaning remains the same. For instance, the number '5' is represented as '101' in binary but remains the same numerical value.

As previously defined, integers consist of positive numbers, negative numbers, and zero. In computers, integers are represented using **Signed Integers** and **Unsigned Integers**.

- **Signed Integers** include both positive (+) and negative (-) signs in memory.
- **Unsigned Integers** assume a positive sign by default and only store the magnitude of the number in memory.

This distinction is significant because **the same binary data can represent different values depending on whether it is treated as signed or unsigned**.

## Unsigned Binary: Concept and Conversion
Unsigned Binary represents unsigned integers in binary. Since there is no sign bit, straightforward mathematical conversion between binary and decimal applies.

> __Natural Numbers__  
> Unsigned integers are equivalent to natural numbers.

### Concept

For example, in a 4-bit space, the binary, decimal, and hexadecimal representations of unsigned integers are as follows:

__4-bit Unsigned Binary__

| Binary (bin) | Decimal (dec) | Hexadecimal (hex) |
|:-:|:-:|:-:|
|0000|0|0x0|
|0001|1|0x1|
|0010|2|0x2|
|0011|3|0x3|
|0100|4|0x4|
|0101|5|0x5|
|0110|6|0x6|
|0111|7|0x7|
|1000|8|0x8|
|1001|9|0x9|
|1010|10|0xA|
|1011|11|0xB|
|1100|12|0xC|
|1101|13|0xD|
|1110|14|0xE|
|1111|15|0xF|

Thus, the decimal range of a 4-bit unsigned integer is $0 \leq \Z \leq 2^4-1$, and the hexadecimal range is from `0x0` to `0xF`.

> __Hexadecimal Representation by Bit Size__  
> Hexadecimal is widely used to represent numbers in computer systems. Here are the maximum hexadecimal values based on bit size:  
> 4 bits  = 0xF  
> 8 bits  = 0xFF  
> 16 bits = 0xFFFF  
> 32 bits = 0xFFFF_FFFF

### Conversion
#### Converting Decimal to Binary
To convert a decimal number to binary, use division (/) and modulus (%) operations as follows:

| Division (/) | Modulus (%) |
|:-:|:-:|
|45 / 2 = 22|45 % 2 = 1|
|22 / 2 = 11|22 % 2 = 0|
|11 / 2 = 5|11 % 2 = 1|
|5 / 2 = 2|5 % 2 = 1|
|2 / 2 = 1|2 % 2 = 0|
|1 / 2 = 0|1 % 2 = 1|

$$
\begin{aligned}
45_{10} &= 22 \times 2 + 1 \newline
&= (11 \times 2 + 0) \times 2 + 1 \newline
&= ((5 \times 2 + 1) \times 2 + 0) \times 2 + 1 \newline
&= (((2 \times 2 + 1) \times 2 + 1) \times 2 + 0) \times 2 + 1 \newline
&= ((((1 \times 2 + 0) \times 2 + 1) \times 2 + 1) \times 2 + 0) \times 2 + 1 \newline
&= (((((0 \times 2 + 1) \times 2 + 0) \times 2 + 1) \times 2 + 1) \times 2 + 0) \times 2 + 1 \newline
&= 1 \times 2^5 + 0 \times 2^4 + 1 \times 2^3 + 1 \times 2^2 + 0 \times 2^1 + 1 \newline
&= 101101_2
\end{aligned}
$$

#### Converting Binary to Decimal
Binary numbers can be converted to decimal using positional notation:

$$
\begin{aligned}
101_2 &= 1 \times 2^2 + 0 \times 2^1 + 1 \times 2^0  \newline
&= 1 \times 4 + 0 \times 2 + 1 \times 1 \newline
&= 5_{10}
\end{aligned}
$$

#### Using Python for Conversion
Using Python's REPL is often the simplest way to convert between binary, decimal, and hexadecimal. Note that the `bin` and `hex` functions return strings, not integers.

```python
>>> bin(1)
'0b1'
>>> bin(2)
'0b10'
>>> int(0b11)
3
>>> int(0b110)
6
>>> hex(0b1)
'0x1'
>>> hex(1)
'0x1'
>>> int(0xf)
15
```

## Signed Binary: Representation and Python Usage
Signed Binary represents signed integers in binary. Unlike unsigned binary, signed binary requires bit patterns to represent the sign. Although there are three methods to represent signed binary, modern computers primarily use the two's complement method.

### Representation Methods
The basic principle for representing negative numbers is to **allocate one bit (0 or 1) for the sign**:
- 0: Positive
- 1: Negative

#### Method 1: Sign-Magnitude
In the sign-magnitude method, the most significant bit (MSB) is used to indicate the sign.
For example, in an 8-bit space, the binary representation of +5 and −5 is:
```
+5 = 0000 0101
-5 = 1000 0101
```
Here, ***0101*** represents the number ***5***, and the MSB is set to 0 (positive) or 1 (negative).
In an 8-bit space, the range of representable values using this method is summarized as follows:

| Binary | Decimal |
|:-:|:-:|
|0000 0000|+0|
|0000 0001|+1|
|0000 0010|+2|
|...|...|
|0111 1111|+127|
|1000 0000|-0|
|1000 0001|-1|
|...|...|
|1111 1111|-127|

Thus, this method allows the representation of 254 unique values: `+0` to `+127` and `-0` to `-127`.

#### Method 2: One's Complement
In one's complement, the binary representation of an unsigned integer is inverted (0 becomes 1 and 1 becomes 0).

__Bit Inversion__
```
0 -> 1
1 -> 0
```
__Representation of `±5` in 8-bit One's Complement__
```
+5 = 0000 0101
-5 = 1111 1010
```
The range of representable values in 8-bit one's complement is summarized as:

| Binary | Decimal |
|:-:|:-:|
|0000 0000|+0|
|0000 0001|+1|
|...|...|
|0111 1110|+126|
|0111 1111|+127|
|1000 0000|-127|
|1000 0001|-126|
|...|...|
|1111 1110|-1|
|1111 1111|-0|

#### Method 3: Two's Complement
In two's complement, the binary representation is obtained by inverting all bits and adding 1 to the least significant bit (LSB). This method is widely used in modern computers as it addresses the shortcomings of the previous methods.

To represent a negative number in two's complement:
1. Compute the one's complement of the unsigned integer.
2. Add 1 to the LSB.

__Representation of ±5 in 8-bit Two's Complement__
```
+5 = 0000 0101
-5 = 1111 1010 (One's Complement)
   +         1
   -----------
   = 1111 1011
```

| Binary | Decimal | Transformation (Positive → One's Complement → +1) |
|:-:|:-:|:-:|
|0000 0000|+0|
|0000 0001|+1|
|0000 0010|+2|
|...|...|
|0111 1110|+126|
|0111 1111|+127|
|1000 0000|-128|1000 0000 → 0111 1111 → 1000 0000|
|1000 0001|-127|0111 1111 → 1000 0000 → 1000 0001|
|1000 0010|-126|0111 1110 → 1000 0001 → 1000 0010|
|...|...|
|1111 1101|-3|0000 0100 → 1111 1011 → 1111 1101|
|1111 1110|-2|0000 0010 → 1111 1101 → 1111 1110|
|1111 1111|-1|0000 0001 → 1111 1110 → 1111 1111|

#### Using Python for Two's Complement Conversion
```python
>>> int("-128", 10).to_bytes(1, byteorder="little", signed=True)
b'\x80'

>>> int("-3", 10).to_bytes(1, byteorder="little", signed=True)
b'\xfd'
>>> bin(0xfd)
'0b11111101'
```

## Signed vs Unsigned Comparison
A summary of the key differences between signed and unsigned integers is as follows:

| Category | Signed Integer | Unsigned Integer |
|:-|:-|:-|
| Value Range | Negative, 0, Positive | 0, Positive |
| Range (8-bit) | -128 to 127 | 0 to 255 |
| Use Cases | Sensors, Offsets, etc. | Memory Addresses, Counters/Timers |
| Sign Bit | Present (MSB) | Absent |

The representable range based on two's complement for signed integers and unsigned integers can be summarized mathematically:

__Unsigned Integer__
$$
0 \leq Z \leq 2^{\text{bit size}} - 1
$$

__Signed Integer__
$$
-2^{\text{bit size} - 1} \leq Z \leq 2^{\text{bit size} - 1} - 1
$$

## Programming Considerations
As seen, the interpretation of binary data changes depending on whether it is signed or unsigned. Understanding these differences is crucial when writing programs. Let’s look at two common pitfalls in C programming.

### Example 1: Comparing Signed and Unsigned Integers
A common mistake is comparing integers of different signedness. In the example below, the signed integer `-1` is treated as an unsigned integer during comparison, resulting in unexpected behavior.

__comp.c__
```c
#include <stdio.h>

void main(void)
{
  int a = -1;
  unsigned b = 1;

  if (a < b) {
    printf("a is less than b\n");
  } else {
    printf("a is greater than or equals to b\n");
  }
}
```
__Output__
```
$ gcc comp.c && ./a.out
a is greater than or equals to b
```
Although intuitively `-1` is less than `1`, the signed integer `a` is converted to an unsigned integer, resulting in a value greater than `b`.

> __Bit Pattern of `int a = -1`__  
> a = -1 = 0xFFFFFFFF (Two's complement representation of -1) = 4,294,967,295

Use the [-W compiler option](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#index-Wsign-compare) to prevent such issues:
```bash
$ gcc -W comp.c
comp.c: In function ‘main’:
comp.c:8:9: warning: comparison of integer expressions of different signedness: ‘int’ and ‘unsigned int’ [-Wsign-compare]
    8 |   if (a < b) {
      |         ^
```

### Example 2: Type Casting
Another common mistake involves type casting between signed and unsigned integers. When casting, the value’s bit pattern remains unchanged, but its interpretation changes.

__main.c__
```c
#include <stdio.h>
#include <stdint.h>

void main(void)
{
  int8_t i8 = -5;  // 1111 1011
  uint8_t u8 = 5;  // 0000 0101
  uint8_t ui8 = (uint8_t)i8;  // 1111 1011

  printf("UINT8_MAX: %d\n", UINT8_MAX);
  printf("INT8_MAX: %d\n", INT8_MAX);

  printf("0x%02x [%d]\n", i8, i8);
  printf("0x%02x [%d]\n", u8, u8);
  printf("0x%02x [%d]\n", ui8, ui8);
}
```
__Output__
```bash
$ gcc main.c && ./a.out
UINT8_MAX: 255
INT8_MAX: 127
0xfffffffb [-5]     # (1)
0x05 [5]            # (2)
0xfb [251]          # (3)
```

- **Sign Extension**: When printing `i8`, the signed integer is promoted to `int`, causing sign extension (filling higher bits with the MSB of `i8`).
- **Bit Pattern Interpretation**: The value of `ui8` is interpreted as an unsigned integer, resulting in `251` instead of `-5`.

## Conclusion
This article explored the differences between **Signed** and **Unsigned Binary**, their representation methods, and programming considerations.

- **Signed Binary**: Represents both positive and negative values (e.g., sensor data, offsets).
- **Unsigned Binary**: Represents only positive values (e.g., memory addresses, timers).

**Did this article help you? Leave your questions or feedback in the comments below!**

