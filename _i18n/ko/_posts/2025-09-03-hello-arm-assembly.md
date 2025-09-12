---
layout: post
lang: ko
ref: hello-arm-assembly
title: "ARM 어셈블리 #0 - 개발환경과 첫 코드"
date: 2025-09-03 23:24:00 +0900
categories: [arm,assembly,tutorial]
tags: [assembly, embedded, low-level, ARM, QEMU, GDB]
---

이번 포스팅에서는 ARM 기반의 어셈블리 코드를 작성하고 실행하는 방법을 소개합니다. 향후 이어질 시리즈의 기본이 되는 내용이므로, 차근차근 따라오시길 추천드립니다. 간단한 어셈블리 코드를 작성하고, 컴파일, 실행, 디버깅하는 기본적인 흐름을 익히는 것이 목표입니다. 또한 이후 포스팅에서 사용할 QEMU 가상환경과 GDB 디버깅 도구도 함께 준비합니다.

## 왜 어셈블리를 배우는가?
제가 어셈블리를 공부하게 된 계기는 [C 컴파일러](https://github.com/seojuncha/ftt-c-compiler)의 코드 생성(codegen) 단계를 직접 구현하려 했던 경험에서 비롯됐습니다.  
컴파일러는 소스 코드를 파싱한 뒤, 중간코드(IR)나 어셈블리 코드를 생성하는데, 이 과정에서 "C 코드가 어떤 어셈블리 코드로 변환되어야 하는가?"에 대한 이해가 부족해 구현이 어려웠습니다.

<figure style="text-align: center;">
  <img src="/assets/img/ast-output-of-my-compiler.png" alt="자체 제작 컴파일러의 AST" width="70%" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">AST는 만들었는데… 도대체 여기서 어떤 어셈블리를 만들어야 하지?</figcaption>
</figure>

어셈블리를 알게 되면, 최적화된 코드 작성, 디버깅, 리버스 엔지니어링 등에서도 큰 도움이 됩니다.  
> "어셈블리를 알면 모든 프로그램이 오픈소스다." 라는 말이 있죠.  
> 물론 현실에서는 ISA(Instruction Set Architecture)를 함께 이해해야 가능한 이야기입니다. 

## 간단한 어셈블리 코드 예시
이제 각 시리즈에서 사용될 기반 코드의 가장 간단한 형태를 살펴보겠습니다.

**only-main.s**
```nasm
  .text
  .global _start
_start:
  b .
```

이 코드는 아무런 기능도 수행하지 않지만, ARM 어셈블리 프로그래밍의 시작점으로 아주 훌륭합니다.
이렇게 짧은 코드 한 줄로도 무한 루프를 만들 수 있습니다.

> C 언어나 Python에서 “Hello, World!“를 출력하는 것도 어셈블리로는 꽤 복잡합니다.
> 따라서 먼저 최소한의 어셈블리 코드로부터 시작해보는 것이 좋습니다.

## 환경 설정
개발은 Debian Linux 환경에서 진행되었으며, Ubuntu에서도 동일하게 적용 가능합니다.
기타 리눅스 배포판 사용자는 패키지 매니저에 맞게 설치하시면 됩니다.

ARM 어셈블리 코드를 작성하고 실행하기 위해 필요한 도구는 다음과 같습니다:
-	arm-none-eabi-gcc (ARM용 크로스 컴파일러)
-	qemu-system-arm (ARM 가상머신)
-	gdb-multiarch (멀티 아키텍처 디버거)

### 1. ARM용 GCC 설치
우리가 사용하는 대부분의 PC는 x86 아키텍처 기반입니다.
그렇기 때문에 기존의 GCC로는 ARM용 바이너리를 만들 수 없습니다.

우리가 설치할 도구는 ARM용 크로스 컴파일러이며,
[공식 툴체인](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)에서 다운로드 가능합니다.

-	OS: Linux (x86_64)
-	Target: bare-metal
-	Format: arm-none-eabi-gcc.tar.gz

<figure style="text-align: center;">
  <img src="/assets/img/arm-toolchain-download-page.png" alt="ARM툴체인 다운로드 페이지" style="display: block; margin: auto;" />
</figure>

> GUI 환경이 아니라면 `wget`으로 다운로드할 수도 있습니다:
> ```bash
> $ wget https://developer.arm.com/path/to/your/toolchain.tar.gz
> ```

압축 해제 후, bin 디렉토리를 PATH에 등록합니다.

설치 확인:
```bash
$ arm-none-eabi-gcc --version
```

### 2. QEMU 설치
QEMU는 다양한 CPU 및 보드를 에뮬레이션할 수 있는 강력한 가상머신입니다.
ARM 하드웨어 없이도 ARM 프로그램을 실행할 수 있기 때문에, 어셈블리 학습에 매우 유용합니다.

[QEMU 공식 다운로드 페이지](https://www.qemu.org/download/)

Debian/Ubuntu에서는 다음 명령으로 설치할 수 있습니다:
```bash
$ sudo apt-get install qemu-system qemu-user-static
```

### 3. GDB (Multiarch) 설치
어셈블리 프로그램을 제대로 디버깅하기 위해 GDB가 필요합니다.
ARM 바이너리를 분석할 수 있는 gdb-multiarch를 설치합니다:

```bash
$ sudo apt-get install gdb-multiarch
```

설치 후, GDB 내부에서 `target remote` 명령을 통해 QEMU에 연결할 수 있습니다.

## 어셈블리 코드 컴파일
작성한 `.s` 어셈블리 파일을 컴파일해 실행 가능한 바이너리(ELF)를 생성합니다.

```bash
$ arm-none-eabi-gcc \
	-O0 \
	-nostdlib \
	-march=armv4 \
	-Ttext=0x10000 \
	only-main.s \
	-o only-main.elf
```

사용한 컴파일 옵션은:
- `-O0`: 최적화 비활성화
- `-nostdlib`: 표준 라이브러리 미포함
- `-march=armv4`: 타겟 아키텍쳐 지정
- `-Ttext=0x10000`: text 섹션 시작 주소


최종 결과물로 `only-main.elf`라는 파일이 생성됩니다.
```bash
$ file only-main.elf
only-main.elf: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, not stripped
```

## 코드 구조 설명
우리가 작성한 `only-main.s` 코드는 총 4줄에 불과하지만, 어셈블리의 핵심 개념이 모두 담겨 있습니다.
각 줄이 의미하는 바를 하나씩 살펴보겠습니다.

### .text - 코드 영역 지정 
```nasm
.text
```
어셈블리에서는 .(dot)으로 시작하는 지시어(directive)를 사용하여 프로그램의 구조를 정의합니다.
`.text`는 이 아래에 작성될 명령어들이 **코드 영역(Text Section)**에 위치함을 의미합니다.
즉, 실제 실행 가능한 명령어들이 저장되는 메모리 영역을 가리킵니다.

> 향후 포스팅에서 다룰 ELF 구조나 메모리 맵에서도 `.text`, `.data`, `.bss` 같은 섹션은 중요한 개념입니다.

<figure style="text-align: center;">
  <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/5/50/Program_memory_layout.pdf/page1-234px-Program_memory_layout.pdf.jpg" alt="Wiki 프로그램 메모리 구조" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">출처: <a href="https://en.wikipedia.org/wiki/Data_segment#Program_memory">[Wikipedia - Program memory]</a></figcaption>
</figure>

### .global _start - 외부 심볼로 내보내기
```nasm
.global _start
```
`.global` 지시어는 해당 레이블(_start)을 외부 심볼로 노출합니다.
이것은 링커가 이 심볼을 **프로그램의 진입점(entry point)**으로 인식할 수 있도록 하기 위해 필요합니다.

C 프로그램에서는 `main()` 함수가 시작점이지만, 어셈블리에서는 `_start`와 같은 명시적인 entry point가 필요합니다.

### _start: - 레이블 정의
```nasm
_start:
```
콜론(:)은 **레이블(label)**을 정의합니다.
레이블은 해당 위치에 이름을 붙여주는 것으로, 이후 분기(branch)나 함수 호출 등의 명령에서 참조할 수 있습니다.

여기서 `_start`는 `.global`로 외부에 노출되며, 링커가 프로그램의 시작 주소로 사용합니다.

### b . - 현재 위치로 분기 (무한 루프)
```nasm
b .
```
b는 분기(Branch) 명령어입니다.
실행 흐름을 지정된 주소(혹은 레이블)로 이동시킵니다.

.(dot)는 현재 명령어의 주소를 의미합니다. 즉, “지금 위치로 다시 분기하라”는 의미이며, 이로 인해 무한 루프가 형성됩니다.

> 이 무한 루프는 디버깅 시 프로그램이 끝나지 않고 멈춰있도록 하기 위한 의도적인 구조입니다.
> 추후 gdb로 연결해 레지스터나 메모리 상태를 확인할 때 유용합니다.


## 마무리
이처럼 짧은 코드이지만, ARM 어셈블리에서 자주 사용하는 핵심 구조를 모두 담고 있습니다.
실제로 이 구조는 이후 장에서 다루게 될 대부분의 예제의 기본 템플릿이 될 것입니다.

이번 포스팅에서는 가장 단순한 ARM 어셈블리 코드를 작성하고, 컴파일 및 실행을 준비했습니다.
다음 포스팅에서는 이 코드를 확장하여 CPU 레지스터에 값을 저장하고, QEMU와 GDB를 통해 상태를 확인해보겠습니다.
