---
layout: post
lang: ko
ref: "arm-offset-addressing"
title: "ARM 어셈블리 #7 - 오프셋(Offset)으로 메모리 주소 계산하기"
date: 2025-11-02 13:40:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["ldr", "str", "offset", "addressing"]
---

메모리에 저장된 여러 데이터를 효율적으로 읽고 쓰기 위해서는 매번 주소를 계산하지 않아도 되는 방법이 필요합니다.  
이번 글에서는 **ARM 어셈블리의 오프셋(Offset) 주소 지정 방식**을 살펴보며, `LDR`과 `STR` 명령어에서 **주소 계산을 단 한 줄로 처리하는 방법**을 이해해보겠습니다.  

## 오프셋의 필요성

어셈블리에서 `ldr r0, [r1]`처럼 단순히 하나의 주소만을 참조하는 경우, 항상 동일한 메모리 주소에만 접근합니다.  
하지만 배열이나 구조체처럼 **연속된 메모리**에 접근해야 한다면, 매번 [`add` 명령]({% post_url 2025-10-27-arm-arithmetic-operations %})으로 주소를 계산해야 하죠.  

예를 들어 다음과 같은 코드가 있다고 가정해보겠습니다.
```armasm
ldr r0, [r1]       @ 첫 번째 데이터 읽기
add r1, r1, #4     @ 다음 데이터 주소로 이동
ldr r2, [r1]       @ 두 번째 데이터 읽기
```

이 방법은 작동하지만, 매번 `add` 명령을 써야 하므로 번거롭습니다.
이 문제를 해결하기 위해 ARM은 오프셋(offset) 기능을 제공합니다.
즉, 주소 계산을 `ldr`/`str` 명령 안에서 함께 수행하는 것이죠.

```armasm
ldr r0, [r1, #4]   @ r1 + 4 주소의 데이터를 바로 읽음

```
이렇게 하면 `add` 없이도 다음 데이터를 바로 읽을 수 있습니다.
명령어 수가 줄고, CPU가 계산하는 사이클도 감소하므로 성능도 개선됩니다.

<figure style="text-align: center;">
  <img src="/assets/img/arm-offset-memory.png" alt="바이트 주소와 int 배열 배치: base + offset" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">바이트 단위 주소에서 <code>int</code>(4바이트) 배열은 인덱스가 1 늘 때마다 주소가 4 증가합니다.</figcaption>
</figure>

## 오프셋을 활용한 주소 계산 원리

오프셋은 말 그대로 **기준 주소(Base Register)**로부터 얼마나 떨어져 있는지를 나타내는 값입니다.
ARM에서는 이 오프셋을 베이스 레지스터에 더하거나 빼서 실제 메모리 주소를 계산합니다.
```
주소 = 베이스 레지스터 + 오프셋
```
오프셋은 다음 두 가지 형태로 지정할 수 있습니다.

- 즉시값(Immediate) — 상수 형태의 값 (`#4`, `#8` 등)
- 레지스터(Register) — 다른 레지스터의 값 (`r2`, `r3` 등)

예를 들어:
```armasm
ldr r0, [r1, #8]    @ 주소 = r1 + 8
ldr r0, [r1, r2]    @ 주소 = r1 + r2
```
오프셋은 덧셈(+), 뺄셈(-) 모두 지원하며,
즉시값의 경우 12비트(`0`~`4095`) 범위를 가집니다.


## 세 가지 문법: 즉시값 / 레지스터 / 시프트 레지스터
ARM은 오프셋을 표현하기 위해 다음 세 가지 문법을 제공합니다.


<figure style="text-align: center;">
  <img src="/assets/img/arm-offset-forms.png" alt="오프셋 세 가지 문법: 즉시값, 레지스터, 시프트 레지스터" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">오프셋은 즉시값, 레지스터, 시프트-레지스터 세 가지로 표현할 수 있습니다.</figcaption>
</figure>

### 1. 즉시값 오프셋 (Immediate Offset)
주소 계산 시 고정된 상수를 더합니다.
```
[Rn, #imm]
```
예시:
```armasm
ldr r0, [r1, #12]   @ r1 + 12 주소의 데이터를 로드
```

### 2. 레지스터 오프셋 (Register Offset)
다른 레지스터의 값을 더합니다.
```
[Rn, Rm]
```
예시:
```armasm
ldr r0, [r1, r2]      @ r1 + r2 주소의 데이터를 로드
```

### 3. 시프트 레지스터 오프셋 (Shifted Register Offset)
다른 레지스터를 시프트(shift)한 후 더합니다.
```
[Rn, Rm, Shift #n]
```
예시:
```armasm
ldr r0, [r1, r2, lsl #2]  @ 주소 = r1 + (r2 << 2)
```
여기서 `lsl #2`는 `×4`와 같은 의미입니다.
int 자료형이 4바이트라면, 배열에서 한 요소를 건너뛸 때 정확히 맞는 크기죠.

## 예제 코드

다음 예제는 오프셋을 사용하여 메모리 주소를 계산하는 방법을 보여줍니다.
```armasm
  ldr r1, =0x1000       @ 베이스 주소 (배열의 시작 주소라고 가정)
  ldr r2, =3            @ 인덱스 i = 3
  ldr r0, [r1, r2, lsl #2]  @ 주소 = 0x1000 + (3 << 2) = 0x100C
                            @ r0 = arr[3]의 값
```
이 예제는 배열 arr[i]의 동작과 동일합니다.
`lsl #2`는 `i * 4`를 의미하며, 각 요소가 4바이트이기 때문에 정확히 다음 주소를 가리킵니다.


<figure style="text-align: center;">
  <img src="/assets/img/arm-offset-forms.png" alt="lsl #2로 i×4 스케일" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;"><code>lsl #2</code>는 인덱스 <code>i</code>를 4배로 스케일링하여 <code>int</code> 크기(4바이트)에 맞춥니다.</figcaption>
</figure>


## C언어 → 어셈블리 변환

C 코드에서 배열의 특정 요소를 읽을 때, 주소 계산은 자동으로 이루어집니다.
예를 들어 다음 C 코드를 보겠습니다.
```c
int arr[4] = {10, 20, 30, 40};
int x = arr[2];
```

이 코드는 컴파일 시 다음과 같은 어셈블리로 변환됩니다.
```armasm
ldr r1, =arr             @ r1 = &arr[0]
ldr r0, [r1, #8]         @ arr[2] → base + (2 * 4) = +8
```

또는 인덱스가 변수라면 다음과 같이 변환됩니다.
```armasm
ldr r1, =arr
mov r2, r0               @ r2 = i
ldr r0, [r1, r2, lsl #2] @ arr[i] = *(base + i*4)
```


정리하자면:
- 배열의 각 요소는 4바이트 단위로 저장됩니다.
- 주소는 항상 “바이트 단위”로 계산되므로, 인덱스가 1 증가하면 주소는 +4 증가합니다.
- ARM 어셈블리는 이를 LSL #2로 효율적으로 표현합니다.

## 마무리
이번 포스팅에서는 LDR과 STR 명령어에서 오프셋을 사용해
메모리 주소를 계산하는 방법을 배웠습니다.

오프셋을 활용하면 단 한 줄로 주소 계산 + 메모리 접근을 처리할 수 있습니다.
이제 배열이나 구조체처럼 연속된 메모리 데이터를 다룰 때 훨씬 효율적으로 코드를 작성할 수 있습니다.

다음 포스팅에서는 pre-index와 post-index 방식을 사용하여
주소를 자동으로 갱신하는 방법을 알아보겠습니다.