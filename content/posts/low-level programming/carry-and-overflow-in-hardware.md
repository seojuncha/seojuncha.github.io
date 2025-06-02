+++
date = '2025-01-20'
title = 'ARM프로세서의 Carry Flag와 Overflow Flag'
tags = ["carry", "overflow", "arm", "arm assembly", "cpsr", "qemu", "gdb"]
+++

하드웨어의 **Carry** 플래그(C)와 **Overflow** 플래그(V) 동작을 프로그래머 입장에서 깊이 이해하는 것은 다양한 측면에서 큰 장점을 제공합니다. 특히, **저수준 시스템 프로그래밍, 임베디드 시스템 개발, 그리고 성능 최적화와 디버깅에서 매우 중요한 역할**을 합니다. 예를 들어,
1. 정확한 수치 연산 구현
2. 고성능 저수준 최적화
3. 디버깅과 문제 해결
4. 오류 검출 및 안정성 향상
5. 다중 정밀도 연산 구현
6. 조건부 분기와 코드 단순화

위와 같은 다양한 장점을 제공합니다. 이번 포스팅에서는 ARM 프로세서에서 **[Carry와 Overflow](carry-and-overflow-kr.md)를 처리하는 방법**을 알아보고 간단한 **어셈블리(*Assembly*) 예제**를 사용해 **QEMU에서 디버깅** 해보겠습니다. 

이런 과정을 통해 우리가 작성한 산술연산 코드가 하드웨어에서 어떻게 동작하는지 깊이 이해함으로써 더 나은 코드 작성과 원활한 문제해결을 도와줄 것 입니다.

## 32-bit ARM Processor
본 포스팅에서는 현재 진행하고 있는 [From the transistor](from-the-transistor-intro.md)프로젝트에서 사용중인 32비트 ARM 프로세서의 내용을 기반으로 합니다.

32비트 프로세서는 **한번에 처리할 수 있는 데이터 크기**와 **메모리 주소 크기**가 32비트 임을 의미합니다.

- 레지스터(Register) 크기가 32비트
- 데이터 버스 및 메모리 버스 크기가 32비트
- 메모리 정렬(Memory Align)단위가 32비트
- 워드(Word)단위가 32비트

## CPSR: Current Program Status Register
ARM 프로세서는 프로그램의 현재 상태를 저장하기 위한 **CPSR\(Current Program Status Register\)** 이라는 특별한 레지스터(*Register*)를 가지고 있습니다.

**CPSR Format**
```
32-bit CPSR
+---+---+---+---+---+----------+---+---+---+----+----+----+----+----+
| N | Z | C | V | Q | DNM(RAZ) | I | F | T | M4 | M3 | M2 | M1 | M0 |
+---+---+---+---+---+----------+---+---+---+----+----+----+----+----+
```

### Condition Code Flags
*CPSR*에서 상위 4비트[31:28]는 프로그램의 상태 코드(*Condition Code*)를 검사하기 위한 다음의 4가지 비트 필드가 있습니다.
- N (***N***egative)
- Z (***Z***ero)
- C (***C***arry)
- V (o***V***erflow)

상태 코드 플래그(*Condition Code Flags*)라는 것은 특정 조건을 만족할 때 1비트 공간에 1을 할당(set)하고 만족하지 않을 때 0을 할당(clear)합니다.

> 보통 플래그(Flag)라는 용어는 참(1) 혹은 거짓(0), 두가지 상태를 제어합니다.
>  참인 상태로 만드는 것을 `set`, 거짓인 상태로 만드는 것을 `clear` 라고 표현합니다.

#### N Flag
- Set : 음수일 때
- Clear : 0 혹은 양수일 때

#### Z Flag
- Set : 0일 때
- Clear : 0이 아닐 때

#### C Flag
- Set : Carry가 발생했을 때
- Clear : Carry가 발생하지 않았을 때

#### V Flag
- Set : Overflow가 발생했을 때
- Clear : Overflow가 발생하지 않았을 때

> **무엇의 조건을 검사하는가?**  
> CPU(예: ARM)는 결국 바이너리 인코딩된 명령어(*Instruction*)를 실행하는 기계입니다. CPSR에서 설정하는 위의 4가지 플래그들은 모두 **명령어 실행 결과를 검사**하는 것 입니다.
>
> 예: 명령어의 연산 결과가 0인경우 -> Z Flag Set

## Carry Flag & Overflow Flag
Carry Flag와 Overflow Flag는 연산의 종류와 CPU 명령어의 유형에 따라 조금씩 다른 동작을 합니다.
ARM 명령어는 크게 다음의 3가지 유형으로 분류할 수 있습니다.
- 데이터 처리 명령(*Data-Processing Instructions*)
  - 산술, 논리, 쉬프트 연산 등, 데이터 연산을 수행
  - `ADD`, `MOV`, `AND`, 등
- 데이터 전송 명령(*Load and Store Instructions*)
  - 메모리와 레지스터간의 데이터를 전송
  - `LDR`, `STR`, 등
- 분기 명령 (*Branch Instructions*)
  - 프로그램의 실행흐름을 제어
  - `B`, `BL`, 등

> 이 외에도 추가적인 명령어 유형으로 구분할 수도 있지만 우선은 3가지로 알고 있어도 무방합니다.  
> ARM 어셈블리에 관한 깊은 내용은 추후 포스팅으로 다룰 예정입니다.

### ARM Assembly with suffix 's'
하지만 모든 데이터 처리 명령어가 상태 플래그를 업데이트하지는 않습니다. 이는 프로그래머에게 효율성과 유연성을 제공하기 위해서 입니다. 상태 플래그의 업데이트도 결국 하드웨어 동작이 필요하기 때문에 모든 명령어에 사용할 경우 전력낭비와 성능 저하를 야기합니다. 

```armasm
ADD r0, r1, r2   @ 상태 플래그 업데이트 하지 않음
ADDS r0, r1, r2  @ 상태 플래그 업데이트
```
## Assembly Example: Compare "ADD" and "ADDS"
지금까지의 내용을 실제 ARM 어셈블리 코드로 확인해보겠습니다. 

> C 언어 예제를 사용하지 않은 이유는 어셈블리 코드의 생성이 컴파일러에 의존적이기 때문입니다.  
> C로 작성된 코드가 어셈블리 코드로 변환되는 것은 전적으로 컴파일러의 역할입니다.
> 코드의 논리구조 및 최적화 등을 고려하여 `S` 접미사(*Suffix*)를 사용할수도, 안 할수도 있습니다.  
>
> 따라서, 예제코드의 간소화를 위해 어셈블리를 사용합니다.

### Environment
- Ubuntu 22.04
- arm-none-eabi-gcc 10.3.1
- gdb-multiarch 12.1

### Code Examples
`add.s`와 `adds.s`, 2개의 어셈블리 코드가 있습니다.   
아래와 같이 테스트 시나리오를 설정합니다.

1. r0 레지스터에 32비트 최대값(0xFFFF_FFFF)를 할당
2. r0 레지스터의 값에 더하기 2를 하고 r1 레지스터에 저장
    - **add를 사용**했을 때, CPSR 변화 확인
    - **adds를 사용**했을 때, CPSR 변화 확인

**add.s**
```armasm
.section .text
.global _start

_start:
  mov r0, #0xffffffff
  add r1, r0, #2
```

**adds.s**
```armasm
.section .text
.global _start

_start:
  mov r0, #0xffffffff
  adds r1, r0, #2
```

**Compile**:
```shell
$ arm-none-eabi-gcc -g -o add.elf add.s -nostdlib -specs=nosys.specs
$ arm-none-eabi-gcc -g -o adds.elf adds.s -nostdlib -specs=nosys.specs
```

### Run on QEMU
QEMU 에뮬레이터를 사용해서 각각의 파일이 32비트 ARM 프로세서에서 실행되도록 합니다.
```shell
$ qemu-system-arm -M versatilepb -nographic -s -S -kernel add.elf
or
$ qemu-system-arm -M versatilepb -nographic -s -S -kernel adds.elf
```

> QEMU 사용법에 관해선 별도의 포스팅이 차차 제작될 예정입니다.  
> 궁금한 점은 [QEMU 공식 문서](https://www.qemu.org/docs/master/)를 확인해주세요.

### Debug with gdb
`gdb`를 사용해 각 명령어마다 실행하며 CPSR 값들을 하나씩 확인해보겠습니다.  

**사용하는 gdb 명령어**  
- `target remote localhost:1234` 
  - QEMU로 실행중인 로컬 gdb서버에 연결
- `si`
  - CPU 명령어 실행
- `i r r0 r1 cpsr`
  - `i r`
    - CPU 레지스터값 출력
  - `r0 r1 cpsr`
    - r0, r1, cpsr 레지스터값 출력

#### Use "ADD" instruction

**1. gdb 실행**
```shell
$ gdb-multiarch add.elf
```

**2. 원격 gdb 서버 연결**
```shell
(gdb) target remote localhost:1234
_start () at add.s:5
5         mov r0, #0xffffffff
```
다음에 실행될 명령어가 `mov r0, #0xffffffff`라고 알려줍니다.

**3. 최초 CPSR 확인**
```shell
(gdb) i r r0 r1 cpsr
r0             0x0                 0
r1             0x0                 0
cpsr           0x400001d3          1073742291
```
`0x400001d3`은 이진수로 `0100 0000 0000 0000 0000 0001 1101 0011` 입니다.
상위 4비트를 살펴보면, 
- N : 0
- Z : 1
- C : 0
- V : 0

이므로, 레지스터 초기화로 인한 Z(Zero) 플래그만 설정되어 있습니다.

**4. mov 명령 실행**
```shell
(gdb) si
6         add r1, r0, #2
```
`si` 은 CPU명령을 수행하는 명령어 입니다.  
그렇기 때문에, `mov r0, #0xffffffff`를 실행하고 그 다음 실행할 명령어가 `add r1, r0, #2` 라고 알려줍니다.


**5. mov 명령 실행에 대한 CPSR 상태 플래그 확인**
```shell
(gdb) i r r0 r1 cpsr
r0             0xffffffff          -1
r1             0x0                 0
cpsr           0x400001d3          1073742291
```
`mov` 명령어로 `r0`레지스터에 `0xffffffff`를 할당했기 때문에, CPSR의 Z플래그가 클리어 되어야 하지만, `s` 접미사가 없으므로 기존 플래그 값을 유지합니다.


**6. add 명령 실행**
```shell
(gdb) si
0x00008008 in ?? ()
```
다음 명령어인 `add r1, r0, #2`를 실행했습니다.

**7. add 명령 실행에 대한 CPSR 상태 플래그 확인**
```shell
(gdb) i r r0 r1 cpsr
r0             0xffffffff          -1
r1             0x1                 1
cpsr           0x400001d3          1073742291
```
`add r1, r0, #2`를 실행했을 때의 CPSR플래그 역시 `s` 접미사가 없으므로 기존 플래그 값을 유지합니다.


#### Use "ADDS" instruction
동일한 과정으로, 이번에는 ADDS를 사용했을 때 레지스터 값 변화를 살펴보겠습니다.

```shell{hl_lines=[11,21]}
$ gdb-multiarch adds.elf
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
_start () at adds.s:5
5         mov r0, #0xffffffff
(gdb) i r r0 r1 cpsr
r0             0x0                 0
r1             0x0                 0
cpsr           0x400001d3          1073742291
(gdb) si
6         adds r1, r0, #2
(gdb) i r r0 r1 cpsr
r0             0xffffffff          -1
r1             0x0                 0
cpsr           0x400001d3          1073742291
(gdb) si
0x00008008 in ?? ()
(gdb) i r r0 r1 cpsr
r0             0xffffffff          -1
r1             0x1                 1
cpsr           0x200001d3          536871379
```
동일한 디버깅 과정을 거쳤을 때, 마지막 CPSR 결과는 `0x200001d3` 입니다.  
`0x200001d3`는 이진수로 `0010 0000 0000 0000 0000 0001 1101 0011` 이고, 상위 4비트는 아래와 같습니다.
- N : 0
- Z : 0
- C : 1
- V : 0

`adds`연산의 결과는 32비트 최대 값인 `0xffffffff`를 초과하였기 때문에 Carry가 발생하고, 덧셈의 결과인 `1`은 부호 있는 정수의 범위를 초과하지 않았기 때문에 Overflow는 발생하지 않습니다. 따라서, CPSR의 Carry 플래그를 활성화하고 Overflow 플래그는 활성화 하지 않는 것을 볼 수 있습니다.

## Conclusion
이번 포스팅에서는 ARM프로세서에서 Carry와 Overflow를 처리하는 방법과 어셈블리 코드 예제를 활용하여 QEMU와 gdb로 실제 동작을 확인했습니다. 사실 하드웨어 레벨에서는 Carry와 Overflow를 검출하는 것이 주 목적이고 이에 대한 처리는 소프트웨어의 몫입니다. 그렇기 때문에 프로그래머는 검출 조건과 방법을 이해해야 더 나은 코드를 작성할 수 있습니다.

다음 포스팅
- 하드웨어에서 산술연산은 어떻게 처리되는가?
- Carry와 Overflow 검출은 어떻게 구현할까?
- QEMU를 활용한 ARM 어셈블리 활용

더욱 자세한 설명이나 궁금하신 점은 편하게 댓글로 알려주세요!