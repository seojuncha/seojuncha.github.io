---
layout: post
lang: ko 
ref: arm-logical-shift
title: "[ARM32] 논리 시프트 이해하기(LSL, LSR)"
date: 2025-09-10 00:13:00 +0900
categories: [arm,assembly,tutorial]
tags: [Assembly, Embedded, low-level programming, ARM, QEMU, GDB]
---
컴퓨터는 비트 단위로 데이터를 조작하는 연산을 자주 수행합니다. 그 중에서도 **시프트(Shift) 연산**은 매우 중요하죠. 이번 글에서는 ARM 어셈블리에서 사용하는 논리 시프트(Logical Shift) 를 중심으로 설명합니다.

추상적인 개념 설명보다는 실습 예제를 통해 쉽게 접근하고, 시프트 연산이 실제로 어떤 효과를 가지는지, 그리고 그 과정에서 캐리 플래그(Carry Flag) 가 어떻게 작동하는지도 함께 살펴보겠습니다.

> 실습 코드는 [GitHub 저장소](https://github.com/seojuncha/arm-assembly-tutorial/tree/main/02_moving-data-around)에서 확인하실 수 있습니다.

## 시프트 피연산자란?
ARM 어셈블리에서는 `mov`, `add`, `sub` 같은 명령어를 사용할 때, 피연산자(operand) 중 하나에 시프트 연산을 미리 적용한 값을 넣을 수 있습니다. 이런 피연산자를 시프트 피연산자(shift operand) 라고 부릅니다.

이 기능 덕분에 별도의 시프트 명령 없이도 다양한 연산을 더 간결하게 표현할 수 있습니다.

예:
```
mov r1, r0, lsl #2    @ R0를 왼쪽으로 2비트 시프트 → R1에 저장
```
이렇게 명령어 한 줄로 시프트 연산과 대입을 동시에 처리할 수 있습니다.

## 논리 시프트란?
**논리 시프트(Logical Shift)** 는 단순히 비트를 왼쪽이나 오른쪽으로 이동시키는 연산입니다. 이때 빈자리는 항상 0으로 채워집니다.

예를 들어 4비트 값 `0110`을 기준으로:
```
왼쪽 시프트: 0110 << 1 = 1100
오른쪽 시프트: 0110 >> 1 = 0011
```
ARM에서는 다음 두 가지 논리 시프트 연산자를 제공합니다:

| 명령어 | 설명 |
|:---|:---|
|`lsl`| 왼쪽 논리 시프트 |
|`lsr`| 오른쪽 논리 시프트 |

시프트 양은 즉시값(#3)이나 레지스터 값(R2)으로 지정할 수 있습니다.
- `lsl #n`, `lsr #n` : 즉시값 (Immediate)
- `lsl rN`, `lsr rN` : 레지스터 값 (Register)

### 예제 코드: 논리 시프트 실습
**logical-shift.s**
```
  .text
  .global _start
_start:
  mov r0, #3
  mov r1, r0, lsl r0
  mov r2, r1, lsr #1
  b .
```
- `mov r0, #3` : r0에 숫자 3을 저장합니다.
- `mov r1, r0, lsl r0` : r0를 r0(=3)만큼 왼쪽 시프트한 값을 r1에 저장합니다.
- `mov r2, r1, lsr #1` : r1을 오른쪽으로 한 칸 밀어 r2에 저장합니다.

예시 흐름: 
```
r0 = 0x3 = 0b0...0011

# r1 = r0 << 3
r1 = 0x18 = 0b0...011000

# r1 = 0x18 >> 1
r1 = 0xC = 0b0...001100
```

### GDB 디버깅으로 확인하기
QEMU와 GDB를 통해 동작을 직접 확인해봅니다.

#### 1. QEMU 실행
```bash
$ qemu-system-arm \
    -machine versatilepb \
    -nographic \
    -S \
    -s \
    -kernel logical-shift.elf
```
#### 2. GDB 연결
```bash
$ gdb-multiarch logical-shift.elf
(gdb) target remote localhost:1234
```
#### 3. 메모리 확인
```bash
(gdb) x/10i 0x10000
```
#### 4. 레지스터 확인
```bash
(gdb) info registers
(gdb) i r r0 r1 r2
```
#### 5. 명령어 한 줄 실행
```bash
(gdb) stepi
```
실행 후 다시 `info registers`를 확인하면 레지스터 값이 변한 걸 확인할 수 있습니다.


## 시프트 연산에서 캐리 플래그는 언제 설정될까?
시프트 연산을 하면 비트들이 밀려나면서 비트가 하나 버려지게 됩니다. 이때 밀려나간 비트는 CPSR 레지스터의 캐리 플래그(C) 에 저장될 수 있습니다.

다만, 일반 `mov` 명령어는 캐리 플래그를 업데이트하지 않습니다. 대신 `movs` 명령어를 사용하면 캐리 플래그도 업데이트됩니다.

### 예제 코드: 캐리 플래그 확인

**logical-right-shift-carry.s**
```
  .text
  .global _start
_start:
  mov r0, 0x1
  mov r1, r0, lsr #1
  movs r2, r0, lsr #1
  b .
```
-	`mov`는 캐리 플래그를 갱신하지 않음
-	`movs`는 결과뿐 아니라 CPSR의 캐리 플래그도 업데이트

GDB에서 캐리 플래그를 확인하려면:
```bash
(gdb) target remote localhost:1234
(gdb) info registers cpsr
(gdb) p ($cpsr >> 29) & 1
```
CPSR의 하위 비트를 보면 C, Z, N 등의 상태를 확인할 수 있습니다.

## 논리 시프트의 한계와 활용
논리 시프트는 항상 0으로 채워지기 때문에 부호 없는 정수 처리에 적합합니다. 하지만 부호 있는 정수(예: 음수)는 부호 비트가 손상될 수 있어 주의가 필요합니다.

예:
```
mov r0, #-4
mov r1, r0, lsr #1
```
`mov r1, r0, lsr #1` 에서 오른쪽 시프트 결과로 부호 비트가 손실됩니다.  
이런 이유로 부호 있는 정수에는 `ASR`(Arithmetic Shift Right)를 사용합니다.

그럼에도 불구하고 논리 시프트는 다음과 같은 상황에서 매우 유용합니다:
- 배열 인덱스 계산
- 메모리 주소 계산
- 비트 마스킹

## 마무리
이번 글에서는 ARM 어셈블리에서 사용되는 논리 시프트 연산에 대해 알아보았습니다.

- `lsl`, `lsr`을 통해 비트를 이동
- 시프트 피연산자를 사용해 명령어 안에서 시프트 적용
- `movs`를 통해 캐리 플래그 확인
- 부호 있는 정수 처리 시에는 주의 필요

다음 글에서는 `asr`, `ror` 등 다른 시프트 유형에 대해 더 살펴보겠습니다.
