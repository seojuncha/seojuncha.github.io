---
layout: post
lang: ko 
ref: "stack-memory-and-addressing-modes"
title: "ARM Assembly #12 - 스택메모리와 주소모드"
date: 2025-11-11 00:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["memory block", "arm ldm", "arm stm", "addressing modes", "stack memory"]
published: false
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
