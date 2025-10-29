---
layout: post
lang: ko
ref: "arm-memory-access"
title: "ARM 어셈블리 #7 — LDR/STR와 주소 모드(Addressing Modes)"
date: 2025-10-29 20:40:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["ldr", "str", "addressing-mode"]
published: false
---

어셈블리에서는 **메모리와 레지스터** 사이의 데이터 이동이 핵심입니다. 
ARM 아키텍처(ARMv4)는 **RISC load/store 구조**로 설계되어 있으며, 메모리 접근은 전용 명령인 `ldr` / `str`로만 수행합니다. 
이번 글에서는 **기본 LDR/STR 사용법**, **주소 모드(Addressing modes)**, **오프셋(Offset) 계산**을 실습 가능한 예제와 함께 정리합니다.

> 참고: x86은 `mov eax, [0x12345678]`처럼 **절대주소를 직접** 표기할 수 있지만,  
> ARM은 인코딩 제약과 설계 철학상 **베이스 + 오프셋(Base + Offset)** 구조만을 사용합니다.  
> (자세한 비교는 별도 포스팅에서 다룹니다.)

## ARM에서 메모리란?
컴파일된 프로그램은 **코드 섹션**(.text)에 저장되고, 변수나 상수는 **데이터/스택/힙** 같은 다른 메모리 영역에 배치됩니다.  
CPU는 **레지스터** 를 사용해 산술 및 논리 연산(`mov`, `add`, `sub`, …)을 수행하므로,  
**메모리 → 레지스터로 값을 읽는 동작(load)** 과 **레지스터 → 메모리에 값을 저장하는 동작(store)** 이 반드시 필요합니다.

## LDR / STR 명령 요약

| 명령어 | 동작 | 기본 형식 |
|:-:|:-:|:-:|
| `LDR` | 메모리의 데이터를 레지스터로 읽음 | `LDR Rt, [Rn {, #±imm12 | ±Rm {, shift #imm}}]` |
| `STR` | 레지스터의 데이터를 메모리에 저장 | `STR Rt, [Rn {, #±imm12 | ±Rm {, shift #imm}}]` |

- **Rd (destination register)**: 데이터를 읽거나 쓰는 대상 레지스터  
- **Rn (base register)**: 메모리 주소의 기준이 되는 베이스 레지스터  
- **Offset**: 즉시값(imm12), 레지스터(Rm), 시프트된 레지스터(Rm, shift #imm)
- **주소 모드(Addressing mode)**:
  - 일반(offset)
  - 선-인덱스(pre-index, `[...]!`)
  - 후-인덱스(post-index, `[...] , offset`)

> 주소 모드 핵심:  
> ARM의 메모리 피연산자에는 **절대주소 즉시값**을 직접 쓸 수 없습니다.  
> 항상 `[Rn, offset]` 형태로 사용해야 합니다.  
> 단, **리터럴 풀(Literal Pool)** 과 **의사명령(Pseudo-op)** 을 이용하면  
> 상수 또는 주소를 편리하게 다룰 수 있습니다 (`ldr r0, =0x1000` 등).

## 실습 준비
- **툴체인**: `arm-none-eabi-gcc`, `gdb-multiarch`
- **에뮬레이터**: `qemu-system-arm -machine versatilepb -nographic -S -s`
- **링크 옵션**: `-nostdlib -Ttext=0x10000`

> 참고: 실제 하드웨어나 QEMU 환경에 따라 로드 주소(`-Ttext`)는 달라질 수 있습니다.  
> 본 포스팅의 예제는 모두 학습용 단일 세그먼트 실행을 기준으로 작성되었습니다.

## STR: 레지스터 → 메모리

**문법**
```
STR Rt, [Rn {, #±imm12 | ±Rm {, shift #imm}}]
```

**예제: simple-str.s**
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00100000     @ Rn: 베이스 주소 (RAM 영역 가정)
    mov r1, #5              @ Rt = 5
    str r1, [r0]            @ [0x00100000] = 5
    b .
```

**컴파일 및 실행**
```bash
$ arm-none-eabi-gcc -nostdlib -Ttext=0x10000 simple-str.s -o simple-str.elf
$ qemu-system-arm -machine versatilepb -nographic -S -s -kernel simple-str.elf
```

**GDB 디버깅**
```bash
(gdb) target remote :1234
(gdb) i r r0 r1
(gdb) x/1w 0x00100000
```

## LDR: 메모리 → 레지스터

**예제: simple-ldr.s**
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00100000     @ 베이스 주소
    ldr r1, [r0]            @ Rt = [0x00100000]
    b .
```

## 주소 모드(Addressing Modes)

ARM은 메모리 주소를 다음과 같이 계산합니다:

```
주소 = 베이스(Rn) + 오프셋(offset)
```

- **오프셋 포맷 (3가지)**  
  - 즉시값(Immediate)
  - 레지스터(Register)
  - 시프트된 레지스터(Shifted Register)
- **오프셋 방식 (3가지)**  
  - 일반(offset)
  - 선-인덱스(pre-index)
  - 후-인덱스(post-index)

> 기억법: **대괄호 안의 연산이 먼저** 수행됩니다.  
> `[...]!`는 연산 후 Rn을 업데이트,  
> `[...] , offset`은 접근 후 업데이트를 의미합니다.

## A. 즉시 오프셋 (Immediate Offset)

### A-1) 일반(offset)
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00700000
    ldr r1, =0x00001000
    str r1, [r0]            @ [0x00700000] = 0x1000

    mov r1, r1, lsl #1
    str r1, [r0, #4]        @ [0x00700004] = 0x2000

    ldr r1, [r0, #-4]       @ Rt = [0x00700000]
    b .
```

### A-2) 선-인덱스(pre-index)
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00700000
    ldr r1, =0x1000
    str r1, [r0]            @ [base] = 0x1000

    mov r1, r1, lsl #1
    str r1, [r0, #4]!       @ [base+4] = 0x2000 ; R0 = base+4

    ldr r1, [r0, #-4]!      @ Rt = [base], R0 = base
    b .
```

### A-3) 후-인덱스(post-index)
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00700000
    ldr r1, =0x1000

    str r1, [r0], #4        @ [base] = 0x1000 ; R0 = base+4
    mov r1, r1, lsl #1
    str r1, [r0]            @ [base+4] = 0x2000

    ldr r1, [r0, #-4]!      @ Rt = [base], R0 = base
    b .
```

## B. 레지스터 오프셋 (Register Offset)

### B-1) 일반(offset)
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00700000
    ldr r1, =0x1000
    mov r2, #4

    str r1, [r0]            @ [base] = 0x1000
    mov r1, r1, lsl #1
    str r1, [r0, r2]        @ [base+4] = 0x2000

    ldr r1, [r0, #-4]       @ Rt = [base]
    b .
```

### B-2) 선-인덱스(pre-index)
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00700000
    ldr r1, =0x1000
    mov r2, #4

    str r1, [r0]
    mov r1, r1, lsl #1
    str r1, [r0, r2]!       @ R0 = base+4

    ldr r1, [r0, -r2]!      @ R0 = base
    b .
```

### B-3) 후-인덱스(post-index)
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00700000
    ldr r1, =0x1000
    mov r2, #4

    str r1, [r0], r2        @ [base] = 0x1000 ; R0 = base+4
    mov r1, r1, lsl #1
    str r1, [r0]            @ [base+4] = 0x2000

    ldr r1, [r0, -r2]!      @ Rt = [base]
    b .
```

## C. 시프트된 레지스터 오프셋 (Shifted Register Offset)

> 시프트 양은 항상 **즉시값(`#imm`)** 으로 지정해야 합니다.  
> 예: `lsl #2`(O) , `lsl r3` (X)

### C-1) 일반(offset)
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00700000
    ldr r1, =0x1000
    mov r2, #1

    str r1, [r0]
    mov r1, r1, lsl #1
    str r1, [r0, r2, lsl #2]    @ [base + (1<<2)] = [base+4] = 0x2000

    ldr r1, [r0]
    b .
```

### C-2) 선-인덱스(pre-index)
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00700000
    ldr r1, =0x1000
    mov r2, #1

    str r1, [r0]
    mov r1, r1, lsl #1
    str r1, [r0, r2, lsl #2]!   @ R0 = base+4

    ldr r1, [r0, -r2, lsl #2]!  @ R0 = base
    b .
```

### C-3) 후-인덱스(post-index)
```armasm
    .text
    .global _start
_start:
    ldr r0, =0x00700000
    ldr r1, =0x1000
    mov r2, #1

    str r1, [r0], r2, lsl #2
    mov r1, r1, lsl #1
    str r1, [r0]
    ldr r1, [r0, -r2, lsl #2]!
    b .
```

## 주소 모드 요약표

| 베이스 | 오프셋 포맷 | 부호 | 유형 | 예시 |
|---|---|---|---|---|
| `Rn` | 즉시값 | `+` | 일반 | `ldr r0, [r1, #4]` |
|  |  |  | 선-인덱스 | `ldr r0, [r1, #4]!` |
|  |  |  | 후-인덱스 | `ldr r0, [r1], #4` |
| `Rn` | 즉시값 | `-` | 일반 | `ldr r0, [r1, #-8]` |
|  |  |  | 선-인덱스 | `ldr r0, [r1, #-8]!` |
|  |  |  | 후-인덱스 | `ldr r0, [r1], #-8` |
| `Rn` | 레지스터 `Rm` | `+` | 일반 | `ldr r0, [r1, r2]` |
|  |  |  | 선-인덱스 | `ldr r0, [r1, r2]!` |
|  |  |  | 후-인덱스 | `ldr r0, [r1], r2` |
| `Rn` | 레지스터 `Rm` | `-` | 일반 | `ldr r0, [r1, -r2]` |
|  |  |  | 선-인덱스 | `ldr r0, [r1, -r2]!` |
|  |  |  | 후-인덱스 | `ldr r0, [r1], -r2` |
| `Rn` | 시프트 레지스터 `Rm, shift #imm` | `+` | 일반 | `ldr r0, [r1, r2, lsl #2]` |
|  |  |  | 선-인덱스 | `ldr r0, [r1, r2, lsl #2]!` |
|  |  |  | 후-인덱스 | `ldr r0, [r1], r2, lsl #2` |
| `Rn` | 시프트 레지스터 `Rm, shift #imm` | `-` | 일반 | `ldr r0, [r1, -r2, lsl #2]` |
|  |  |  | 선-인덱스 | `ldr r0, [r1, -r2, lsl #2]!` |
|  |  |  | 후-인덱스 | `ldr r0, [r1], -r2, lsl #2` |


## 실전 팁

### ️MOV 즉시값의 제약
ARM 상태에서 `mov` 즉시값은 8비트 값 + 짝수 비트 회전으로 표현됩니다.  
임의의 32비트 상수는 단일 `mov`로 불가능할 수 있습니다.  
이럴 땐 **`ldr Rt, =0x7000000`** 형태의 **의사명령(Pseudo-op)** 을 사용하세요.

### 리터럴 풀(Literal Pool)
`ldr Rt, =0xDEADBEEF` 같은 코드는  
어셈블러가 **리터럴 풀(literal pool)** 을 자동으로 만들고  
`ldr Rt, [pc, #offset]` 형태로 치환합니다.

### PC-relative Load
직접 `[pc, #offset]`을 사용해 리터럴 풀의 데이터를 읽을 수도 있습니다.

---

## GDB 로그 관리 팁
전체 단계를 모두 기록하기보다, 상태 변화가 있는 지점만 캡처하는 것이 좋습니다.

| 시점 | 명령 | 확인 명령 |
|---|---|---|
| 저장 직후 | `str` | `x/2w base` |
| 인덱스 업데이트 후 | `info reg r0` |
| 로드 직후 | `info reg Rt` |


## 마무리

오늘은 ARM 아키텍처의 **LDR/STR 명령어**와 **9가지 주소 모드**를 예제와 함께 살펴봤습니다.  
이 개념은 함수 호출 시 스택 관리, 지역 변수 접근, 구조체 멤버 접근 등  
모든 고수준 언어의 동작을 구성하는 기본입니다.

> 다음 포스팅에서는  
> **`ldrb` / `strb` / `ldrh` / `strh`** 와 같은 바이트·하프워드 명령어,  
> 그리고 **`ldm` / `stm`** 다중 전송 명령어를 통해  
> **스택 프레임 구성**과 연결해 살펴보겠습니다.
