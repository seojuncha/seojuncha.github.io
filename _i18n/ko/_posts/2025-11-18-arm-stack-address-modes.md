---
layout: post
lang: ko 
ref: "arm-stack-address-modes"
title: "ARM Assembly #12 - 스택메모리와 주소모드"
date: 2025-11-17 00:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["arm ldm", "arm stm", "addressing modes", "stack memory"]
published: false
---

LDM과 STM의 다양한 주소 모드를 스택 메모리의 PUSH/POP 동작에서는 어떻게 활용할 수 있을지 살펴보겠습니다.  

앞선 포스팅에서는 `LDM`/`STM`을 이용해 여러 워드를 한 번에 읽고 쓰는 방법까지만 다뤘습니다. 
이번 글에서는 그 연장선에서, 주소 모드 개념과 스택 구조를 연결해서 정리해보겠습니다.  

## 주소 모드란?  
앞선 포스팅에서 사용한 예제들은 모두 `LDM`/`STM` 실행 시 **메모리 주소가 아래에서 위로(증가하는 방향)** 으로만 움직였습니다.  
하지만 실제로는 상황에 따라 **메모리 주소가 증가할 수도 있고, 감소할 수도 있습니다.**  
또한 베이스 레지스터(`Rn`)가 가리키는 주소를 **포함한 위치에서 시작할지**, 아니면 **그 다음 주소부터 시작할지**도 제어할 수 있습니다.  

이렇게 메모리의 진행 방향과 시작 위치를 정의하는 것이 바로 ARM의 **주소 모드(Addressing Mode)** 입니다.  


## 4개의 주소 모드: IA, IB, DA, DB  
ARM의 `LDM`/`STM` 명령은 다음 네 가지 주소 모드를 제공합니다.  

| 주소 모드 | 설명 | 
|-|-|
| IA | Increment After (증가 후) | 
| IB | Increment Before (증가 전)| 
| DA | Decrement After (감소 후) | 
| DB | Decrement Before (감소 전)| 

**Increment / Decrement** 는 주소가 **위로 증가하는지, 아래로 감소하는지**를 의미합니다.  
**Before / After** 는 베이스 레지스터(`Rn`)가 가리키는 주소를 **메모리 블럭에 포함할지 말지를** 결정합니다.  


## IA (Increment After, 기본값)  
- 증가(Increment): 낮은 주소에서 높은 주소로 진행합니다.  
- 후(After): 베이스 주소(`Rn`)를 포함해 그 위치부터 시작합니다.  

#### 참조 주소 계산  
- 시작 주소 = `Rn`  
- 끝 주소 = `Rn + (레지스터 수 × 4) − 4`  
- 실행 후 `Rn = Rn + (레지스터 수 × 4)`  

예를 들어, 레지스터 목록 개수가 3개이고 `Rn = 0x1000`이라면 다음과 같습니다.  

```text
0x1008  : end address
0x1004
0x1000  : start address

Rn = 0x100C
```
<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png"
    alt="IA 주소 모드 메모리 배치"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    IA 주소 모드는 베이스 레지스터가 가리키는 주소에서 시작해, 위쪽으로 주소를 증가시키면서 연속된 워드를 참조합니다.
    </figcaption>
</figure>


## IB (Increment Before)
- 증가(Increment): 낮은 주소에서 높은 주소로 진행합니다.
- 전(Before): 베이스 주소(Rn) 바로 다음 주소부터 시작합니다.

#### 참조 주소 계산
- 시작 주소 = `Rn + 4`
- 끝 주소 = `Rn + (레지스터 수 × 4)`
- 실행 후 `Rn = Rn + (레지스터 수 × 4)`

예를 들어, 레지스터 목록 개수가 3개이고 `Rn = 0x1000`이라면 다음과 같습니다.
```
0x100C  : end address
0x1008
0x1004  : start address
0x1000

Rn = 0x100C
```

<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png"
    alt="IB 주소 모드 메모리 배치"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    IB 주소 모드는 베이스 주소 바로 다음 위치에서 시작해, 위쪽으로 주소를 증가시키며 워드를 참조합니다.
    </figcaption>
</figure>

## DA (Decrement After)
- 감소(Decrement): 높은 주소에서 낮은 주소로 진행합니다.
- 후(After): 베이스 주소(Rn)를 포함해 그 위치까지를 범위로 사용합니다.

#### 참조 주소 계산
- 시작 주소 = `Rn − (레지스터 수 × 4) + 4`
- 끝 주소 = `Rn`
- 실행 후 `Rn = Rn − (레지스터 수 × 4)`

예를 들어, 레지스터 목록 개수가 3개이고 `Rn = 0x1000`이라면 다음과 같습니다.
```
0x1000  : end address
0x0FFC
0x0FF8  : start address
0x0FF4
0x0FF0

Rn = 0x0FF4
```

<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png"
    alt="DA 주소 모드 메모리 배치"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    DA 주소 모드는 베이스 주소를 포함한 위쪽 주소에서 시작해, 아래 방향으로 연속된 워드를 참조합니다.
    </figcaption>
</figure>

## DB (Decrement Before)
- 감소(Decrement): 높은 주소에서 낮은 주소로 진행합니다.
- 전(Before): 베이스 주소(Rn) 바로 아래 주소부터 시작합니다.

#### 참조 주소 계산
- 시작 주소 = `Rn − (레지스터 수 × 4)`
- 끝 주소 = `Rn − 4`
- 실행 후 `Rn = Rn − (레지스터 수 × 4)`

예를 들어, 레지스터 목록 개수가 3개이고 `Rn = 0x1000`이라면 다음과 같습니다.
```
0x1000
0x0FFC  : end address
0x0FF8
0x0FF4  : start address

Rn = 0x0FF4
```

<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png"
    alt="DB 주소 모드 메모리 배치"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    DB 주소 모드는 베이스 주소 아래에서 시작해, 아래 방향으로 연속된 워드를 참조하는 감소 스택 구조에 자주 사용됩니다.
    </figcaption>
</figure>

## 실전 문법에서 주소 모드는 이렇게 표기합니다
주소 모드를 개념적으로 이해하는 것도 중요하지만,
실제로 ARM 어셈블리를 작성할 때는 주소 모드를 명령어 뒤에 접미사(suffix) 형태로 붙여서 사용합니다.

예를 들어 IA, IB, DA, DB는 다음과 같이 표기됩니다:
```armasm
  ldmia r0!, {r1-r3}
  ldmdb r0!, {r1-r3}
  stmia r0!, {r1-r3}
  stmdb r0!, {r1-r3}
```
- LDM/STM 뒤에 바로 주소 모드 두 글자(IA/IB/DA/DB) 를 붙입니다.

즉, 띄어쓰기나 점(.)을 사용하지 않고 명령어 이름 자체가 LDMIA, LDMDB 같은 형태가 됩니다.

> 자동 갱신(`!`)은 주소 모드와는 별개입니다!  
> 여기서
> ```armasm
>   ldmia r0!, {r1-r3}
> ```
> - IA는 주소 모드 (메모리가 어떤 방향으로 접근되는지)  
> - **!**는 베이스 레지스터(R0)를 자동으로 증가/감소시키는 기능입니다.

## 동일 주소 모드에서의 STM과 LDM
이제 일반 주소 모드에서 STM과 LDM을 같은 모드로 사용했을 때, 메모리와 레지스터 사이의 데이터 이동을 살펴보겠습니다.

```armasm 
  mov r0, #0x8000
  mov r1, #0x1
  mov r2, #0x2
  mov r3, #0x3

  stmia r0, {r1-r3}

  mov r1, #0
  mov r2, #0
  mov r3, #0

  ldmia r0, {r1-r3}
```

이 경우에는 `STMIA`와 `LDMIA`가 동일한 주소 모드를 사용하므로,
같은 메모리 블럭에 데이터를 저장하고 그 자리에서 그대로 다시 읽어오게 됩니다.

<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png"
    alt="동일 주소 모드를 사용하는 STM과 LDM 메모리 접근"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    동일한 주소 모드를 사용하면 STM과 LDM은 같은 메모리 영역을 기준으로 데이터를 저장하고 다시 불러옵니다.
    </figcaption>
</figure>


하지만 스택 메모리는 이렇게 동작하지 않습니다.
스택에서는 PUSH와 POP이 서로 반대 방향으로 진행되기 때문에, 같은 주소 모드를 그대로 사용할 수 없습니다.

<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png"
    alt="FD 스택에서 PUSH와 POP의 반대 방향 주소 접근"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    FD 스택에서 PUSH는 주소가 감소하는 방향으로, POP은 주소가 증가하는 방향으로 진행합니다.
    </figcaption>
</figure>

## 스택용 주소 모드: FD, FA, ED, EA
ARM은 스택을 다루기 위해 FD, FA, ED, EA 라는 스택 전용 주소 모드 이름을 추가로 제공합니다.
이 모드들은 새로운 동작이 아니라, 앞에서 본 IA/IB/DA/DB를 스택 관점에서 다시 이름 붙인 별칭 입니다.

| 스택 모드 | 의미 |
| - | - |
| FD | Full Descending |
| FA | Full Ascending |
| ED | Empty Descending |
| EA | Empty Ascending |

**Full / Empty**는 PUSH(STM), POP(LDM) 이후에 스택 포인터(SP)가 가리키는 위치가 데이터 위인지, 비어 있는 위치인지를 의미합니다.  
**Ascending / Descending**은 PUSH 했을 때 스택이 주소가 증가하는 방향으로 자라는지(Ascending), 감소하는 방향으로 자라는지(Descending) 를 의미합니다.

### 예시: FD 스택의 PUSH
ARM에서 가장 일반적으로 사용하는 스택 구조는 Full Descending( FD ) 스택입니다.
FD 스택에서 PUSH는 다음과 같이 구현할 수 있습니다.

```armasm
  mov sp, #0x8000
  mov r1, #0x1
  mov r2, #0x2
  mov r3, #0x3

  stmdb sp!, {r1-r3}
```

여기서 `stmdb sp!, {r1-r3}` 는 스택 모드 이름으로는 `stmfd sp!, {r1-r3}`와 동일한 동작을 합니다.
SP는 `0x8000`에서 `0x7FF4`로 이동합니다.
실제 메모리에 저장되는 위치는 다음과 같습니다.

| 주소 | 값(레지스터) |
| - | - |
|  `0x7FF4` | `0x1`(R1) |
|  `0x7FF8` | `0x2`(R1) |
|  `0x7FFC` | `0x3`(R1) |

<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png"
    alt="FD 스택에서 STMDB에 의한 PUSH 메모리 배치"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    FD 스택에서 STMDB(SMFD)는 SP를 위쪽 주소에서 아래 방향으로 이동시키며, 새로운 데이터는 더 낮은 주소에 저장됩니다.
    </figcaption>
</figure>

### 예시: FD 스택의 POP
이번에는 FD 스택에 저장된 값을 POP으로 되돌려 받는 동작을 살펴보겠습니다.

```armasm
  @ 앞에서 STMDB sp!, {r1-r3}가 실행되어 있다고 가정합니다.

  ldmia sp!, {r1-r3}
```

FD 스택에서 POP 동작은 일반 주소 모드로는 `ldmia sp!`, 스택 주소 모드 이름으로는 `ldmfd sp!`와 같습니다.
로드 순서는 다음과 같습니다.

| 주소 | 레지스터 |
| - | - |
|  `0x7FF4` | `R1` |
|  `0x7FF8` | `R2` |
|  `0x7FFC` | `R3` |

POP 이후에는 SP가 다시 `0x8000`으로 되돌아옵니다.

<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png"
    alt="FD 스택에서 LDMIA에 의한 POP 메모리 배치"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    FD 스택에서 LDMIA(LDMFD)는 가장 낮은 주소부터 순서대로 레지스터에 값을 불러오며, SP를 다시 위쪽 주소로 이동시킵니다.
    </figcaption>
</figure>

이처럼 FD 스택에서는 PUSH에 `STMDB`, POP에 `LDMIA` 를 사용해야, 같은 스택 메모리 블럭을 기준으로 올바르게 동작합니다.

## 일반 주소 모드와 스택 주소 모드 비교
앞에서 살펴본 것처럼, 스택 주소 모드는 결국 IA/IB/DA/DB에 붙여진 별칭입니다.

### STM 기준 (PUSH)

| 일반 주소모드 | 스택 주소모드 |
|--|--|
| IA | EA |
| IB | FA |
| DA | ED |
| DB | FD |

### LDM 기준 (POP)

| 일반 주소모드 | 스택 주소모드 |
|--|--|
| IA | FD |
| IB | ED |
| DA | FA |
| DB | EA |


실제로 ARM 코드에서는 `STMFD`, `LDMFD`처럼 스택용 주소 모드를 직접 쓰기도 하고,
직접 `STMDB sp!`, `LDMIA sp!` 형태로 일반 주소 모드를 사용하는 경우도 많습니다.

어느 쪽을 사용하든 동작은 동일하므로, 튜토리얼 단계에서는 일반 주소 모드에 익숙해지는 것이 더 중요합니다.

## 마무리
이번 포스팅에서는 `LDM`과 `STM`에서 제공하는 주소 모드가 어떤 의미를 가지는지,
그리고 이 주소 모드가 스택 메모리의 PUSH/POP 동작과 어떻게 연결되는지를 살펴봤습니다.

주소 모드는 단순히 문법 옵션이 아니라, 메모리의 방향과 범위를 정밀하게 제어하기 위한 도구입니다.
이를 이해하면, 스택뿐만 아니라 다양한 메모리 블럭 처리에서도 `LDM`/`STM`을 훨씬 더 유연하게 활용하실 수 있습니다.

다음 포스팅에서는 실행 흐름을 제어하기 위한 분기 명령어 `B`를 살펴보겠습니다.