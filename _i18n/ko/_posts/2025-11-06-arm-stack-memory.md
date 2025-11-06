---
layout: post
lang: ko
ref: "arm-stack-memory"
title: "ARM 어셈블리 #10 - 스택 메모리(Stack Memory) 이해하기"
date: 2025-11-06 17:30:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["arm stack memory", "ldm", "stm"]
published: false
---

오늘은 CPU가 임시 데이터를 다루기 위해 활용하는 **스택 메모리(Stack Memory)** 에 대해 알아보겠습니다. 
앞선 포스팅들에서 살펴본 메모리 접근 명령의 응용법을 기반으로, 다음 포스팅에서 다룰 블록 단위 메모리 접근 명령(`LDM`, `STM`)을 이해하기 위한 기초 개념을 다집니다.

**이전 포스팅들:**
- [LDR과 STR을 활용한 메모리 접근]({% post_url 2025-10-31-arm-ldr-str-basic %})  
- [메모리 오프셋 주소 적용]({% post_url 2025-11-02-arm-offset-addressing %})  
- [Pre-index와 Post-index]({% post_url 2025-11-03-arm-assembly-pre-post-index %})  

## 임시데이터란?  
프로그램 실행 중 일시적으로 생성되고 사라지는 데이터를 말합니다. 대표적으로 **함수의 지역 변수(local variable)**, **매개변수(parameter)**, 그리고 **함수 호출 시의 반환 주소(return address)** 등이 이에 해당합니다. 이러한 데이터들은 프로그램 실행 내내 유지될 필요가 없기 때문에, 일정한 메모리 영역을 계속 차지하지 않아도 됩니다.  

## ARM 메모리 영역  

임시 데이터도 결국 CPU가 연산하기 전에 메모리에 저장되어야 합니다.  

일반적으로 메모리는 역할에 따라 여러 영역으로 구분됩니다.  

- **코드 영역 (Code Segment)**: 실행 명령어가 저장되는 영역  
- **데이터 영역 (Data Segment)**: 전역변수와 정적변수가 저장되는 영역  
- **힙 영역 (Heap Segment)**: 동적으로 생성되는 데이터가 저장되는 영역  
- **스택 영역 (Stack Segment)**: 함수 호출과 지역 변수가 저장되는 영역  

이 중 **코드 영역**과 **스택 영역**은 필수적으로 정의되어야 하며, 이번 글에서는 스택 영역에 집중해 보겠습니다.  

<figure style="text-align: center;">
  <img src="/assets/img/aapcs32-memory-categories.png" alt="[AAPCS32] Memory Categories" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">[AAPCS32] 6.2 Processes, Memory and the Stack의 메모리 영역 분류</figcaption>
</figure>


## 스택이란?  
스택(Stack)은 말 그대로 무언가를 “쌓는” 구조입니다.
데이터 구조의 일종으로, 컴퓨터 과학의 여러 분야에서 널리 활용됩니다.  

### 스택 기본 동작: PUSH & POP  
스택에는 데이터를 [쌓는 동작(**PUSH**)과 꺼내는 동작(**POP**)](https://www.w3schools.com/dsa/dsa_data_stacks.php)이 있습니다.  
PUSH는 데이터를 스택 위에 올리는 것이고, POP은 쌓인 데이터 중 가장 위의 데이터를 꺼내는 동작입니다.  

### 스택 vs 큐  

스택(Stack)은 **쌓는 구조**, 큐(Queue)는 **줄 서는 구조**로 비유할 수 있습니다.  

두 구조 모두 선형 데이터 구조이지만, **데이터의 입출력 순서**가 다릅니다.  

- **스택(Stack)**: [LIFO (Last In, First Out)](https://www.geeksforgeeks.org/dsa/lifo-principle-in-stack/) — 나중에 들어온 데이터가 먼저 나옵니다.  
- **큐(Queue)**: [FIFO (First In, First Out)](https://www.geeksforgeeks.org/dsa/introduction-to-queue-data-structure-and-algorithm-tutorials/) — 먼저 들어온 데이터가 먼저 나옵니다.  

스택의 핵심은 **가장 최근의 데이터부터 처리한다는 점**입니다.  
이 특성이 함수 호출과 지역 변수 관리에서 매우 유용하게 활용됩니다.  

## ARM 스택 메모리  
스택 메모리는 메모리 공간 중 스택의 형태로 동작하는 영역을 말합니다.  
ARM의 스택 메모리 영역은 지역 변수나 함수 인자 등 임시 데이터를 저장하기 위한 연속된 메모리 공간입니다.  

### ARM 스택 메모리 구조  
ARM 아키텍처는 기본적으로 **FD(Full Descending)** 구조를 사용합니다.  
즉, 스택이 **주소가 감소하는 방향으로 확장**되며, 스택 포인터(SP)는 **마지막으로 저장된 데이터의 주소**를 가리킵니다.  

<figure style="text-align: center;">
  <img src="/assets/img/arm-stack-fd.svg" alt="스택 메모리의 확장 방향과 SP 값" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">FD 구조 스택의 확장 방향</figcaption>
</figure>

> **참고: 스택 구조의 네 가지 유형**  
> - **Full Stack**: SP가 마지막으로 입력된 데이터의 주소를 가리킴  
> - **Empty Stack**: SP가 비어 있는 영역의 주소를 가리킴  
> - **Descending Stack**: 스택이 주소가 감소하는 방향으로 확장됨  
> - **Ascending Stack**: 스택이 주소가 증가하는 방향으로 확장됨  
>
> 따라서 FD 구조는 “주소가 감소하면서 데이터가 쌓이고, SP는 가장 최근 데이터(Full)를 가리키는” 형태입니다.  

### 스택 포인터(SP)
스택 메모리도 결국 메모리이기 때문에 주소로 접근해야 합니다.  
FD 구조에서는 마지막으로 PUSH된 데이터의 주소를 항상 추적해야 하므로, 이를 담당하는 전용 레지스터가 필요합니다.  
이를 **스택 포인터(Stack Pointer, SP)** 라고 합니다.  

> **ARMv4** 아키텍처에서는 **R13 레지스터**가 SP로 사용됩니다.  

### LDR/STR을 활용한 PUSH & POP 구현  

#### STR로 구현한 PUSH  
PUSH 동작은 데이터를 스택에 추가하는 것입니다. 다음 두 단계로 이루어집니다.  

1. SP 값을 4만큼 감소시킵니다.  
2. 현재 SP 위치에 값을 저장합니다.  

```armasm
  mov sp, #0x80000
  str r1, [sp, #-4]!
```

데이터가 스택의 상단에 쌓이므로, SP는 감소해야 합니다.
ARM의 스택은 FD 구조를 사용하며, 워드(4바이트) 단위로 동작하기 때문에 4를 감소시킵니다.

#### LDR로 구현한 POP
POP은 스택의 최상단 데이터를 꺼내는 동작입니다. 다음 두 단계로 구성됩니다.

1. 현재 SP가 가리키는 값을 불러옵니다.
2. SP 값을 4만큼 증가시킵니다.

```armasm
  mov sp, #0x80000
  ldr r1, [sp], #4
```

FD 구조에서는 SP가 마지막 데이터의 주소를 가리키므로, 먼저 그 값을 읽은 뒤 SP를 증가시켜 다음 데이터로 이동합니다.

#### 예제 코드와 디버깅
**stack-push-pop.s**
```armasm
  .text
  .global _start
_start:
  mov sp, #0x80000
  mov r1, #0x1
  mov r2, #0x2
  mov r3, #0x3
  
  str r1, [sp, #-4]!
  str r2, [sp, #-4]!
  str r3, [sp, #-4]!
  
  ldr r0, [sp], #4
  ldr r0, [sp], #4
  ldr r0, [sp], #4
  
  b .
```

> 이 예제에서는 단순히 스택의 시작 주소를 `0x80000`으로 지정했습니다.
> 실제 시스템에서는 링커 스크립트 또는 커널 초기화 과정에서 SP의 초기값이 설정됩니다.

**컴파일 & QEMU 실행**
```bash
$ arm-none-eabi-gcc -nostdlib -Ttext=0x10000 -o stack-push-pop.elf stack-push-pop.s
$ qemu-system-arm -nographic -machine versatilepb -S -s -kernel stack-push-pop.elf
```

**GDB 디버깅**
```bash
$ gdb-multiarch stack-push-pop.elf

(gdb) target remote :1234    # GDB 서버 접속

# 스택 PUSH 상태 확인
(gdb) x/3w $sp
(gdb) i r r1 r2 r3
(gdb) i r sp                 # STR 실행 후 SP 확인

# 스택 POP 상태 확인
(gdb) i r r0
(gdb) x/3w $sp
(gdb) i r sp                 # LDR 실행 후 SP 확인
```

<figure style="text-align: center;">
  <img src="/assets/gif/" alt="GDB에서 PUSH/POP 확인" style="display: block; margin: auto;" /> 
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  GDB에서 PUSH/POP 동작 확인
  </figcaption>
</figure>


## 마무리
이번 글에서는 스택의 개념과 **ARM의 스택 메모리 구조 및 동작 방식**을 함께 살펴보았습니다.
스택은 함수 호출과 지역 변수 관리에 핵심적인 역할을 담당합니다.

하지만 여러 데이터를 한 번에 저장하거나 복원해야 할 때, `LDR`과 `STR`만 사용하면 다음과 같이 비효율적인 코드가 됩니다.
```armasm
  mov r1, #0x1
  mov r2, #0x2
  mov r3, #0x3
  
  str r1, [sp, #-4]!
  str r2, [sp, #-4]!
  str r3, [sp, #-4]!
  @ ...
  @ 저장하는 데이터의 개수만큼 str 명령을 반복해야 함
```

명령어가 많아질수록 CPU 파이프라인 효율이 떨어지게 됩니다.
이러한 비효율을 해결하기 위한 명령어가 바로 `LDM`(Load Multiple) 과 `STM`(Store Multiple) 입니다.
다음 포스팅에서는 이 두 명령어를 통해 **블록 단위 메모리 접근을 효율적으로 수행하는 방법**을 살펴보겠습니다.