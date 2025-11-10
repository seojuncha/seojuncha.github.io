---
layout: post
lang: ko
ref: "arm-memory-block-access"
title: "ARM 어셈블리 #11 - 메모리 블럭 접근(LDM, STM)"
date: 2025-11-11 00:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["memory block", "arm ldm", "arm stm"]
published: false
---

이번 글에서는 메모리에 저장된 여러 개의 워드를 **한 번에 읽거나 저장**할 수 있는 ARM 명령어 `LDM`과 `STM`에 대해 알아보겠습니다.  

이전 포스팅에서 살펴본 `LDR`과 `STR` 명령은 한 번에 하나의 워드만 다룰 수 있었습니다.  
하지만 배열이나 구조체처럼 **연속된 메모리 영역**을 다룰 때, 매번 `LDR`이나 `STR`을 반복해서 사용하는 것은 비효율적입니다.  
이러한 문제를 해결하기 위해 ARM은 메모리 블럭 단위로 데이터를 읽고 쓸 수 있는 `LDM`(Load Multiple)과 `STM`(Store Multiple) 명령을 제공합니다.  

## 메모리 블럭
ARM 프로그래밍에서는 배열이나 구조체처럼 **연속된 데이터를 한꺼번에 다루는 상황**이 자주 발생합니다.  
메모리 내부에서는 이러한 데이터들이 여러 개의 워드 단위로 연속해서 저장되어 있습니다.  
이런 연속된 영역을 **메모리 블럭(memory block)** 이라고 부릅니다.  

<figure style="text-align: center;">
  <img src="/assets/img/memory-block.png" alt="연속된 워드로 구성된 메모리 블럭" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    메모리 블럭은 연속된 워드들로 구성되어 있으며, 각 워드는 순차적으로 증가하는 주소를 가집니다.  
  </figcaption>
</figure>

메모리는 실제로는 선형 구조이지만, 이해를 돕기 위해 2차원 배열 형태로 표현하면 좀 더 직관적으로 볼 수 있습니다.  
각 칸은 1워드(4바이트) 크기의 저장공간을 의미합니다.  

[이전 포스팅]({% post_url 2025-10-31-arm-ldr-str-basic %})에서 설명드린 것처럼, 메모리는 16진수 주소를 사용해 한 워드 단위로 접근할 수 있습니다.  

 
### LDR/STR 사용 시의 비효율성
`STR`과 `LDR` 같은 단일 주소 접근 명령은 한 번에 하나의 워드만 읽거나 쓸 수 있습니다.  

```armasm
  mov r0, #0x8000
  mov r1, #0x1
  str r1, [r0]
```
<figure style="text-align: center;">
  <img src="/assets/img/ldr-str-single-access.png" alt="단일 워드 접근 구조" style="display: block; margin: auto;" /> 
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  LDR/STR 명령은 메모리와 레지스터 간의 1:1 관계로 데이터를 전송합니다.  
  </figcaption>
</figure>

즉, Mem[address]와 Register의 관계는 워드 단위의 1:1 관계입니다.
주소를 오프셋(offset) 방식으로 조작할 수는 있지만, 여러 워드를 다루려면 결국 `LDR`과 `STR` 명령을 반복해야 합니다.
배열이나 구조체처럼 연속된 데이터를 다룰 때 이런 반복은 비효율적입니다.

 
## LDM과 STM
ARM은 이러한 연속된 메모리 블럭을 효율적으로 다루기 위해 `LDM`과 `STM` 명령을 제공합니다.

- `LDM`: 메모리 블럭에서 여러 레지스터로 데이터를 불러옵니다.
- `STM`: 여러 레지스터의 값을 메모리 블럭에 저장합니다.

```
  ldm Rn, {registers}
  stm Rn, {registers}
```

여기서,  
`Rn`은 접근할 메모리의 **베이스 주소(base address)** 입니다.
`{registers}`는 `,`로 구분된 레지스터 목록으로, 연속된 레지스터는 `-`를 사용해 표현합니다.

> `Rn`을 시작 주소(start address)가 *아니라* 베이스 주소(base address) 라고 부릅니다.
> 실제 시작 주소는 이후에 설명드릴 주소 모드(addressing mode) 에 따라 달라집니다.

### 레지스터 목록 다루기
다음 세 가지 예시를 통해 여러 레지스터 목록을 어떻게 사용하는지 살펴보겠습니다.

> 아래 예시에서는 `LDM`만 사용했지만, `STM` 명령도 동일한 방식으로 동작합니다.  
> 단지 `STM`은 반대 방향으로, 레지스터의 값을 메모리에 저장합니다.

#### 예시 1) 0x8000 위치의 메모리에서 2개의 워드를 R1, R2에 저장할 때
```armasm
  mov r0, #0x8000
  ldm r0, {r1, r2}
```
- `R1` ← Mem[`0x8000`] 
- `R2` ← Mem[`0x8004`] 
 
<figure style="text-align: center;">
  <img src="/assets/img/ldm-comma-seperated-registers.png" alt="LDM 명령의 쉼표 구분 레지스터 목록" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  `LDM r0, {r1, r2}` 명령은 연속된 두 워드를 각각 R1과 R2에 불러옵니다.
  </figcaption>
</figure>


#### 예시 2) 0x8000 위치의 메모리에서 4개의 워드를 R1~R4에 저장할 때
```armasm
  mov r0, #0x8000
  ldm r0, {r1 - r4}
```
- `R1` ← Mem[`0x8000`] 
- `R2` ← Mem[`0x8004`] 
- `R3` ← Mem[`0x8008`] 
- `R4` ← Mem[`0x800C`] 

<figure style="text-align: center;">
  <img src="/assets/img/ldm-range-registers.png" alt="LDM 명령의 연속 레지스터 목록" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
   `LDM r0, {r1-r4}` 명령은 4개의 연속된 워드를 메모리에서 읽어 R1부터 R4까지 순서대로 저장합니다. 
  </figcaption>
</figure>


#### 예시 3) 0x8000 위치의 메모리에서 R1, R2, R7에 저장할 때 
```armasm
  mov r0, #0x8000
  ldm r0, {r1 - r2, r7}
```
- `R1` ← Mem[`0x8000`] 
- `R2` ← Mem[`0x8004`] 
- `R7` ← Mem[`0x8008`] 

<figure style="text-align: center;">
  <img src="/assets/img/ldm-range-comma-seperated-registers.png" alt="LDM 명령의 복합 레지스터 목록" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
   `LDM r0, {r1-r2, r7}` 명령은 비연속적인 레지스터 목록을 지정해 사용할 수 있습니다. 
  </figcaption>
</figure>
 
### 메모리 주소 자동 업데이트
```
  ldm Rn!, {registers}
  stm Rn!, {registers}
```
베이스 레지스터 `Rn` 뒤에 `!`를 붙이면, 명령 실행 후 **자동으로 다음 주소로 갱신**됩니다.
 
```armasm
  mov r0, #0x8000
  mov r1, #0x1
  mov r2, #0x2
  mov r3, #0x3
 
  stm r0!, {r1 - r3}
  @ r0 = r0 + (3 * 4) = 0x8000 + 0xC = 0x800C
```
즉, 한 번의 명령으로 여러 데이터를 저장하고 다음 메모리 위치로 자동 이동할 수 있습니다.

---
지금까지 살펴본 예제에서는 메모리 주소가 항상 **아래에서 위로(증가 방향)** 으로 갱신되었습니다.  
하지만 실제로는 상황에 따라 **주소가 증가할 수도, 감소할 수도 있으며**, 베이스 레지스터(`Rn`)가 갱신되기 전에 접근할지, 후에 접근할지도 결정할 수 있습니다.  
이러한 동작 방식을 제어하는 것이 바로 **주소 모드(Addressing Mode)** 입니다.  

ARM의 `LDM`과 `STM` 명령은 다음 네 가지 주소 모드를 지원합니다.  
 
### 4개의 주소모드:IA, IB, DA, DB

| 주소 모드 | 설명 |
|--|--|
| IA | Icrement After |
| IB | Icrement Before |
| DA | Decrement After |
| DB | Decrement Before |

<figure style="text-align: center;">
  <img src="/assets/img/multi-memory-inst-addr-mode.png" alt="LDM/STM 주소 모드 개요" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  LDM/STM 명령은 IA, IB, DA, DB의 네 가지 주소 모드를 지원하며, 각 모드는 주소 증가 및 시작 위치가 다릅니다.
  </figcaption>
</figure>

참고:
- Increment / Decrement는 U-bit에 의해 결정됩니다.
  - Increment (U=1): 아래에서 위로 (low → high address)
  - Decrement (U=0): 위에서 아래로 (high → low address)
- After / Before는 P-bit에 의해 결정됩니다.
  - After (P=0): Rn을 포함 (included Rn)
  - Before (P=1): Rn을 제외 (excluded Rn)

> [스택 포스팅]({% post_url 2025-11-06-arm-stack-memory %})의 그림처럼,
> 메모리의 높은 주소가 위에, 낮은 주소가 아래에 있다고 생각하면 이해하기 쉽습니다.
 
#### IA (default):
- start_address = Rn
- end_addrses = Rn + (# of registers * 4) - 4
- Rn = Rn + (# of registers * 4)

```
<bottom & included>
# of registers = 3, Rn = 0x1000
 
0x1008  : end address
0x1004
0x1000  : start address
 
Rn = 0x100C
```

<figure style="text-align: center;">
  <img src="/assets/img/ia-addr-mode-in-memory.png" alt="IA 주소 모드 메모리 배치" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  IA(Increment After) 모드는 베이스 주소부터 시작해 레지스터 개수에 따라 주소가 증가합니다.
  </figcaption>
</figure>
 
#### IB:
- start_address = Rn + 4
- end_address = Rn + (# of registers * 4)
- Rn = Rn + (# of registers * 4)

```
<bottom & excluded>
# of registers = 3, Rn = 0x1000
 
0x100C  : end address
0x1008
0x1004  : start address
0x1000
 
Rn = 0x100C
```

<figure style="text-align: center;">
  <img src="/assets/img/ib-addr-mode-in-memory.png" alt="IB 주소 모드 메모리 배치" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  IB(Increment Before) 모드는 베이스 주소 다음 주소부터 시작하여 주소가 증가합니다.  
  </figcaption>
</figure>
 
#### DA:
- start_address = Rn - (# of registers * 4) + 4
- end_address = Rn
- Rn = Rn - (# of registers * 4)

```
<top & included>
# of registers = 3, Rn = 0x1000
 
0x1000  : end address
0x0FFC
0x0FF8  : start address
0x0FF4
 
Rn = 0xFFF4
```

<figure style="text-align: center;">
  <img src="/assets/img/da-addr-mode-in-memory.png" alt="DA 주소 모드 메모리 배치" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  DA(Decrement After) 모드는 상단 주소부터 시작하여 주소가 감소합니다.
  </figcaption>
</figure>
 
#### DB:
- start_address = Rn - (# of registers * 4)
- end_address = Rn - 4
- Rn = Rn - (# of registers * 4)

```
<top & excluded>
# of registers = 3, Rn = 0x1000

0x1000
0x0FFC  : end address
0x0FF8
0x0FF4  : start address
 
Rn = 0xFFF4
```

<figure style="text-align: center;">
  <img src="/assets/img/db-addr-mode-in-memory.png" alt="DB 주소 모드 메모리 배치" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  DB(Decrement Before) 모드는 베이스 주소 이전의 주소부터 시작하여 주소가 감소합니다.
  </figcaption>
</figure>
 
## 마무리
이번 글에서는 `LDM`과 `STM`을 이용해 여러 워드를 한 번에 다루는 방법을 살펴보았습니다. 
다음 포스팅에서는 스택 메모리에서 `LDM`과 `STM`이 어떻게 사용되는지를 함께 알아보겠습니다. 
스택은 여러 레지스터를 한꺼번에 저장(push)하거나 복원(pop)해야 하기 때문에 이 두 명령이 특히 유용합니다.