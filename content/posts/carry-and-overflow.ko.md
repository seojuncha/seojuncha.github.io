+++
date = '2025-01-15'
title = "예제로 알아보는 컴퓨터 연산의 Overflow"
tags = ["carry", "overflow", "computer arithmetic"]
math = true
+++

컴퓨터는 유한한 크기의 메모리를 사용하여 숫자를 표현합니다. 특정 연산이 표현범위를 초과하면 잘못된 결과를 초래할 수 있습니다. 예를들어: 
- 8비트 정수에서 127 + 1을 계산하면 실제 결과는 128이지만, Signed Integer의 경우 Overflow로 인해 결과는 -128이 됩니다.

이러한 결과는 프로그램의 논리 오류로 이어질 수 있습니다.
또한, Overflow는 악용될 경우 보안 취약점으로 이어질 수 있습니다. 널리 알려져있는 버퍼 오버플로우(Buffer Overflow)와 같은 취약점은 공격자가 코드를 실행하거나 시스템을 제어할 수 있는 가능성을 제공합니다. 

이 외에도 하드웨어 설계측면에서도 중요한 역할을 하며 프로그래머는 이런 산술 연산의 한계를 이해해야만 안전하고 정확한 연산 결과를 보장할 수 있습니다. 이번 포스팅에서는 **Overflow**의 개념과 **프로그래밍 예제를 통한 Overflow의 위험성**을 살펴보고 **Overflow와 Carry를 비교**하여 서로의 차이점을 정확히 이해하는 것을 목표로 합니다.

# What is Overflow?
**Overflow**란 컴퓨터 연산에서 특정 데이터 타입이 표현할 수 있는 숫자 범위를 초과했을 때 발생하는 현상입니다. 간단히 말해, **숫자가 너무 커지거나 작아져서 표현할 수 없을 때 발생**하는 문제입니다.

서론에서 설명한 듯이, **컴퓨터는 무한한 메모리를 사용할 수 없습니다**. 예를 들어:
- 8비트 부호 있는 정수(Signed Integer)는 `-128`부터 `127`까지의 값을 표현할 수 있습니다.
- 8비트 부호 없는 정수(Unsigned Integer)는 `0`부터 `255`까지의 값을 표현할 수 있습니다.

8비트 부호 없는 정수 값 `255`에서 `1`을 더하면 `256`이 되어야 하지만 한정된 메모리 공간(8비트)에서의 표현범위를 벗어나므로 Overflow가 발생합니다.

# Overflow C Example
Overflow가 발생하는 C언어 예제를 살펴보겠습니다.

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
아래의 과정으로 변수 `ui`와 `i`에 값을 할당합니다.
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

> **왜 `uint8_t ui = UINT8_MAX + 1;`에서만 경고가 발생할까?**  
> 
> `UINT8_MAX`와 `INT8_MAX`는 상수(Constant)값으로 C언어에서는 int형으로 처리되기 때문에 `UINT8_MAX + 1`과 `INT8_MAX + 1`은 int형 연산으로 처리되어 32비트 공간에 저장됩니다.  
>   
> 따라서 컴파일러는 32비트 signed값을 8비트 unsigned값으로 변환시의 데이터 손실에 대한 경고를 표시합니다.
>
> 단, C언어 표준에서는 signed정수의 overflow를 정의 되지 않은 동작(Undefined Behavior)으로 규정하기 때문에 컴파일러가 경고로 처리하지 않습니다.

## Integer Overflow in Python
파이썬은 다른 언어와 달리 정수형 연산에서 데이터 크기에 제한이 없습니다. 이것은 무제한 정밀도(arbitrary-precision)를 사용하기 때문입니다.
```python-repl
>>> 2**63 - 1    # 64비트 정수형 최대값
9223372036854775807
>>> 2**63
9223372036854775808
>>> 2**66
73786976294838206464
```
따라서, 파이썬의 정수형 연산에서는 Overflow의 개념이 없습니다.

> 하지만, 정수형 연산이 아닌 실수형 연산에서는 부동소수점(Floating-Point) 이슈가 발생할 수 있습니다.  
> 또한, 컴퓨터가 할당할 수 있는 메모리를 넘어서는 경우에는 메모리 부족 이슈가 생기거나 성능 저하가 생길 수 있습니다.

# Carry vs Overflow
Carry와 Overflow는 유사한 것으로 보이지만 다른 개념입니다.
- Carry: **메모리 공간을 초과**한 데이터 할당
- Overflow: 메모리 공간에서 표현할 수 있는 **데이터 범위를 초과**

## For Unsigned Integer
Unsigned Integer 연산에서는 **Carry와 Overflow가 동일하게 발생합니다**.

예: 4비트 공간에서 $7+9$
```
  0111   (7 in Decimal)
+ 1001   (9 in Decimal)
-------
 10000   (16 in Decimal, carry and overflow)
```
**4비트** Unsigned Integer의 표현범위는 `0` 부터 `15` 까지 입니다. 예시의 결과인 `16` 은
- Carry 발생: MSB에서 Carry Out 발생
- Overflow 발생: 최대값인 15를 초과

```
Carry (O)
[ 0111 ] + [ 1001 ] = [ 1 0000 ]
  4-bit      4-bit      5-bit

Overflow (O)
[ 0111 ] + [ 1001 ] = [ 1 0000 ]
[ 1 0000 ] = 16 (> 15)
```

## For Signed Integer
Signed Integer는 Unsigned Integer와 달리, **Carry와 Overflow가 동일하게 발생하지 않습니다**.

예: 4비트 공간에서 $5 + 6$
```
  0101   (5 in Decimal)
+ 0110   (6 in Decimal)
-------
  1011   (-5 in Decimal, no carry, but overflow occurs)
```
Signed Integer는 최상위 비트(MSB)를 부호 비트로 사용하기 때문에 Carry와 Overflow가 각각 별도로 발생합니다.  
- Carry 발생 안함: MSB에서 Carry Out이 발생하지 않음
- Overflow 발생: 두 양수의 합이 signed integer의 표현범위를 초과

```
Carry (X)
[ 0101 ] + [ 0110 ] = [ 1011 ]
  4-bit      4-bit      4-bit

Overflow (O)
[ 0101 ] + [ 0110 ] = [ 1011 ]
[ 1011 ] = -5
         => 11 (> 7)
```

> **`-5` 는 4비트 Signed Integer 범위인 `-8~7` 에 속한다?**  
> 맞습니다. `-5` 는 표현 가능한 범위에는 포함되지만 하드웨어에서는 Overflow로 처리합니다.  
> 두 양수의 합인 `11`이 양수의 표현범위에 벗어나기 때문입니다.

# Conclusion
이번 포스트에서는 Overflow의 개념과 발생하는 이유, 그리고 프로그래밍 언어에서 어떻게 처리되는지 살펴봤습니다. Carry와 Overflow를 정수가 메모리에 저장되는 관점에서 이해하는 것이 중요하고 표현하는 정수의 유형에 따라서 Carry와 Overflow는 동시에 발생할 수도 있고 그렇지 않을 수도 있다는 점을 이해하길 바랍니다. 

다음 주제
- 하드웨어 측면에서의 이진수 산술 연산
- ARM 프로세서의 Carry와 Overflow처리

궁금하신 점이나 추가로 다루길 바라는 주제는 댓글로 남겨주세요!