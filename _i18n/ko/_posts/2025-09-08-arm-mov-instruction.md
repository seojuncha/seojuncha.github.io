---
layout: post
lang: ko
ref: "arm-mov-instruction"
title: "ARM 어셈블리 #1 - mov 명령어로 레지스터에 값 저장하기"
date: 2025-09-08 22:24:00 +0900
categories: [arm,assembly,tutorial]
tags: [assembly, embedded, low-level, ARM, QEMU, GDB]
---
컴퓨터는 연산을 하기 위한 기계입니다.

이런 연산은 결국 CPU가 수행하죠.
CPU가 연산을 하기 위해서는 두 가지가 필요합니다:
바로 피연산자(operand) 와 연산자(operator) 입니다.
그리고 이 피연산자들은 대부분 레지스터(register) 에 저장된 값을 사용합니다.

어셈블리 언어는 CPU의 레지스터를 직접 제어할 수 있는 가장 저수준의 언어 중 하나입니다.
오늘은 이 레지스터에 값을 저장하는 기본적인 방법들을 소개합니다.

> 실습에 사용한 코드는 [GitHub 저장소](https://github.com/seojuncha/arm-assembly-tutorial/tree/main/02_moving-data-around)에서 확인할 수 있습니다!

## mov명령어로 레지스터에 값 복사하기

### 즉시값(Immediate Value) 저장
```armasm
  .text
  .global _start
_start:
  mov r0, #3
  b .
```
`mov r0, #3`: 숫자 3을 레지스터 R0에 저장합니다.

`mov` 문법은 단순합니다:
```
mov 목적지 레지스터, 값
```
여기서 `#3`처럼 숫자 앞에 # 기호가 붙은 값은 **즉시값(immediate value)** 이라고 부릅니다.
즉시값은 `#3`처럼 10진수 또는 `#0x3`처럼 16진수로도 표현할 수 있습니다.

> 컴파일과 QEMU, GDB설치는 [이전 포스팅](hello-arm-assembly)를 참고하세요.

### 컴파일
```bash
$ arm-none-eabi-gcc -nostdlib -Ttext=0x10000 imm-to-reg.s -o imm-to-reg.elf
```

### QEMU 실행 및 GDB 디버깅

#### QEMU 실행

```bash
qemu-system-arm \
  -machine versatilepb \
  -nographic \
  -S -s \
  -kernel imm-to-reg.elf
```
qemu의 공식문서를 살펴보면 굉장히 많은 옵션들을 제공하고 있는 것을 볼 수 있습니다.
우선은 꼭 필요한 옵션들만 사용해보겠습니다.

간단히 각 옵션을 설명하자면:

| 옵션 | 설명 |
|:--|:--|
| `-machine versatilepb` | QEMU에서 ARM 보드 중 하나인 VersatilePB 사용 |
| `-nographic` | GUI 창 없이 콘솔에서 실행 |
| `-S` | CPU 실행을 멈추고 대기 (GDB 연결용) | 
| `-s` | 포트 1234번에서 GDB 서버 실행 | 
| `-kernel` | 실행할 ELF 파일 지정 | 

> 처음 실행 시 QEMU가 잠시 멈춘 것처럼 보일 수 있습니다. 하지만 이는 정상이며, GDB 연결을 기다리는 중입니다.

#### GDB 연결
```bash
$ gdb-multiarch imm-to-reg.elf
```
![GDB실행 이후 화면]()

GDB에 들어간 후 다음 명령으로 QEMU의 GDB 서버에 연결합니다.
```gdb
(gdb) target retmote localhost:1234
```

![GDB 서버 접속 이후 화면]()

QEMU와 연결되었다면, 프로그램이 메모리의 코드 영역에 제대로 로드됐는지 확인해 봅니다.
특정 메모리 주소의 값을 확인하는 명령어는 `x/{숫자}{타입}`의 포맷을 사용합니다.

```gdb
(gdb) x/10i 0x10000
```

![시작주소부터 명령어 10개 출력]()

`info registers` 명령으로 현재 레지스터 상태를 확인할 수 있습니다.

```gdb
(gdb) info registers   # 또는 간단히 i r
(gdb) i r pc r0        # PC(프로그램 카운터)와 R0 값만 출력
```

![PC와 R0 레지스터 출력 결과]()

현재 PC는 `0x10000`을 가리키고 있으므로 다음에 실행할 명령어는 `mov r0, #3`이라는 것을 알 수 있습니다.
PC가 가리키는 명령어를 실행하기 위해선 다음 명령어 실행을 의미하는 `stepi`를 사용합니다.

#### stepi로 한 줄씩 실행
```gdb
(gdb) stepi   # 혹은 간단히 s i
```
다시 레지스터를 확인해 보면 다음과 같은 변화가 있습니다:
```gdb
(gdb) i r pc r0
```
- PC: `0x10000` -> `0x10004`
- R0: `0` -> `3`

`step`를 한 번 더 실행하면 `b .` 명령어로 돌아가며 무한 루프 상태가 됩니다.  
`b .`은 현재 PC 위치로 계속 이동하므로 PC값은 변하지 않고, 따라서 프로그램은 무한루프 상태로 유지됩니다.

### 레지스터 간 값 복사
이전 예제에서는 `#3` 으로 표현하는 즉시값을 레지스터 `R0` 에 할당했습니다.
이제 레지스터 내부의 값을 다른 레지스터로 복사하는 방법을 알아보겠습니다.

```armasm
  .text
  .global _start
_start:
  mov r0, #3
  mov r1, r0
  b .
```
`mov r1, r0`: `R0`의 값을 `R1`에 복사합니다.

> 디버깅 과정은 동일하므로 생략합니다.


## mov vs mvn 은 어떤 차이점이 있을까?
`mov`와 유사한 `mvn` 명령어를 하나 더 살펴보겠습니다.
둘다 동일하게 레지스터에 값을 복사하는 역할지만, `mvn`은 입력값의 [1의 보수](https://ko.wikipedia.org/wiki/1%EC%9D%98_%EB%B3%B4%EC%88%98)를 저장하는 명령입니다.

|mvn|mov|
|:--|:--|
|mvn r0, #0|mov r0, #-1|
|mvn r0, #-1|mov r0, #0|
|mvn r0, #0xF|mov r0, #-16|

> 팁! 저는 이런 보수계산과 16진수 변환들은 파이썬 REPL을 활용합니다.
> ```python
> >>> ~0xf
> -16
> ```


## 다음 포스팅 예고: Shift Operand
`mov`, `mvn`과 같은 명령어는 ARM 아키텍처에서 **데이터 처리 명령어(Data Processing Instructions)** 에 속합니다.
이들 명령어는 피연산자에 쉬프트를 적용할 수 있는 특징이 있습니다.

이 기능은 `add`, `sub` 같은 연산에도 동일하게 사용되며, 다음 포스팅에서 자세히 다룰 예정입니다.

![ARM 레퍼런스 매뉴얼의 쉬프트 피연산자들]()

## 마무리
이번 포스팅에서는 `mov`, `mvn` 명령어를 중심으로 레지스터에 값을 저장하고 복사하는 기본적인 방법을 알아보았습니다.
또한 QEMU와 GDB를 활용한 실습을 통해 우리가 작성한 코드가 실제로 어떻게 동작하는지 확인해보았습니다.

처음엔 복잡하게 느껴질 수 있지만, 앞으로 계속해서 이 구조를 반복적으로 사용하게 될 것입니다.
이번 포스팅이 앞으로 나올 더 복잡한 예제의 기반이 되길 바랍니다.

다음 포스팅에서는 **Shift Operand**에 대해 구체적으로 다룰 예정입니다.
