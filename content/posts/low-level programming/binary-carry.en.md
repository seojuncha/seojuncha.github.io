+++
date = '2025-01-14'
title = 'Understanding Carry in Arithmetic: Binary Operations Simplified'
math = true
tags = ["carry", "binary operation", "arithmetic operation", "binary arithmetic"]
+++

Carry is a fundamental concept in computer arithmetic. It ensures the accuracy of operations and plays a critical role in hardware and software design, debugging, and advanced algorithm optimization. Especially in systems with limited bit space, understanding how Carry works is essential. In this article, we will explore **the definition and concept of Carry**, **how it works in binary and decimal arithmetic**, and **how programming languages handle Carry**.

Example: When unsigned int exceeds its maximum range:
```c
unsigned int a = 4294967295; // maximum value (32-bit unsigned integer)
unsigned int b = 1;
unsigned int c = a + b; // Overflow: c has 0
```

## What is Carry?
Let’s start with the concept of carrying in decimal addition:  
$$
5 + 4 = 9
$$

No carry is generated when 5 + 4 = 9, as the result fits within a single digit. Carry occurs when the sum exceeds the value that a single digit can represent. For example:
$$
9 + 5 = 14  (Carry)
$$
Here, `9 + 5` results in `14`, where the ones place overflows into the tens place.

The same principle applies to binary addition. In binary, the maximum value a single digit can represent is 1. When adding 1 + 1, carry is generated:
$$
\begin{aligned}
&0 + 0 = 0  (No Carry) \newline
&1 + 0 = 1  (No Carry) \newline
&1 + 1 = 10 (Carry)
\end{aligned}
$$

```
1 + 1 = 10 (Carry generated)
=> In binary addition, carry is handled as a 1 passed to the next higher bit.
```

Carry is important because it **occurs when the sum exceeds the capacity of a fixed-width space**. For example, in a 4-bit memory, the maximum binary value `1111` overflows to `10000`.

## Carry in and Carry out
***Carry in*** is the carry value passed from a lower bit operation, and ***Carry out*** is the carry value resulting from the current bit operation. This process is typically handled by a hardware "Full Adder" circuit.

*Carry in* ensures continuity and accuracy in multi-bit operations.

Example: Adding `13` and `7` in a `4-bit` space
```
carry in   : 1   1   1   0
13 (1101)  : 1   1   0   1
7  (0111)  : 0   1   1   1
---------------------------
sum        : 1   0   0   0
carry out  : 1   1   1   1

```
Each bit’s addition (from LSB to MSB) includes carry in:
```
1 + 1 + (0 : Carry In) = 0 (Carry Out)
1 + 0 + (1 : Carry In) = 0 (Carry Out)
1 + 1 + (1 : Carry In) = 1 (Carry Out)
0 + 1 + (1 : Carry In) = 0 (Carry Out)
```

Carry occurs when the sum exceeds the capacity of a bit. The result here is `13 + 7 = 20 (0b10100)`.

## Carry in Arithmetic Operations
Let’s analyze carry in basic arithmetic operations like addition and subtraction.

### Binary Addition
Addition in binary operates similarly regardless of whether the numbers are signed or unsigned.

Example 1: Unsigned `3 + 9` in 4-bit space
```
  0011
+ 1001
------
  1100  = 12 in Decimal
```

Example 2: Signed `3 + 2` in 4-bit space
```
  0011
+ 0010
------
  0101  = 5 in Decimal
```
> For signed integers in 4-bit space, the maximum positive value is 7. Results exceeding this will be discussed in the next post.

### Binary Subtraction
In computers, subtraction is performed by adding the two's complement of the subtrahend.
$$
7 - 3 = 7 + (-3)
$$

For example, in 4-bit space:  
Represent `-3` as `two's complement of 3`:
- 3 = 0011
- -3 = 1101

Example 1: $7 - 3$
- 7 = 0111
- 3 = 0011
  - -3 = 1101

```
  0111   (7 in binary)
+ 1101   (-3 in binary)
-------
 10100  = 4 (ignore carry)
```
Example 2: $3 - 7$
- 3 = 0011
- 7 = 0111
  - -7 = 1001
```
  0011   (3 in binary)
+ 1001   (-7 in binary)
-------
  1100  = (interpret based on signed/unsigned)
```
For unsigned integers:
- 1100 = 12

For signed integers:
- 1100 = -4

## Carry in Shift Operations
Carry is also relevant in shift operations, which move bits left or right. The behavior depends on the direction of the shift:
- **Left Shift**: Carry occurs if the MSB is exceeded.
- **Right Shift**: Carry occurs if the LSB is exceeded.

Example of Left Shift:  
Left shifts are used to double the value of a binary number, while right shifts are used to halve it. However, without considering the carry, data loss may occur.
```python
value = 0b0111  # 7 in binary
left_shift = value << 2  # Shift left by 2 bits
carry = left_shift >> 4  # Extract carry from MSB
print(bin(carry))  # Outputs: 0b1
```

Left Shift Example (4-bit):
```
0111 << 1 = (0)1110 (No Carry)
0111 << 2 = (1)1100 (Carry)
```

Right Shift Example (4-bit):
```
0110 >> 1 = 0011(0)  (No Carry)
0110 >> 2 = 0001(1)  (Carry)
```

### Utilizing Carry in Shift Operations
The carry in shift operations is often used in tasks such as circular data shifts (Circular Shift) or [CRC (Cyclic Redundancy Check)](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) calculations.
```python
def circular_left_shift(value, bits, size):
    return ((value << bits) | (value >> (size - bits))) & ((1 << size) - 1)
```

## Programming Examples

### Example 1: Detecting Carry in Python
Here’s a Python function to detect carry:
```python
def has_carry(value, bits):
  """
  Detects if carry occurs beyond a given bit width.
  """
  return bool(value & (1 << bits))
```
Example:
```python
a, b = 13, 5

print(f"{a:05b}")
print(f"{b:05b}")
print(f"{a + b:05b}")
print(has_carry(a + b, 4))
print(has_carry(a + b, 5))
```
Output
```
01101
00101
10010
True
False
```

### Example 2: Unsigned Integer Carry in C
This C example demonstrates how carry affects unsigned integers:

```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
  uint8_t ui8 = UINT8_MAX; // 8-bit max value = 255
  uint8_t ui8_plus_one = UINT8_MAX + 1;  // Carry wraps to 0
  uint8_t ui8_plus_two = UINT8_MAX + 2;  // Carry wraps to 1

  uint16_t ui16 = UINT8_MAX;  // Expand to 16-bit
  uint16_t ui16_plus_one = UINT8_MAX + 1;  // 256
  uint16_t ui16_plus_two = UINT8_MAX + 2;  // 257

  printf("8-bit Memory\n");
  printf("  %u [0x%02X]\n", ui8, ui8);  // 255
  printf("  %u [0x%02X]\n", ui8_plus_one, ui8_plus_one);  // 0
  printf("  %u [0x%02X]\n", ui8_plus_two, ui8_plus_two);  // 1

  printf("16-bit Memory\n");
  printf("  %u [0x%04X]\n", ui16, ui16);  // 255
  printf("  %u [0x%04X]\n", ui16_plus_one, ui16_plus_one);  // 256
  printf("  %u [0x%04X]\n", ui16_plus_two, ui16_plus_two);  // 257

  return 0;
}
```

Output:
```
8-bit Memory
  255 [0xFF]
  0 [0x00]
  1 [0x01]
16-bit Memory
  255 [0x00FF]
  256 [0x0100]
  257 [0x0101]
```
In this case, the compiler makes the warning message.

Compiler Warning
```
carry.c: In function ‘main’:
carry.c:6:26: warning: unsigned conversion from ‘int’ to ‘uint8_t’ {aka ‘unsigned char’} changes value from ‘256’ to ‘0’ [-Woverflow]
    6 |   uint8_t ui8_plus_one = UINT8_MAX + 1;
      |                          ^~~~~~~~~
carry.c:7:26: warning: unsigned conversion from ‘int’ to ‘uint8_t’ {aka ‘unsigned char’} changes value from ‘257’ to ‘1’ [-Woverflow]
    7 |   uint8_t ui8_plus_two = UINT8_MAX + 2;
      |                          ^~~~~~~~~
```

## Conclusion
Carry is essential for ensuring arithmetic accuracy in both hardware and software. It also aids in debugging and optimizing algorithms. In the next post, we will explore the differences between **[Carry and Overflow](/posts/carry-and-overflow/)** and how to handle complex arithmetic errors.

**Feel free to leave a comment for more topics!**

## Related Articles
- [Binary representation for Signed and Unsigned integers](/posts/binary-representation/)
