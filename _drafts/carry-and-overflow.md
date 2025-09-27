---
layout: post
lang: ko
ref: "carry-and-overflow"
date: 2025-09-27 11:40:00 +0900
title: "예제로 알아보는 캐리와 오버플로우"
tags: ["carry", "overflow", "computer arithmetic"]
published: false
---

컴퓨터는 유한한 크기의 **n비트 정수 표현**을 사용합니다. 특정 연산이 이 표현 범위를 벗어나면 결과가 왜곡됩니다. 예를 들어:
- 8비트 정수에서 `127 + 1`은 실제 결과가 128이어야 하지만, Signed Integer의 경우 오버플로우(Overflow)가 발생하여 `-128`이 됩니다.

이러한 문제는 단순한 논리 오류뿐만 아니라 보안 취약점으로 악용될 수 있습니다. 예컨대, 널리 알려진 **버퍼 오버플로우(Buffer Overflow)** 공격은 시스템 제어 권한을 공격자에게 넘길 수도 있습니다.  

하드웨어 설계나 저수준 프로그래밍에서 이 개념을 정확히 이해하는 것은 안전하고 올바른 연산을 보장하기 위해 필수적입니다. 이번 글에서는 **오버플로우의 개념**, **C/Python 예제**, 그리고 **Carry와 Overflow의 차이점**을 살펴봅니다.

> 캐리(Carry)에 관한 설명은 [이 포스팅]()을 참고해주세요.

## 오버플로우(Overflow)란?
**Overflow**란 특정 데이터 타입이 표현할 수 있는 숫자 범위를 벗어난 경우 발생하는 현상입니다.  
- 8비트 **부호 있는 정수(Signed)**: `-128` ~ `127`  
- 8비트 **부호 없는 정수(Unsigned)**: `0` ~ `255`

예:  
- Unsigned 8비트에서 `255 + 1 = 256` → 표현 불가 → 0으로 wraparound  
- Signed 8비트에서 `127 + 1 = 128` → 표현 불가 → −128로 wraparound

## C언어 오버플로우 발생 예제
Overflow가 발생하는 C언어 예제를 살펴보겠습니다.

**overflow.c**
```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
  uint8_t ui = UINT8_MAX + 1;   // 255 + 1 = 256 → 0
  int8_t i = INT8_MAX + 1;      // 127 + 1 = 128 

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

### 왜 `uint8_t` 에서만 경고가 발생할까?
- `UINT8_MAX`와 `INT8_MAX`는 int 상수로 처리됩니다. 따라서 `UINT8_MAX + 1`, `INT8_MAX + 1`은 32비트 int 연산으로 수행됩니다.
- `uint8_t ui = 256`은 int(256) → uint8_t 변환 시 데이터 손실이 명확하므로 컴파일러가 경고합니다.
- `int8_t si = 128`의 경우는 표현 불가 값 변환이라 C 표준상 *구현정의 동작*입니다. 보통 2의 보수 표현에서는 `−128`이 됩니다.
(진짜 Undefined Behavior는 `int x = INT_MAX; x = x + 1;`처럼 int 자체에서 연산 중 오버플로우가 발생하는 경우입니다.)

## 캐리(Carry)와 오버플로우(Overflow) 비교
Carry와 Overflow는 비슷해 보이지만 다른 개념입니다.
- 캐리(C): 비트 덧셈에서 최상위 비트(MSB)에서 자리올림(carry-out)이 발생했는지 (또는 뺄셈에서 자리내림/borrow 여부)를 나타냅니다.
- 오버플로우(V/OF): Signed 정수 해석에서 결과가 표현 범위를 벗어났는지 여부.

### 부호 없는 정수
Unsigned에서는 범위를 넘는 순간 Carry와 Overflow가 **동시에 발생**합니다.

예: 4비에서 $7 + 9$
```
  0111   (7)
+ 1001   (9)
-------
 10000   (16)
```
- 캐리 발생: 4비트 초과비트가 1
- 오버플로우 발생: 16 > 15

> 하지만, CPU의 Overflow 플래그는 Signed연산 검출용입니다. 따라서, Unsigned에서 Overflow라고 부를 땐, Carry로 표현합니다.

### 부호 있는 정수
Signed에서는 Carry와 Overflow가 다르게 동작합니다.

예: 4비트에서 $5 + 6$
```
  0101   (5)
+ 0110   (6)
-------
  1011   (-5)
```
- 캐리 발생: 없음(MSB에서 자리올림 없음)
- 오버플로우 발생: 두 양수의 합인데 결과 부호가 바뀌었음

즉, `Carry=0`, `Overflow=1` 입니다. 

### 요약표

| 연산 | 조건 | 캐리(C) | 오버플로우(V) |
|-|-|-|-|
| 덧셈 | | | | 
| 덧셈 ||| |
| 뺄셈 ||| |
| 뺄셈 ||| |


## Conclusion
이번 글에서는 오버플로우의 정의, C/Python에서의 동작, 그리고 Carry와 Overflow의 차이를 살펴봤습니다.

- Unsigned에서는 Carry=Overflow
- Signed에서는 Carry와 Overflow가 다르게 발생
- C에서는 signed overflow는 Undefined Behavior, 표현 불가 값 변환은 구현정의
