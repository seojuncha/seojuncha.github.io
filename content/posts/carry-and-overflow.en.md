+++
date = '2025-01-20'
title = 'What Is Overflow in Computer Arithmetic? Explained with Examples'
tags = ["carry", "overflow", "computer arithmetic"]
math = true
+++

Computers rely on finite-sized memory to represent numbers, which can lead to incorrect results if certain operations exceed the representable range. For example:

- In an 8-bit integer, calculating 127 + 1, `127+1` should result in `128`. However, for signed integers, this causes an overflow, resulting in −128 instead.  

Such results can lead to logical errors in programs. Moreover, overflow can be exploited to introduce security vulnerabilities. Well-known vulnerabilities like buffer overflow can allow attackers to execute code or gain control of the system.

From a hardware design perspective, overflow also plays a crucial role, making it essential for programmers to understand the limitations of arithmetic operations. This knowledge ensures safe and accurate results.

In this post, we’ll explore the **concept of overflow**, examine **the risks of overflow through programming examples**, and **compare overflow with carry** to clarify their differences.

# What is Overflow?
***Overflow*** occurs in computer arithmetic when a number exceeds the representable range of a specific data type. Simply put, it’s a problem that arises when a number is too large or too small to be represented.

As mentioned earlier, computers cannot use infinite memory. For example:
- An 8-bit signed integer can represent values from `−128` to `127`.
- An 8-bit unsigned integer can represent values from `0` to `255`.

If you add `1` to an 8-bit unsigned integer with a value of `255`, the result should be `256`. However, because this **exceeds the range of an [8-bit representation](/posts/binary-representation/)**, overflow occurs.

# Overflow C Example
Here’s an example in C to demonstrate overflow:

**overflow.c**
```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
  uint8_t ui = UINT8_MAX + 1;   // UINT8_MAX = 255
  int8_t i = INT8_MAX + 1;      // INT8_MAX = 127

  printf("%u\n", ui);
  printf("%d\n", i);

  return 0;
}
```
Output (GCC-11.4):
```
overflow.c: In function ‘main’:
overflow.c:5:16: warning: unsigned conversion from ‘int’ to ‘uint8_t’ {aka ‘unsigned char’} changes value from ‘256’ to ‘0’ [-Woverflow]
    5 |   uint8_t ui = UINT8_MAX + 1;
      |                ^~~~~~~~~
0
-128
```
The variables `ui` and `i` are assigned values through the following process.
- UINT8_MAX + 1
  - 255 + 1 = 256
  - 256 -> 0
```
+---+---+---+----+-----+-----+
| 0 | 1 | 2 | .. | 254 | 255 |
+---+---+---+----+-----+-----+
                          ↑
```
```
+---+---+---+----+-----+-----+
| 0 | 1 | 2 | .. | 254 | 255 |
+---+---+---+----+-----+-----+
  ↑
```

- INT8_MAX + 1
  - 127 + 1 = 128
  - 128 -> -128
```
+------+------+-----+-----+-----+
| -128 | -127 | ... | 126 | 127 |
+------+------+-----+-----+-----+
                             ↑
```
```
+------+------+-----+-----+-----+
| -128 | -127 | ... | 126 | 127 |
+------+------+-----+-----+-----+
    ↑
```

> **Why does the warning occur only in `uint8_t ui = UINT8_MAX + 1;`?**  
> `UINT8_MAX` and `INT8_MAX` are constant values that are treated as int in C. Therefore, `UINT8_MAX + 1` and `INT8_MAX + 1` are processed as int operations and stored in a 32-bit space  
>   
> Therefore, the compiler generates a warning about data loss when converting a 32-bit signed value to an 8-bit unsigned value.
>
> However, since the **[C standard defines signed integer overflow as undefined behavior](https://www.gnu.org/software/c-intro-and-ref/manual/html_node/Signed-Overflow.html)**, the compiler does not treat it as a warning.

## Integer Overflow in Python
Unlike other languages, Python does not impose a limit on the size of integers in arithmetic operations. This is because it uses ***[arbitrary-precision](https://peps.python.org/pep-0237/)***.
```python-repl
>>> 2**63 - 1    # Maximum value for a 64-bit signed integer.
9223372036854775807
>>> 2**63
9223372036854775808
>>> 2**66
73786976294838206464
```
Therefore, the concept of overflow does not exist in Python's integer arithmetic.

> However, in floating-point operations rather than integer operations, floating-point issues may occur.
> Additionally, if operations [exceed the memory](https://docs.python.org/3/library/exceptions.html#MemoryError) that the computer can allocate, memory shortage issues or performance degradation may occur.

# Carry vs Overflow
Carry and Overflow may seem similar, but they are distinct concepts.
- ***Carry***: Data allocation **exceeding memory space**
- ***Overflow***: **Exceeding the representable data range** within memory space

## For Unsigned Integer
In unsigned integer operations, carry and overflow occur simultaneously.

Example: In a 4-bit space, $7+9$
```
  0111   (7 in Decimal)
+ 1001   (9 in Decimal)
-------
 10000   (16 in Decimal, carry and overflow)
```
The representable range of a 4-bit unsigned integer is `0` to `15`. The result of the example, `16`, indicates:
- Carry : Carry Out occurs from the MSB.
- Overflow: Exceeds the maximum value of 15.

```
Carry (O)
[ 0111 ] + [ 1001 ] = [ 1 0000 ]
  4-bit      4-bit      5-bit

Overflow (O)
[ 0111 ] + [ 1001 ] = [ 1 0000 ]
[ 1 0000 ] = 16 (> 15)
```

## For Signed Integer
Unlike unsigned integers, signed integers do not experience carry and overflow simultaneously.

Example: In a 4-bit space, $5 + 6$
```
  0101   (5 in Decimal)
+ 0110   (6 in Decimal)
-------
  1011   (-5 in Decimal, no carry, but overflow occurs)
```
Signed integers use the most significant bit (MSB) as the sign bit, causing carry and overflow to occur separately.
- No Carry: No Carry Out occurs from the MSB.
- Overflow: The sum of two positive numbers exceeds the representable range of a signed integer.

```
Carry (X)
[ 0101 ] + [ 0110 ] = [ 1011 ]
  4-bit      4-bit      4-bit

Overflow (O)
[ 0101 ] + [ 0110 ] = [ 1011 ]
[ 1011 ] = -5
         => 11 (> 7)
```

> **`-5` falls within the 4-bit signed integer range of `-8` to `7`.**  
> That's correct. `-5` is within the representable range but is treated as overflow in hardware.  
> This is because the sum of two positive numbers, `11`, exceeds the representable range for positive numbers.

# Conclusion
In this post, we explored the concept of overflow, its causes, and how it is handled in programming languages. It is important to understand carry and overflow from the perspective of how integers are stored in memory. Depending on the type of integer being represented, carry and overflow may occur simultaneously or independently. 

**Next Post Topics**:
- Binary arithmetic operations from a hardware perspective
- [Carry and overflow handling in ARM processors](/posts/carry-and-overflow-in-hardware/)

If you have any questions or topics you'd like me to cover, please leave a comment!