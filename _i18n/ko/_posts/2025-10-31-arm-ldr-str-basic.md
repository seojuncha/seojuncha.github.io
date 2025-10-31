---
layout: post
lang: ko
ref: "arm-ldr-str-basic"
title: "ARM 어셈블리 #7 - LDR과 STR로 메모리에 접근하는 가장 단순한 방법"
date: 2025-10-31 20:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["ldr", "str", "memory-access", "armv4"]
---

이 글은 ARM 어셈블리를 처음 배우는 사람을 위한 **메모리 접근 입문 튜토리얼**입니다.
ARMv4 아키텍처 기준으로 `ldr`(load)과 `str`(store) 명령어를 사용하여 CPU가 메모리의 데이터를 읽고 쓰는 가장 기본적인 방식을 다룹니다.

특히 아래 내용을 순서대로 학습할 수 있습니다.
- 왜 연산 전에 메모리에서 레지스터로 값을 읽어와야 하는지
- `ldr`과 `str` 명령어의 구조와 역할
- 대괄호(`[]`)가 의미하는 주소 접근 방식
- **QEMU**와 **GDB**를 사용해 레지스터와 메모리 상태를 직접 확인하는 방법

> **핵심 포인트:**  
> `ldr`과 `str`은 모든 ARM 프로그램의 기본이 되는 레지스터–메모리 데이터 이동의 출발점입니다.
> 이 글을 통해 “CPU가 실제로 데이터를 어떻게 읽고 저장하는가?”를 직접 이해할 수 있습니다.


<figure>
<svg role="img" aria-labelledby="title desc" viewBox="0 0 800 240" width="100%" xmlns="http://www.w3.org/2000/svg">

  <title id="title">CPU ↔ Memory data flow with LDR/STR</title>
  <desc id="desc">Shows LDR reading from Memory to Register and STR writing from Register to Memory.</desc>
  <style>
    .fg { stroke:#E6E6E6; fill:#E6E6E6; }
    .box { fill:#0B1220; stroke:#A3B1C6; }
    .wire { stroke:#E6E6E6; }
    .muted { fill:#C8D0DB; }
    .marker path { fill:#E6E6E6; }  
    .label{font:16px sans-serif}
    .note{font:15px sans-serif}
    .mono{font:14px monospace}
  </style>
  <defs>
    <marker class="marker" id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" />
    </marker>
  </defs>

  <!-- CPU Registers -->
  <rect class="box" x="60" y="50" width="280" height="140" rx="8" />
  <text class="label fg" x="200" y="80" text-anchor="middle">CPU Registers</text>
  <!-- little register slots -->
  <g transform="translate(90,100)">
    <rect class="box" x="0" y="0" width="50" height="24" rx="4" />
    <rect class="box" x="60" y="0" width="50" height="24" rx="4" />
    <rect class="box" x="120" y="0" width="50" height="24" rx="4" />
    <rect class="box" x="180" y="0" width="50" height="24" rx="4" />
    <text class="note fg" x="25" y="17" text-anchor="middle">r0</text>
    <text class="note fg" x="85" y="17" text-anchor="middle">r1</text>
    <text class="note fg" x="145" y="17" text-anchor="middle">r2</text>
    <text class="note fg" x="205" y="17" text-anchor="middle">…</text>
  </g>

  <!-- Memory -->
  <rect class="box" x="460" y="30" width="280" height="180" rx="8" />
  <text class="label fg" x="600" y="60" text-anchor="middle">Memory</text>
  <g transform="translate(490,80)">
    <rect class="box" x="0" y="0" width="220" height="28" />
    <rect class="box" x="0" y="36" width="220" height="28" />
    <rect class="box" x="0" y="72" width="220" height="28" />
    <text class="note fg" x="8" y="19">0x1000</text>
    <text class="note fg" x="8" y="55">0x1004</text>
    <text class="note fg" x="8" y="91">0x1008</text>
  </g>

  <!-- Arrows -->
  <line class="wire" x1="340" y1="120" x2="460" y2="120" stroke="#000" stroke-width="2" marker-end="url(#arrow)"/>
  <text class="label fg" x="400" y="110" text-anchor="middle">STR</text>
  <line class="wire" x1="460" y1="160" x2="340" y2="160" stroke="#000" stroke-width="2" marker-end="url(#arrow)"/>
  <text class="label fg" x="400" y="150" text-anchor="middle">LDR</text>
</svg>
<figcaption>
LDR은 메모리 → 레지스터, STR은 레지스터 → 메모리로 데이터가 이동한다.
</figcaption>
</figure>


## 메모리에 접근해야 하는 이유

ARM CPU의 데이터 연산 명령어(`mov`, `add` 등)은 메모리에 직접 접근하지 않고 **항상 레지스터에 저장된 값으로만 계산**을 수행합니다.
따라서 **메모리에 저장된 값은 연산 전에 반드시 레지스터로 읽어와야 합니다**.

ARMv4에서 제공하는 일반 목적 레지스터는 총 **16**개(`r0` ~ `r15`) 입니다.
하지만 프로그램이 사용하는 모든 데이터를 16개의 레지스터 안에 담는 것은 불가능합니다.
그래서 필요한 데이터만 레지스터로 불러와 연산하고, 그 결과를 다시 메모리에 저장하는 방식으로 동작합니다.

<figure>
<svg role="img" aria-labelledby="title desc" viewBox="0 0 800 260" width="100%" xmlns="http://www.w3.org/2000/svg">

  <title id="title">Registers vs Memory roles</title>
  <desc id="desc">Shows 16 general-purpose registers contrasted with large memory as main storage.</desc>
  <style>
    .fg { stroke:#E6E6E6; fill:#E6E6E6; }
    .box { fill:#0B1220; stroke:#A3B1C6; }
    .wire { stroke:#E6E6E6; }
    .muted { fill:#C8D0DB; }
    .marker path { fill:#E6E6E6; }  
    .label{font:16px sans-serif}
    .note{font:13px sans-serif}
    .mono{font:14px monospace}
  </style>

  <!-- Registers grid -->
  <rect class="box" x="60" y="30" width="300" height="200" rx="8" />
  <text class="label fg" x="210" y="55" text-anchor="middle">Registers (16)</text>
  <g transform="translate(80,75)">
    <!-- 4 x 4 grid -->
    <g class="row" transform="translate(0,0)">
      <rect class="box" x="0" y="0" width="60" height="30" rx="4"/><text class="note fg" x="30" y="20" text-anchor="middle">r0</text>
      <rect class="box" x="70" y="0" width="60" height="30" rx="4"/><text class="note fg" x="100" y="20" text-anchor="middle">r1</text>
      <rect class="box" x="140" y="0" width="60" height="30" rx="4"/><text class="note fg" x="170" y="20" text-anchor="middle">r2</text>
      <rect class="box" x="210" y="0" width="60" height="30" rx="4"/><text class="note fg" x="240" y="20" text-anchor="middle">r3</text>
    </g>
    <g class="row" transform="translate(0,40)">
      <rect class="box" x="0" y="0" width="60" height="30" rx="4"/><text class="note fg" x="30" y="20" text-anchor="middle">r4</text>
      <rect class="box" x="70" y="0" width="60" height="30" rx="4"/><text class="note fg" x="100" y="20" text-anchor="middle">r5</text>
      <rect class="box" x="140" y="0" width="60" height="30" rx="4"/><text class="note fg" x="170" y="20" text-anchor="middle">r6</text>
      <rect class="box" x="210" y="0" width="60" height="30" rx="4"/><text class="note fg" x="240" y="20" text-anchor="middle">r7</text>
    </g>
    <g class="row" transform="translate(0,80)">
      <rect class="box" x="0" y="0" width="60" height="30" rx="4"/><text class="note fg" x="30" y="20" text-anchor="middle">r8</text>
      <rect class="box" x="70" y="0" width="60" height="30" rx="4"/><text class="note fg" x="100" y="20" text-anchor="middle">r9</text>
      <rect class="box" x="140" y="0" width="60" height="30" rx="4"/><text class="note fg" x="170" y="20" text-anchor="middle">r10</text>
      <rect class="box" x="210" y="0" width="60" height="30" rx="4"/><text class="note fg" x="240" y="20" text-anchor="middle">r11</text>
    </g>
    <g class="row" transform="translate(0,120)">
      <rect class="box" x="0" y="0" width="60" height="30" rx="4"/><text class="note fg" x="30" y="20" text-anchor="middle">r12</text>
      <rect class="box" x="70" y="0" width="60" height="30" rx="4"/><text class="note fg" x="100" y="20" text-anchor="middle">r13</text>
      <rect class="box" x="140" y="0" width="60" height="30" rx="4"/><text class="note fg" x="170" y="20" text-anchor="middle">r14</text>
      <rect class="box" x="210" y="0" width="60" height="30" rx="4"/><text class="note fg" x="240" y="20" text-anchor="middle">r15</text>
    </g>
  </g>

  <!-- Memory -->
  <rect class="box" x="440" y="30" width="300" height="200" rx="8" />
  <text class="label fg" x="590" y="55" text-anchor="middle">Main Memory (Data store)</text>
  <text class="note fg" x="590" y="90" text-anchor="middle">Large, holds program data</text>
  <text class="note fg" x="590" y="110" text-anchor="middle">Registers used for compute</text>
</svg>
<figcaption>
레지스터는 연산을 위한 임시 저장소, 메모리는 주 데이터 저장소다.
</figcaption>
</figure>

## 메모리에 접근하는 어셈블리 명령어

메모리에 접근하기 위한 명령어는 다음 두 가지입니다:
- `ldr` : 메모리에서 레지스터로 데이터를 읽어온다.
- `str` : 레지스터에서 메모리로 데이터를 저장한다.

두 명령어의 기본 문법은 다음과 같습니다.
```
ldr Rd, [Rn]
str Rd, [Rn]
```

여기서,
- `Rd` : 데이터를 읽거나 저장할 대상 레지스터(destination register)
- `Rn` : 접근할 메모리 주소(base register)

`ldr`은 메모리 주소 `Rn`에 저장된 값을 레지스터 `Rd`로 읽어오고,
`str`은 레지스터 `Rd`의 값을 메모리 주소 `Rn`이 가리키는 위치에 저장합니다.

### 대괄호와 베이스 레지스터의 의미

대괄호(`[]`) 안의 값은 메모리 주소를 나타내고, `[주소]` 형태는 그 주소에 저장된 값을 의미합니다.

예를 들어,
```
ldr r0, [r1]
```
이 명령은 *" **`r1`이 가리키는 주소의** 메모리 값을 읽어와 `r0`에 저장한다"* 는 뜻입니다.

<figure>
<svg role="img" aria-labelledby="title desc" viewBox="0 0 800 220" width="100%" xmlns="http://www.w3.org/2000/svg">

  <title id="title">Base register addressing</title>

  <desc id="desc">Shows r1 holding an address 0x1000 and [r1] meaning value at memory[0x1000].</desc>

  <style>
    .fg { stroke:#E6E6E6; fill:#E6E6E6; }
    .box { fill:#0B1220; stroke:#A3B1C6; }
    .wire { stroke:#E6E6E6; }
    .muted { fill:#C8D0DB; }
    .marker path { fill:#E6E6E6; }  
    .label{font:16px sans-serif}
    .note{font:15px sans-serif}
    .mono{font:14px monospace}
  </style>

  <defs>
    <marker class="marker" id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" />
    </marker>
  </defs>

  <!-- r1 register -->
  <rect class="box" x="60" y="60" width="240" height="100" rx="8"/>
  <text class="label fg" x="180" y="88" text-anchor="middle">r1</text>
  <text class="label fg" x="180" y="120" text-anchor="middle">0x00001000</text>

  <!-- memory cell -->
  <rect class="box" x="480" y="40" width="240" height="140" rx="8"/>
  <text class="label fg" x="600" y="70" text-anchor="middle">Memory[0x1000]</text>
  <rect class="box" x="510" y="90" width="180" height="40"/>
  <text class="mono fg" x="600" y="115" text-anchor="middle">0x000004D2</text>

  <!-- arrow and labels -->
  <line x1="300" y1="110" x2="480" y2="110" stroke="#6f6c6cff" stroke-width="2" marker-end="url(#arrow)"/>
  <text class="note fg" x="390" y="100" text-anchor="middle">[r1] ≡ *(uint32_t*)0x1000</text>
</svg>
<figcaption>
<code>r1</code>이 주소 0x1000을 담고 있으면, <code>[r1]</code>은 그 주소의 메모리 값이다.
</figcaption>
</figure>

## CPU가 주소를 통해 데이터를 읽고 쓰는 과정

CPU는 베이스 레지스터(`Rn`)에 저장된 값을 주소(address)로 해석합니다.
그 주소가 가리키는 메모리 위치를 찾아, 그 위치의 데이터를 읽거나 씁니다.

즉,
- `ldr`은 주소를 해석해 데이터를 읽는 동작,
- `str`은 주소를 해석해 데이터를 쓰는 동작입니다.

<figure>
<svg role="img" aria-labelledby="title desc" viewBox="0 0 800 240" width="100%" xmlns="http://www.w3.org/2000/svg">

  <title id="title">CPU ↔ Memory data flow with LDR/STR</title>
  <desc id="desc">Shows LDR reading from Memory to Register and STR writing from Register to Memory.</desc>
  <style>
    .fg { stroke:#E6E6E6; fill:#E6E6E6; }
    .box { fill:#0B1220; stroke:#A3B1C6; }
    .wire { stroke:#E6E6E6; }
    .muted { fill:#C8D0DB; }
    .marker path { fill:#E6E6E6; }  
    .label{font:16px sans-serif}
    .note{font:15px sans-serif}
    .mono{font:14px monospace}
  </style>
  <defs>
    <marker class="marker" id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" />
    </marker>
  </defs>

  <!-- CPU Registers -->
  <rect class="box" x="60" y="50" width="280" height="140" rx="8" />
  <text class="label fg" x="200" y="80" text-anchor="middle">CPU Registers</text>
  <!-- little register slots -->
  <g transform="translate(90,100)">
    <rect class="box" x="0" y="0" width="50" height="24" rx="4" />
    <rect class="box" x="60" y="0" width="50" height="24" rx="4" />
    <rect class="box" x="120" y="0" width="50" height="24" rx="4" />
    <rect class="box" x="180" y="0" width="50" height="24" rx="4" />
    <text class="note fg" x="25" y="17" text-anchor="middle">r0</text>
    <text class="note fg" x="85" y="17" text-anchor="middle">r1</text>
    <text class="note fg" x="145" y="17" text-anchor="middle">r2</text>
    <text class="note fg" x="205" y="17" text-anchor="middle">…</text>
  </g>

  <!-- Memory -->
  <rect class="box" x="460" y="30" width="280" height="180" rx="8" />
  <text class="label fg" x="600" y="60" text-anchor="middle">Memory</text>
  <g transform="translate(490,80)">
    <rect class="box" x="0" y="0" width="220" height="28" />
    <rect class="box" x="0" y="36" width="220" height="28" />
    <rect class="box" x="0" y="72" width="220" height="28" />
    <text class="note fg" x="8" y="19">0x1000</text>
    <text class="note fg" x="8" y="55">0x1004</text>
    <text class="note fg" x="8" y="91">0x1008</text>
  </g>

  <!-- Arrows -->
  <line class="wire" x1="340" y1="120" x2="460" y2="120" stroke="#000" stroke-width="2" marker-end="url(#arrow)"/>
  <text class="label fg" x="400" y="110" text-anchor="middle">STR</text>
  <line class="wire" x1="460" y1="160" x2="340" y2="160" stroke="#000" stroke-width="2" marker-end="url(#arrow)"/>
  <text class="label fg" x="400" y="150" text-anchor="middle">LDR</text>
</svg>
<figcaption>
LDR moves data Memory → Register; STR moves data Register → Memory.
</figcaption>
</figure>

## 실습 예제

아래 예제는 `ldr`과 `str`의 기본 동작을 보여줍니다.
**QEMU**의 `versatilepb` 보드와 `gdb-multiarch`를 이용하면, 레지스터와 메모리의 상태를 직접 확인할 수 있습니다.

> 컴파일과 실습방법은 [이전 포스팅]({% post_url 2025-09-08-arm-mov-instruction %})을 참고해주세요.

```armasm
  .text
  .global _start
_start:
  mov r0, #3         @ r0에 10진수 3를 저장
  ldr r1, =0x1000    @ r1에 메모리 주소 0x1000을 저장
  str r0, [r1]       @ [0x1000] = 3
```
이 코드를 실행하면 CPU는 `r1`에 저장된 주소(`0x1000`)를 해석하고, 그 주소의 메모리 위치에 `r0`의 값(`3`)을 저장합니다.

다음은 메모리에서 값을 읽어오는 예제입니다.
```armasm
  .text
  .global _start
_start:
  ldr r1, =0x1000    @ r1에 0x1000 주소를 저장
  ldr r0, [r1]       @ r0에 [0x1000]의 값을 읽어온다
```

만약 메모리 `0x1000`에 앞서 `3`이 저장되어 있다면, 이 명령 실행 후 `r0`에는 `3`이 저장될 것 입니다.

---

이 포스팅에서는 `ldr`과 `str`을 통해 CPU가 메모리에 접근하고 데이터를 주고받는 가장 기본적인 원리를 살펴봤습니다.
다음 포스팅에서는 오프셋(offset)을 이용한 더 유연한 메모리 접근 방식을 다뤄보겠습니다.