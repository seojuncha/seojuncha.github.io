---
layout: post
lang: ko
ref: "arm-assembly-pre-post-index"
title: "ARM 어셈블리 #9 - pre-index와 post-index로 자동 주소 계산하기"
date: 2025-11-03 12:40:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["ldr", "str", "pre-index", "post-index", "addressing mode"]
published: false
---

ARM 어셈블리에서 메모리에 접근할 때, 단순히 `[R1]` 형식으로 주소를 지정하는 것만으로는 부족할 때가 있습니다. 
배열이나 버퍼처럼 연속된 데이터를 다룰 때, 매번 `ADD` 명령으로 주소를 갱신하는 것은 비효율적이기 때문입니다. 
이 문제를 해결하기 위해 ARM은 **pre-index**와 **post-index**라는 자동 주소 계산 기능을 제공합니다. 
이 글에서는 두 방식의 차이와 동작 원리를 이해하기 쉽게 설명하고, 마지막에는 실제 예제 코드와 함께 CPU 내부에서 주소가 어떻게 계산되는지도 알아보겠습니다.  

## pre-index / post-index가 왜 필요한가  
[이전 포스팅]({% post_url 2025-11-02-arm-offset-addressing %})에서는 `[Rn, #offset]` 형태의 기본 오프셋 주소 계산 방식을 다뤘습니다.
이 방식은 메모리 접근 시, 단순히 기준 주소(`Rn`)에 일정한 오프셋 값을 더한 뒤 그 주소로 접근하는 방법이었죠.
하지만 오프셋 계산은 명령이 실행될 때마다 반복해서 수행되며, 레지스터 `Rn`의 값은 자동으로 갱신되지 않습니다.

ARM에서 메모리에 접근할 때, CPU는 대괄호 안의 표현식을 먼저 계산하여 실제 주소를 얻습니다.  
예를 들어 `LDR R0, [R1]`은 `R1`이 가리키는 주소로부터 데이터를 읽습니다.  
하지만 만약 배열처럼 연속된 데이터를 반복해서 읽는다면, 매번 `ADD R1, R1, #4` 같은 명령을 추가로 실행해야 하므로 코드가 길어지고 성능이 떨어집니다.

이 문제를 해결하기 위해 ARM은 메모리 접근 명령(`LDR`/`STR`) 자체에 주소를 자동으로 계산하는 기능을 포함시켰습니다.  

즉, CPU가 `ADD`를 대신 수행하도록 만들어 명령어 수를 줄이고 파이프라인 효율을 높인 것이죠.

## 자동 주소 계산이란 무엇인가  
CPU가 명령을 실행하는 동안, 메모리 접근 명령이 자동으로 주소를 증가시키거나 감소시키는 기능을 말합니다.  
이 기능은 두 가지 방식으로 동작합니다: **pre-index**와 **post-index**입니다.  

### pre-index 방식  
pre-index는 말 그대로 **"먼저 인덱스를 더한 뒤 접근"**하는 방식입니다.  
즉, CPU는 `[Rn, #offset]!`의 형태를 만나면 먼저 `Rn + offset`을 계산하여 Rn에 저장하고,  
그 주소로 메모리에 접근합니다.  

예를 들어 아래 코드를 보겠습니다.

```armasm
ldr r0, [r1, #4]!   @ R1에 4를 더한 뒤 그 주소에서 값을 읽는다
```

이 명령은 다음과 같은 순서로 실행됩니다:

1. R1 ← R1 + 4
2. R0 ← Mem[R1]

즉, 주소를 먼저 갱신하고 그 주소를 사용해 메모리에 접근합니다.

### post-index 방식
post-index는 반대로 **"먼저 접근하고 나중에 인덱스를 더하는"** 방식입니다.
즉, `[Rn], #offset` 형태를 만나면 CPU는 현재 `Rn`이 가리키는 주소로 접근한 뒤,
그 다음에 `Rn + offset` 계산 결과를 Rn에 저장합니다.

예를 들어 아래 코드처럼 동작합니다.
```armasm
ldr r0, [r1], #4    @ R1이 가리키는 주소에서 값을 읽은 뒤 R1에 4를 더한다
```

실행 순서는 다음과 같습니다:
1. R0 ← Mem[R1]
2. R1 ← R1 + 4

## 메모리 접근 시점과 주소 갱신 시점 비교
두 방식 모두 결과적으로 `Rn`은 `Rn + offset`으로 변경되지만, 어떤 시점에 변경되느냐에 따라 동작이 달라집니다.  
아래 두 코드를 비교해보겠습니다.

```armasm
ldr r0, [r1, #4]!   @ pre-index: R1에 4를 더한 후 그 주소로 메모리 접근
ldr r0, [r1], #4    @ post-index: R1이 가리키는 주소에서 읽은 뒤 R1에 4를 더함
```
- 첫 번째 명령은 `R1`이 먼저 갱신된 뒤 메모리에 접근합니다.
- 두 번째 명령은 `R1`이 나중에 갱신됩니다.

따라서, pre-index는 “다음 주소부터 접근해야 할 때”,
post-index는 “현재 주소를 먼저 사용해야 할 때” 적합합니다.

## 예제 코드 실행 (GCC / QEMU / GDB)
아래 코드는 두 방식을 비교하기 위한 간단한 예제입니다.

**pre-post-index.s**
```armasm
  .text
  .global _start
_start:
  ldr r1, =array
  ldr r0, [r1, #4]!   @ pre-index: R1 ← R1+4 후 로드
  ldr r2, [r1], #4    @ post-index: R2 ← Mem[R1], 그 후 R1 ← R1+4
  b .
array:
  .word 0x11111111
  .word 0x22222222
  .word 0x33333333
  .word 0x44444444
```

**컴파일 & QEMU실행**:
```bash
$ arm-none-eabi-gcc \
        -O0 \
        -nostdlib \
        -march=armv4 \
        -Ttext=0x10000 \
        pre-post-index.s \
        -o pre-post-index.elf
$ qemu-system-arm -nographic -S -s -kernel pre-post-index.elf
```

**디버깅**:
```bash
$ gdb-multiarch 

(gdb) target remote :1234    # GDB서버 접속 
(gdb) i r r0 r1 r2           # R0, R1, R2 레지스터 값 출력
(gdb) x/4w 0x10010           # 메모리 주소 0x10010의 4개 워드값 출력
```

<object type="image/svg+xml" data="/assets/gif/gdb-prepost.svg" width="720"></object>

- pre-index 명령 실행 후에는 `R1`이 `array+4`로 변경되고,
- post-index 명령 실행 후에는 `R1`이 `array+8`로 변경됩니다.

이 과정을 통해 메모리 접근 시점의 차이를 직접 확인할 수 있습니다.

## 하드웨어 관점에서 보기
CPU 내부에서는 명령이 파이프라인 단계를 거치며 실행됩니다.
보통 주소 계산은 Execute 단계에서 ALU가 수행하고, 메모리 접근은 Memory 단계에서 발생합니다.

- pre-index는 Execute 단계에서 주소를 먼저 계산하고, 계산된 주소로 Memory 단계에서 접근합니다.
- post-index는 Memory 단계에서 접근한 후, Write Back 단계에서 주소를 갱신합니다.

이러한 설계 덕분에 CPU는 `add` 명령 없이도 주소를 자동으로 갱신할 수 있으며,
이는 반복적인 메모리 접근에서 명령어 수를 줄여 파이프라인 효율을 높입니다.

## 마무리
pre-index와 post-index는 모두 **주소 갱신을 자동화**해주는 기능이지만,
그 차이는 **주소 갱신이 메모리 접근보다 앞인지, 뒤인지**입니다.

pre-index는 다음 데이터를 미리 읽어야 하는 경우에,
post-index는 현재 데이터를 읽고 다음으로 넘어갈 때 유용합니다.

다음 포스팅에서는 이 개념을 확장하여, **CPU가 메모리를 어떻게 인식하고 접근하는지(명령어 파이프라인 동작 포함)**를 자세히 살펴보겠습니다.