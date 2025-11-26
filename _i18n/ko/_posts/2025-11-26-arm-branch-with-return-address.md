---
layout: post
lang: ko
ref: "arm-branch-with-return-address"
title: "ARM 어셈블리 #14 - 복귀주소를 자동 저장하는 BL 명령어 활용"
date: 2025-11-26 18:30:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["arm bl", "arm branch instruction", "link register", "arm lr"]
---

`BL` 명령은 [이전 포스팅]({% post_url 2025-11-23-arm-basic-branch %})에서 설명한 `B` 명령에 함수 호출 기능을 더한 명령어입니다.
`B` 명령이 단순히 특정 레이블로 점프하는 역할이었다면, `BL` 명령은 점프 이후 복귀해야 하는 주소를 `LR`(Register `R14`)에 자동으로 저장합니다.
따라서 `BL`은 ARM에서 함수 호출을 구현하는 핵심 매커니즘이며, 스택, 함수 호출 규약, 프레임 포인터 등 이후에 배우게 될 개념들의 기반이 됩니다.

<figure style="text-align: center;">
  <a href="https://youtu.be/cMMjuVVKaS0" target="_blank">
  <img src="/assets/img/ARM Assembly - bl instruction.png"
    alt="Youtube Video to explain ARM BL instruction"
    style="display: block; margin: auto;" /> 
  </a>
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  <a href="https://www.youtube.com/@seojuncha" target="_blank">youtube.com/@seojuncha</a>
  </figcaption>
</figure>

[GitHub 샘플코드](https://github.com/seojuncha/arm-assembly-tutorial/blob/main/05_branching/branch-with-link.s)

## 복귀주소 저장의 필요성
함수를 호출하면 함수 바디가 끝난 뒤 원래 호출 지점으로 되돌아와야 합니다.
이 복귀 주소를 모르면 다음에 실행해야 할 명령어가 어디인지 알 수 없기 때문에, 별도로 저장해둘 필요가 있습니다.

## BL 명령
```armasm
  bl label
```
문법은 `b label`과 동일하지만 내부 동작이 다릅니다.
`bl label`을 실행하면 label 위치로 점프하면서, **해당 `BL` 다음 명령어의 주소(복귀 주소)**를 `LR`에 저장합니다.
이후 서브루틴 코드가 끝나면 `LR`에 저장된 주소를 `PC`로 로드하여 원래 실행 흐름으로 돌아갑니다.

<figure style="text-align: center;">
  <img src="/assets/img/bl-with-lr-and-pc.png"
    alt="BL 명령어 수행 시 PC와 LR의 동작 흐름 다이어그램"
    style="display: block; margin: auto;" /> 
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  그럼 1. BL 명령어가 실행되면 PC는 서브루틴으로 이동하고, LR은 복귀할 명령어의 주소를 저장합니다. 
  </figcaption>
</figure>

## LR(Link Register) 레지스터
ARM은 함수 호출을 구현하기 위해 LR을 복귀 주소를 저장하는 전용 레지스터로 사용합니다.
ARMv4에서는 `R14`가 `LR`로 사용됩니다.

```armasm
  bl foo
  mov r0, #2   @ LR이 저장한 명령어 위치
```

서브루틴이 끝나면 `LR`에 저장된 주소로 복귀해야 하므로, `PC`를 `LR` 값으로 업데이트합니다.

```armasm
  mov pc, lr
```

> ARMv4에는 아직 `bx` 명령이 없기 때문에, `mov pc, lr` 형태로 직접 복귀합니다.
> ARMv5 이후부터는 `bx lr`이 일반적으로 사용됩니다.

## 예제 코드
```armasm
  .text
  .global _start
_start:
  mov r0, #2
  bl foo
  mov r1, r0
  b .
foo:
  add r0, r0, #3
  mov pc, lr
```

### 실행 흐름
1. `mov r0, #2` → R0 = 2
2. `bl foo` → PC = foo, LR = `mov r1, r0`의 주소
3. `add r0, r0, #3` → R0 = 5
4. `mov pc, lr` → PC가 `mov r1, r0`로 점프
5. `mov r1, r0` 실행

이 예제는 서브루틴 `foo`에서 `R0`에 `3`을 더한 후, 그 값을 `R1`에 저장하는 간단한 흐름입니다.
만약 `mov pc, lr`이 없다면 `R1`에 결과값이 저장되지 못하고 프로그램 흐름이 중단됩니다.

### 디버깅
```bash
(gdb) target remote :1234
(gdb) display/i $pc
(gdb) display/i $lr
```

`bl foo` 직후의 `PC`와 `LR` 값을 비교하면 `BL` 명령이 다음 명령어 주소를 `LR`에 저장한다는 점을 명확히 확인할 수 있습니다.
특히 `LR` 값이 `mov r1, r0`의 주소와 일치하는지 확인하면 `BL`의 동작을 직관적으로 이해할 수 있습니다.

## 마무리
다음 포스팅에서는 조건부 명령어 실행을 위한 condition code에 대해 알아보겠습니다.
지금까지는 모든 명령어가 항상 실행되었지만,
ARM 명령어는 뒤에 두 글자의 조건 코드를 붙여 특정 조건에서만 실행되도록 만들 수 있습니다.