+++
date = '2025-01-11'
title = "Two's Complement: Benefits & Applications"
tags = ["binary arithmetic", "two's complement"]
math = true
+++
Two's complement is the most widely used method for representing negative numbers in modern computer systems. Unlike the sign-magnitude or one's complement representations discussed in a previous post, two's complement avoids many of their drawbacks, making it the standard choice in most systems today.

In this post, we will explore why two's complement is used, its advantages, and how it simplifies arithmetic operations. We'll also demonstrate its usage through C and Python code examples. By the end of this post, you will understand the critical role of two's complement in system design and programming.


## Why Use Two's Complement?
### Simplicity in Arithmetic Operations
One's complement requires additional steps to handle arithmetic operations. For example, when subtracting 3 from 5 using one's complement, the results differ from the expected arithmetic result.

Example: 5 - 3

Arithmetic equation: 5 - 3 = 5 + (-3)

Using one's complement representation:
```
  0000 0101 (+5)
+ 1111 1100 (-3 in 1's complement)
--------------
 10000 0001 (Carry generated, result: 1)
```
Using two's complement representation:
```
  0000 0101 (+5)
+ 1111 1101 (-3 in 2's complement)
--------------
 10000 0010 (Carry ignored, result: 2)
```
With one's complement, an additional step of adding the carry is required to obtain the correct result. In contrast, two's complement automatically yields the correct result, even if the carry is ignored. This simplicity makes two's complement far more efficient for hardware design and arithmetic processing.

> A future post will cover carry bits that arise when exceeding the bit range.

### A Single Representation for Zero
In both sign-magnitude and one's complement, zero can have two representations: +0 and –0. This inconsistency requires additional hardware to handle. In contrast, two's complement has only one representation for zero: a number with all bits set to 0.

**Zero in One's Complement**
```
+0: 0000
-0: 1111
```
**In Two's Complement**: Zero is represented as 0000 (all bits set to 0).

## How to Compute Two's Complement
To compute the two's complement of a number:  
1. Invert all the bits of the unsigned binary integer (one's complement).
2. Add 1 to the result.

**Example Table: 4-bit Unsigned and Two's Complement Representation**
|Binary|Unsigned|Signed|
|:-:|:-:|:-:|
|0000|0|0|
|0001|1|1|
|0010|2|2|
|0011|3|3|
|0100|4|4|
|0101|5|5|
|0110|6|6|
|0111|7|7|
|1000|8|-8|
|1001|9|-7|
|1010|10|-6|
|1011|11|-5|
|1100|12|-4|
|1101|13|-3|
|1110|14|-2|
|1111|15|-1|

### Implementing Two's Complement in Python
```python
def twos_complement(value, bits):
  if value < 0:
    value = (1 << bits) + value
  return bin(value)[2:]
```
The formula `(1 << bits) + value `works as follows:

Example: `value = 5,` `bits = 4`
1. Compute `1 << bits `(2 raised to the power of `bits`):
    - `1 << 4` results in 16 (`0b10000` in binary).
2. Add the negative value (`-5`) to the result:
    - `16 - 5 = 11`
    - `11` in binary is `0b1011`.

For an unsigned integer $uz$, a signed integer $sz$, and $n$ bits, the relationship is: $2^n - sz = uz$

## Are Negative Numbers Really Stored as Two's Complement?
As a programmer, it's crucial to understand how negative numbers are stored and processed in memory. This understanding helps in debugging and system-level programming.

### C Example: Observing Two's Complement
Below is a simple C program to observe how a signed integer is stored as two's complement.

**main.c**
```c
#include <stdio.h>

int main(void) {
  int a = -5;

  printf("%X\n", a);
  printf("%u\n", a);

  return 0;
}
```
1. Declaring a Negative Variable
```c
int a = -5;
```
In C, negative numbers are stored in memory using two's complement:
- Convert 5 to binary: `0000 0101`
- Invert all bits: `1111 1010` (one's complement)
- Add 1: `1111 1011` (two's complement)

2. Output Functions
```c
printf("%X\n", a);
```
Outputs the hexadecimal representation of `a`. For `-5` in 32-bit representation, the result is FFFFFFFB.

```c
printf("%u\n", a);
```
Outputs the unsigned integer representation of a. The result is $2^{32} - 5 = 4,294,967,291$.

**Output**
```
FFFFFFFB
4294967291
```

### Python Example: Observing Negative Values
```python-repl
>>> bin(-6)
'-0b110'
>>> ~5
-6
>>> bin(5)
'0b101'
>>> ~(-6)
5
```

Python handles integers as signed by default and uses two's complement representation. For example:
- `5` in binary: `0b101` (equivalent to `0b000...0101` in infinite bit-length)
- Bitwise NOT operation (`~5`): Flips all bits to produce `0b111...1010`.
- Result: The flipped bits represent `-6`.

> **Why is 5 treated as `0b000...0101` instead of 0b101?**  
> Python uses unlimited bit-length for integers, so negative numbers require additional bits for proper representation. For example:
> - -6 is represented as `111...1010` (infinite bits).

 The relationship: `~n = -(n + 1)`.

**Verifying the Two's Complement Representation of `-6`**
1. Flip all bits of `5` (`0b101`): `0b010`.
2. Add 1: `0b011` (equal to 3).

## Conclusion
In this post, we explored the reasons why two's complement is the standard representation for negative numbers in modern computer systems. Its simplicity in arithmetic operations, unique zero representation, and hardware efficiency make it superior to other methods.

In a future post, we will delve deeper into arithmetic operations with two's complement, including [overflow handling and the role of carry bits](/posts/carry-and-overflow/).

If you found this post helpful, consider subscribing to the blog or leaving a comment with your questions!