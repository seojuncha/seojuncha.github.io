---
layout: post
lang: ko
ref: "arm-memory-block-access"
title: "ARM 어셈블리 #11 - "
date: 2025-11-11 16:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["memory block", "arm ldm", "arm stm"]
published: false
---

메모리 블럭내의 여러개의 워드를 한번에 다루기 위한 ldm과 stm의 기본적인 사용법을 살펴보자.
배열이나 구조체처럼 여러개의 연속된 메모리를 한번에 다루기 위해 사용되는 ldm과 stm을 알아보자.


## 메모리 블럭
ARM 프로그래밍에서 배열이나 구조체 처럼 연속된 데이터를 한꺼번에 다루는 상황은 매우 흔하다.
메모리 안에는 이 데이터들이 여러 워드 단위로 연속되어 저장된다.
이 연속된 영역을 메모리 블럭이라고 한다.

<figure style="text-align: center;">
  <img src="/assets/img/memory-block.png" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  </figcaption>
</figure>

메모리는 선형구조이지만, 2차원 배열형태로 표현하면 직관적이다.
하나의 열은 1워드(4바이트) 저장공간이다.

[여기]({% post_url %})에서 언급한 것 처럼 16진수 메모리 주소를 사용해서 하나의 워드만큼의 메모리 값에 접근할 수 있다.

 
### ldr/str 사용시 비효율성
`str`과 `ldr`같은 단일 주소 접근으로는 하나의 워드를 읽어서 레지스터에 저장하거나, 하나의 레지스터에서 하나의 워드만 메모리에 저장한다.
 
```armasm
  mov r0, #0x8000
  mov r1, #0x1
  str r1, [r0]
```
 
<figure style="text-align: center;">
  <img src="/assets/img/" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  </figcaption>
</figure>
 
즉, Mem[address] <-> register 관계가 워드 단위의 1:1이다.
다양한 offset방식으로 주소는 편리하게 다룰 수 있지만, 여러개의 워드를 다루려면 ldr/str을 반복적으로 사용해야 한다.
하지만, 배열이나 구조체와 같이 연속된 메모리 영역에 저장되는 경우 반복적인 ldr/str 호출은 비효율적이다.
 
## ldm과 stm
ARM은 이런 연속된 메모리 영역에 저장되는 블럭단위의 여러 워드를 효율적으로 다루기 위해 `LDM`과 `STM`을 제공한다.

- `LDM`: 메모리 블럭에서 여러 레지스터로 데이터 불러오기
- `STM`: 여러 레지스터의 값을 메모리 블럭에 저장하기

```
  ldm Rn, {registers}
  stm Rn, {registers}
```
여기서, 
- `Rn`은 접근할 메모리의 베이스 주소
- `{regsters}`는 `,`로 구분된 레지스터 목록. 여러 레지스터를 순차적으로 접근할 때는 `-`를 사용

> `Rn`을 시작주소가 아니라 베이스 주소라고 표현했다.
> 뒤에 살펴보겠지만 메모리 접근 시작주소는 어떤 주소모드를 사용하는지에 따라 다르다.

### 레지스터 목록 다루기
3개의 예시에서 여러 레지스터 목록을 어떻게 다루는지 살펴보겠다.
> 예시는 ldm만 사용했지만, stm도 동일하게 레지스터 목록을 사용할 수 있다.
> 단지 stm은 ldm과 반대로 레지스터 목록의 값들을 메모리에 저장할 뿐이다.

#### 예시 1) `0x8000`위치의 메모리로 부터 2개의 워드를  `R1`과 `R2`에 저장할 때:
```armasm
  mov r0, #0x8000
  ldm r0, {r1, r2}
```
 
<figure style="text-align: center;">
  <img src="/assets/img/ldm-comma-seperated-registers.png" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  </figcaption>
</figure>
 
#### 예시 2) `0x8000`위치의 메모리로 부터 4개의 워드를  `R1`부터 `R4`에 저장할 때:
```armasm
  mov r0, #0x8000
  ldm r0, {r1 - r4}
```
 
<figure style="text-align: center;">
  <img src="/assets/img/ldm-range-registers.png" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  </figcaption>
</figure>
 
#### 예시 3) `0x8000`위치의 메모리로 부터 5개의 워드를 `R1`부터 `R4`, 그리고 R7에 저장할 때:
```armasm
  mov r0, #0x8000
  ldm r0, {r1 - r2, r7}
```
<figure style="text-align: center;">
  <img src="/assets/img/ldm-range-comma-seperated-registers.png" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
  </figcaption>
</figure>
 
### 메모리 주소 자동 업데이트
```
  ldm Rn!, {registers}
  stm Rn!, {registers}
```
메모리 주소를 저장한 베이스 레지스터에 `!`를 추가하면 메모리로부터 값을 읽어오거나 저장한 이후에 베이스 레지스터 값을 갱신해준다.
 
```armasm
  mov r0, #0x8000
  mov r1, #0x1
  mov r2, #0x2
  mov r3, #0x3
 
  stm r0!, {r1 - r3}
  @ r0 = r0 + 레지스터 개수(3) * 4 = 0x8000 + 0xC = 0x800C
```
 
### 4개의 주소모드:IA, IB, DA, DB
| 주소 모드 | 설명 |
|--|--|
| IA | Icrement After |
| IB | Icrement Before |
| DA | Decrement After |
| DB | Decrement Before |

<figure style="text-align: center;">
  <img src="/assets/img/multi-memory-inst-addr-mode.png" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;"></figcaption>
</figure>

 
Increment/Decrement의 기준: (U-bit)
- Increment -> Upward(U=1) : bottom of range, low to high address
- Decrement -> Downward(U=0) : top of range, high to low address
 
After/Before의 기준: (P-bit)
- After (P=0) : included Rn, 시작주소 포함.
- Before (P=1) : excluded Rn. beyond the top(U=0) & below the bottom(U=1), 시작주소 바로 다음부터.

> 스택에 관해 다룬 포스팅의 그림처럼 높은 주소가 위에, 낮은 주소가 아래에 오는 메모리 형태로 생각하면 이해하기 쉽다.
 
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
  <img src="/assets/img/ia-addr-mode-in-memory.png" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;"></figcaption>
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
  <img src="/assets/img/ib-addr-mode-in-memory.png" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;"></figcaption>
</figure>
 
#### DA
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
0x0FF0
 
Rn = 0xFFF4
```
<figure style="text-align: center;">
  <img src="/assets/img/da-addr-mode-in-memory.png" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;"></figcaption>
</figure>
 
#### DB:
- start_address = Rn - (# of registers * 4)
- end_address = Rn - 4
- Rn = Rn - (# of registers * 4)
```
<top & excluded>
# of registers = 3, Rn = 0x1000
 0x1000 0x0FFC  : end address
0x0FF8
0x0FF4  : start address
 
Rn = 0xFFF4
```
<figure style="text-align: center;">
  <img src="/assets/img/db-addr-mode-in-memory.png" alt="" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;"></figcaption>
</figure>
 
## 마무리
다음은 이전 포스팅의 내용과 이번 포스팅의 내용을 종합한 스택메모리에서의 `LDM`과 `STM`을 사용해보고,
스택 메모리용 주소 모드를 알아보겠다. 
스택의 특성상 push/pop으로 여러 레지스터를 한번에 저장하거나 불러와야 하기 때문에 ldm과 stm이 유용하다.