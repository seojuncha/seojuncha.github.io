---
layout: post
lang: ko
ref: "arm-arithmetic-shift"
title: "ARM 어셈블리 #3 - 부호를 유지하는 시프트인 산술시프트 (ASR)"
date: 2025-09-15 21:53:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: [Assembly, ASR, ARM, QEMU, GDB]
---

CPU의 중요한 모듈중 하나인 ALU는 산술논리장치입니다. 이름에서 유추할 수 있듯이 논리연산 뿐 아니라 산술연산도 가능해야 하죠.
산술연산에서 유의해야 할 것은 피연산자의 부호 유무입니다. [앞선 포스팅]({% post_url 2025-09-10-arm-logical-shift %})에서는 논리 시프트를 사용한 비트 단위의 연산을 고려했다면,
이번 포스팅에서는 부호를 고려한 산술 시프트 연산자인 `asr`의 특징과 사용법을 살펴보겠습니다.

## 산술 시프트
ARM에서는 산술 시프트를 위한 하나의 명령어를 제공합니다.

| 명령어 | 설명 |
|:---|:---|
| `asr` | 오른쪽 산술 시프트 |

`asr`은 아래와 같이 사용할 수 있습니다:
- `asr #<imm>` : 즉시값만큼 시프트
- `asr <Rs>` : 레지스터값만큼 시프트

### 산술 시프트의 부호 유지 방법
산술 시프트에서 기존 값의 부호를 유지하기 위해,

 ***"시프트 이전 데이터의 최상위 비트값으로 시프트 공간을 채운다."***

라는 규칙을 가지고 있습니다.

예를 들어, 최상위 비트가 0이고 3비트 만큼 시프트 한다면, 최상위 3비트는 `000`이 되고,
최상위 비트가 1이면, 최상위 3비트는 `111`이 됩니다.

**최상위 비트가 0인 경우**
```
Before (32-bit):  
0 1 0 1 1 1 0 0 ...  

ASR #3 →  

After (32-bit):  
0 0 0 0 1 0 1 1 ...
^ ^ ^  
| | |  
+---- 시프트로 새로 채워진 상위 비트 (부호 유지: 0)
```

**최상위 비트가 1인 경우**
```
Before (32-bit):  
1 0 1 0 0 0 1 1 ...

ASR #3 →  

After (32-bit):  
1 1 1 1 0 1 0 ...
^ ^ ^  
| | |  
+---- 시프트로 새로 채워진 상위 비트 (부호 유지: 1)
```

### 산술 시프트 예제

**asr.s**
{% highlight armasm mark_lines="4 5" %}
  .text
  .global _start
_start:
  mov r0, #-32         @ -32 = 0xffffffe0
  mov r1, r0, asr #1   @ R1 = 0xfffffff0 = -16
  b .
{% endhighlight %}

`-32`는 16진수로 `0xffffffe0` 입니다. 여기서 부호를 유지한채 오른쪽 1비트를 시프트하면:
```
0xffffffe0 >> 1 = 0xfffffff0 (-16)
```
1비트 시프트 한다는 것은 `숫자/2`동작이므로 `-32/2 = -16` 이라는 부호를 유지한 산술연산이 가능해집니다.

## 왜 오른쪽 시프트만 있을까?
결과부터 얘기하면, 왼쪽 시프트는 최상위 비트가 바뀌더라도 그것이 의도한 산술 연산일 수 있기 때문입니다.

**left-shift-for-negative.s**
{% highlight armasm mark_lines="4 5" %}
  .text
  .global _start
_start:
  mov r0, #0x40000000
  movs r1, r0, lsl #1
  b .
{% endhighlight %}

`0x40000000 << 1`은 `0x80000000`이므로 테이블로 작성하면 아래와 같습니다:

| 2진수 | 16진수 | 10진수 |
|:-:|:-:|:-:|
| 0b0100... | **0x4000**0000 | 1073741824 |
| 0b1000... | **0x8000**0000 | 2147483648 혹은 -2147483648 |

보다시피 `0x80000000`이라는 데이터는 정수의 표현법에 따라 양수가 될 수도 있고 음수가 될 수도 있습니다.
그래서 ARM 프로세서는 `0x80000000`과 같이 최상위 비트가 1일 경우 부호 있는 정수 표현의 음수라는 것을 나타내기 위해 CPSR의 음수(Negative) 플래그를 설정합니다.

프로그래머는 CPSR의 플래그를 확인하여 `R1`의 데이터를 부호 있는 정수로 활용할 것인지 아니면 부호 없는 정수로 활용할 것인지 선택할 수 있죠.

> 나중에 살펴볼 산술 연산(예: `add`, `sub`, 등)에서는 부호 있는 정수의 오버플로우를 나타내기 위해 **오버플로우(V) 플래그**라는 것도 설정합니다.
> 지금은 논리연산(`lsl`)만 사용하였으므로 오버플로우 플래그를 설정하지 않습니다.
> 산술연산에서의 플래그 설정 및 확인은 향후 산술 연산을 다루는 포스팅에서 자세히 살펴볼 예정입니다.

<figure style="text-align: center;">
  <img src="/assets/img/non-add-sub-v-left-unchanged.png" alt="ARM 매뉴얼에 명시된 오버플로우 플래그의 설정방법 두가지" style="display: block; margin: auto;" />
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">덧셈이나 뺄셈연산이 아닌 경우 보통 오버플로우 플래그를 변경하지 않는다고 ARM매뉴얼에 명시되어있다.</figcaption>
</figure>

결과적으로 왼쪽 시프트는 오히려 부호를 유지한다는 개념을 가지면 안되고 논리 시프트와 같은 역할을 하기 때문에 `lsl`명령어 하나로만 유지되는 것 입니다.

### GDB로 플래그 확인하기
**left-shift-for-negative.s** 파일을 컴파일하고 실행한 후 GDB로 음수 플래그를 확인해보겠습니다.

#### 1. 컴파일
```bash
$ arm-none-eabi-gcc \
    -Ttext=0x10000 \
    -nostdlib \
    left-shift-for-negative.s \
    -o left-shift-for-negative.elf
```
#### 2. QEMU 실행
```bash
$ qemu-system-arm \
    -machine versatilepb \
    -nographic \
    -S \
    -s \
    -kernel left-shift-for-negative.elf
```
#### 3. GDB 연결 후 플래그 확인
```
$ gdb-multiarch left-shift-for-negative.elf
```

{% highlight gdb mark_lines="7 9 10" %}
(gdb) target remote :1234
0x00010000 in _start ()    
(gdb) si                   # mov r0, #0x40000000 
0x00010004 in _start ()    
(gdb) p ($cpsr >> 31) & 1
$2 = 0
(gdb) si                   # movs r1, r0, lsl #1
0x00010008 in _start ()
(gdb) p ($cpsr >> 31) & 1
$2 = 1
{% endhighlight %}

## 마무리
이번 글에서는 산술시프트 연산자인 `asr`을 활용하여 부호를 유지하고 시프트하는 방법을 알아봤습니다.
- `asr`의 사용
- 산술 시프트에서 왼쪽 시프트가 없는 이유
- 음수 플래그를 확인하는 방법

다음 포스팅에서는 시프트 연산의 마지막 챕터로, 비트를 순환시키는 `ror`과 `rrx` 명령어를 다룰 예정입니다.
