---
layout: post
lang: ko
ref: "arm-cpsr-condition-flags"
title: "ARM 어셈블리 #15 - CPSR과 조건 플래그를 활용한 조건부 명령 실행"
date: 2025-11-27 20:20:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["condition flags", "arm cpsr"]
---

지금까지는 명령어가 실행될 순서가 되면 무조건 실행되는 구조만 살펴봤습니다.
하지만 ARM 아키텍처는 조금 더 특별합니다.
`CPSR` 레지스터의 조건 플래그를 사용하면 특정 조건에서만 명령어를 실행하도록 만들 수 있습니다.

이 기능은 ARM의 핵심 특징 중 하나이며, 모든 명령어에 조건을 붙여 분기 없이도 조건문을 구현할 수 있다는 점이 큰 장점입니다.
또한, 불필요한 분기를 줄여 파이프라인 성능 향상에도 도움이 됩니다.


## CPSR (Current Program Status Register) 레지스터
`CPSR` 레지스터는 일반적인 범용 레지스터(`R0`~`R15`)와 달리,
CPU의 프로그램 상태를 저장하는 특수 목적 레지스터입니다.

CPU는 연산 결과에 따라 아래의 4가지 상태를 가질 수 있으며,
이 값들이 `CPSR`의 상위 4비트에 각각 1비트씩 기록됩니다.

- 음수
- 0 혹은 양수
- 캐리
- 오버플로우

| 비트 위치 | 플래그          | 의미    |
| ----- | ------------ | ----- |
| 31    | N (Negative) | 음수    |
| 30    | Z (Zero)     | 결과가 0 |
| 29    | C (Carry)    | 캐리    |
| 28    | V (Overflow) | 오버플로우 |

### 플래그를 업데이트하려면?
연산 결과를 플래그에 반영하려면 명령어 뒤에 s를 붙여야 합니다.  
예: `adds`, `subs`, `movs` 등,
이 내용은 [이전 포스팅]({% post_url 2025-09-10-arm-logical-shift %})에서 다뤘습니다.

## 조건 플래그 (Condition Flags)
ARM의 거의 모든 명령어는 `CPSR` 플래그 값을 기준으로 명령어를 실행하거나 건너뛸 수 있습니다.
이를 위해 명령어 뒤에 두 자리 조건 코드(`EQ`, `NE`, `MI`, `PL` 등)를 붙여 사용합니다.

<figure style="text-align: center;"> 
  <img src="/assets/img/condition-code-table-in-arm-document.png" 
    alt="ARM condition code table" 
    style="display: block; margin: auto;" /> 
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;"> 
  그림 1. ARM 아키텍처에서 제공하는 조건 코드 목록
  </figcaption> 
</figure>

> ARM은 총 15개의 조건 코드를 제공합니다.  
> 지금까지 사용한 명령어들은 모두 기본값인 `AL`(Always) 조건이 적용된 형태로, 생략된 것입니다.  
> `AL`은 항상 실행된다는 의미입니다.

## 명령어 + 조건
```
  {instruction}{condition}
```

예를 들어, 이전 연산 결과가 음수일 때만 `mov`를 실행하고 싶다면:

```armasm
  movmi r0, r1
```

- `MI`(Minus) = `N == 1`

즉, 결과가 음수일 때만 실행됩니다.

## 예제 코드
```armasm
  .text
  .global _start
_start:
  movs r0, #-1
  addmi r0, r0, #5
  addpl r0, r0, #1
  b .
```

위 예제는 `R0`에 `-1`을 저장한 뒤:

- 음수(`N==1`)이면 `5`를 더하고
- 양수 또는 0(`N==0`)이면 `1`을 더하는
동작을 수행합니다.

> 플래그 업데이트를 위해 `movs`를 사용했습니다.

여기서
- `MI`: `N == 1` → 음수일 때 실행
- `PL`: `N == 0` → 양수 또는 0일 때 실행

따라서 C 코드로 표현하면 다음과 같습니다.
```c
  int r0 = -1;
  r0 += (r0 < 0) ? 5 : 1;
```

## 디버깅
GDB에서 `CPSR` 전체를 확인하려면:
```bash
(gdb) p/x $cpsr
```
각 플래그는 다음과 같이 추출할 수 있습니다.

```bash
(gdb) p ($cpsr >> 31) & 1   # N flag
(gdb) p ($cpsr >> 30) & 1   # Z flag
(gdb) p ($cpsr >> 29) & 1   # C flag
(gdb) p ($cpsr >> 28) & 1   # V flag
```

## 마무리
이번 글에서는 `CPSR` 플래그와 조건 코드를 이용한 조건부 명령 실행을 살펴봤습니다.
다음 글에서는 비교 연산자(`CMP`, `CMN` 등)를 이용한 실제 조건 분기를 다루고,
조건부 실행과 조건 분기를 어떻게 선택해야 하는지도 함께 설명드리겠습니다.