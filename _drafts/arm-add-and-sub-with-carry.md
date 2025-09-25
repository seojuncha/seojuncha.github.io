---
layout: post
lang: ko
ref: ""
title: "ARM 어셈블리 #6 - 산술연산"
date: 2025-09-25 21:40:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: []
published: false
---

지금까지는 ARM 어셈블리로 레지스터에 값을 할당하고 시프트하는 방법을 배웠다면 오늘은 산술연산을 다뤄보겠습니다.
수학적으로 기술할 수 있는 기본적인 산술 연산(사칙연산)은 덧셈, 뺄셈, 곱셈, 나눗셈이 있습니다.
ARMv4기반의 어셈블리 코드에서는 `add`, `sub`, `mul`을 사용하여 덧셈과 뺄셈 그리고 곱셈를 할 수 있습니다.

이번 포스팅에서는 사칙연산 명령어들 중 기본적인 덧셈/뺄셈과 캐리를 포함하는 덧셈/뺄셈을 함께 알아보겠습니다.

> 곱셈과 나눗셈에 관한 내용은 성격이 조금 달라서 별도의 포스팅에 작성할 예정입니다!

## ARMv4의 덧셈과 뺄셈
> 지금까지는 ISA버전과 무관하게 "ARM에서 제공하는 명령어"라고 했지만, 앞으로는 조금 더 정확히 ARMv4라고 명시하겠습니다.

오늘 다루게 될 명령어들입니다.
| 명령어 | 설명 |
| :-: | :-: |
| `add` | 더하기 |
| `adc` | 더하기(캐리 포함)|
| `sub` | 빼기 |
| `sbc` | 빼기(캐리 포함) |
| `rsb` | 역방향 빼기 |
| `rsc` | 역방향 빼기(캐리 포함) |

코드를 작성할 때, 위 6개 명령어들은 모두 [이항 연산자]()인 것을 생각하면 도움이 될 것 같습니다.
```
result = first-operand op second-operand

<opcode> = <Rd>, <Rn>, <shifter-operand>
  <opcode> = op
  <Rd> = result
  <Rn> = first-operand
  <shifter-operand> = second-operand
```
사칙연산 명령어들도 ARM 데이터 처리 명령의 일부입니다. 
그렇기 때문에 [이전 포스팅]()에서 얘기한것 처럼 시프트 피연산자를 사용하고 있습니다.

예를 들어 이런식으로 작성할 수 있습니다.
```armasm
```

각각의 명령어들을 하나씩 알아보겠습니다.

## ADD
```
어셈블리 문법: add <Rd>, <Rn>, <Shifter-Operand>

수학적 표현: C = A + B
  C: <Rd>
  A: <Rn>
  B: <Shifter-Operand>
```
--> 그림이 나을듯!

수학에서는 `c=a+b`와 `c=b+a`가 동일한, 교환법칙이 성립하지만, 어셈블리 문법에서는 성립하지 않음을 유의해야 합니다.

유효한 문법:
```armasm
add r0, r1, #2
```
- 첫번째 피연산자는 레지스터: `R1`
- 두번째 피연산자는 시프트 피연산자의 형태중 즉시값: `#2`

유효하지 않은 문법:
```armasm
add r0, #2, r1
```
- 첫번째 피연산자는 무조건 레지스터여야 합니다.

### ADD 예제 코드
**add.s**
{% highlight armasm mark_lines="5" %}
  .text
  .global _start:
_start:
  mov r0, #3
  add r1, r0, #3
  add r2, r0, lsl #2
  b .
{% endhighlight %}
 
주요 라인 풀이:
- `mov r0, #3`:
  - `R0 = 3`
- `add r1, r0, #3`:
  - `R1 = R0 + 3`
  - `R1= 3 + 3`
  - `R1 = 6`
- `add r2, r0, r0 lsl #2`:
  - `R2 = R0 + (3 << 2)`
  - `R2 = 3 + (3 << 2)
  - `R2 = 3 + 12`
  - `R2 = 15`

`add r2, r0, r0 lsl #2`에서 `r0 lsl #2`만큼이 두번째 피연산자인 시프트 피연산자입니다.
그렇기 때문에 `add`동작 이전에 `R0`를 2만큼 왼쪽 시프트한 이후에 그 값을 덧셈에 사용했습니다.

## SUB
```
어셈블리 문법: sub <Rd>, <Rn>, <Shifter-Operand>

수학적 표현: C = A - B
  C: <Rd>
  A: <Rn>
  B: <Shifter-Operand>
```

`sub`는 첫번째 피연산자(`<Rn>`)에서 두번째 피연산자(`<Shifter-Operand>`)를 빼는 동작입니다.

### SUB 예제 코드
**sub.s**
{% highlight armasm mark_lines="5" %}
  .text
  .global _start:
_start:
  mov r0, #4
  sub r1, r0, #0x1
  b .
{% endhighlight %}

주요 라인 풀이:
- `mov r0, #4`:
  - `R0 = 4`
- `sub r1, r0, #0x1`:
  - `R1 = R0 - 0x1`
  - `R1 = 4 - 1`
  - `R1 = 3`

## RSB
```
어셈블리 문법: rsb <Rd>, <Rn>, <Shifter-Operand>

수학적 표현: C = B - A
  C: <Rd>
  A: <Rn>
  B: <Shifter-Operand>
```

`rsb`는 **두번째** 피연산자(`<Shifter-Operand>`)에서 첫번째 피연산자(`<Rn>`)를 빼는 동작입니다.
`sub`의 반대 방향 뺄셈이기 때문에 *Reverse SuB* 입니다.

### RSB 예제 코드
**rsb.s**
{% highlight armasm mark_lines="6" %}
  .text
  .global _start:
_start:
  mov r0, #5
  mov r1, #2
  rsb r2, r1, r0
  b .
{% endhighlight %}

주요 라인 풀이:
- `mov r0, #5`:
  - `R0 = 5`
- `mov r1, #2`:
  - `R1 = 2`
- `rsb r2, r1, r0`:
  - `R2 = R0 - R1`
  - `R2 = 5 - 2`
  - `R2 = 3`

만약, `rsb`가 아닌 `sub`를 사용했다면, `R2 = R1 - R0`이기 때문에 `-3`이 될 것입니다.

## ADC, SBC, RBC
기본적인 `add`, `sub`, 그리고 `rsb`는 크게 어렵지 않습니다. 
하지만 컴퓨터의 연산은 그렇게 간단하지 않죠.

덧셈과 뺄셈에서는 올림과 빌림이라는 개념이 있습니다.
수학적으로 이 두개는 조금 다른 의미이지만 컴퓨터는 이것들을 캐리(Carry)로 표현합니다.

> 컴퓨터 연산에서 캐리에 관한 내용은 [이 포스팅]에서 자세하게 다루었습니다.

그래서 캐리를 포함한 연산을 위해 아래와 같은 별도의 명령어를 제공합니다.

캐리를 포함한 버전:
- `adc` = `add` + 캐리(올림)
- `sbc` = `sub` + 캐리(빌림)
- `rbc` = `rsb` + 캐리(빌림)
 
### 예제: 캐리를 사용한 덧셈과 뺄셈
**adc.s**
```armasm
  .text
  .global _start
_start:
  mov r0, #0x3
  movs r0, r0, lsr #1
  mov r1, #2
  adc r2, r0, r1
  b .
```
 
주요 라인별 설명:
- `mov r0, #0x3`
  - `R0 = 3`
- `movs r0, r0, lsr #1`
  - `R0 = R0 >> 1`
  - `R0 = 3 >> 1`
  - `R0 = 1
  - 단, `movs`이므로 밀려난 비트가 캐리플래그로 설정
- `mov r1, #2`
  - `R1 = 2`
- `adc r2, r0, r1`
  - `R2 = R0 + R1 + Carry`
  - `R2 = 1 + 2 + 1`
  - `R2 = 4`
 
디버깅:
```
(gdb) x/5i 0x10000
=> 0x10000 <_start>:    mov     r0, #3
   0x10004 <_start+4>:  lsrs    r0, r0, #1
   0x10008 <_start+8>:  mov     r1, #2
   0x1000c <_start+12>: adc     r2, r0, r1
   0x10010 <_start+16>: b       0x10010 <_start+16>
(gdb) p ($cpsr >> 29) & 1
$1 = 0
(gdb) i r r0 r1 r2
r0             0x0      0
r1             0x0      0
r2             0x0      0
(gdb) si
0x00010004 in _start ()
(gdb) i r r0 r1 r2
r0             0x3      3
r1             0x0      0
r2             0x0      0
(gdb) p ($cpsr >> 29) & 1
$2 = 0
(gdb) si
0x00010008 in _start ()
(gdb) p ($cpsr >> 29) & 1
$3 = 1
(gdb) i r r0 r1 r2
r0             0x1      1
r1             0x0      0
r2             0x0      0
(gdb) si
0x0001000c in _start ()
(gdb) i r r0 r1 r2
r0             0x1      1
r1             0x2      2
r2             0x0      0
(gdb) si
0x00010010 in _start ()
(gdb) p ($cpsr >> 29) & 1
$4 = 1
(gdb) i r r0 r1 r2
r0             0x1      1
r1             0x2      2
r2             0x4      4
(gdb)
```


**sbc.s**
```armasm

```


## 마무리