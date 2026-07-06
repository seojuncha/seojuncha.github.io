---
layout: post
lang: ko
ref: "arm-arithmetic-operations"
title: "[ARM32] 산술연산"
date: 2025-10-27 20:20:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["add", "sub", "adc", "sbc", "rsb", "rsc"]
---

이번 포스팅에서는 **ARM의 기본 산술 연산(Arithmetic Operations)** 명령어들을 다뤄보겠습니다.  
특히 덧셈과 뺄셈, 그리고 **Carry(캐리)**를 포함하는 연산까지 함께 살펴봅니다.
지금까지는 레지스터에 값을 저장하고 시프트(shift)하는 방법을 배웠다면, 오늘은 그 값을 **연산**하는 단계로 나아가 보겠습니다.

> 곱셈과 나눗셈에 관한 내용은 성격이 조금 달라서 별도의 포스팅에 작성할 예정입니다!

## ARMv4의 덧셈과 뺄셈
ARM 어셈블리의 기본 산술 연산 명령어는 다음과 같습니다:

| 명령어 | 설명 |
| :-: | :-: |
| `add` | 더하기 |
| `adc` | 더하기(캐리 포함)|
| `sub` | 빼기 |
| `sbc` | 빼기(캐리 포함) |
| `rsb` | 역방향 빼기 |
| `rsc` | 역방향 빼기(캐리 포함) |

> 이제부터는 “ARM” 대신 좀 더 정확히 **ARMv4**라고 명시하겠습니다. 

이 명령어들은 모두 **이항 연산자(binary operator)** 형태입니다.

```
result = first-operand op second-operand
<opcode> = <Rd>, <Rn>, <shifter-operand>
```

| 필드 | 의미 |
| :--: | :--: |
| `<opcode>` | 연산 종류 (add, sub 등) |
| `<Rd>` | 결과가 저장될 레지스터 |
| `<Rn>` | 첫 번째 피연산자 |
| `<shifter-operand>` | 두 번째 피연산자 (즉시값 또는 시프트된 레지스터) |

즉, 이 명령어들도 모두 [이전 포스팅]({% post_url 2025-09-25-arm-sfhiter-operand %})에서 다뤘던 **시프트 피연산자(Shifter Operand)**를 사용합니다.

## 1. ADD : 더하기
```
어셈블리 문법: add <Rd>, <Rn>, <Shifter-Operand>

수학적 표현: C = A + B
  C: <Rd>
  A: <Rn>
  B: <Shifter-Operand>
```

수학에서는 `C = A + B`와 `C = B + A`가 같지만,  
**어셈블리에서는 첫 번째 피연산자(`<Rn>`)의 위치가 고정**되어 있으므로 교환법칙이 성립하지 않습니다.

**올바른 예시**
```
add r0, r1, #2
```
- 첫번째 피연산자는 레지스터: `R1`
- 두번째 피연산자는 시프트 피연산자의 형태중 즉시값: `#2`

**잘못된 예시**
```
add r0, #2, r1
```
- 첫번째 피연산자는 반드시 레지스터여야 합니다.

### ADD 예제
**add.s**
{% highlight text mark_lines="5" %}
  .text
  .global _start:
_start:
  mov r0, #3
  add r1, r0, #3
  add r2, r0, lsl #2
  b .
{% endhighlight %}
 
주요 라인 해석:
- `mov r0, #3`:
  - `R0 = 3`
- `add r1, r0, #3`:
  - `R1 = R0 + 3`
  - `R1= 3 + 3`
  - `R1 = 6`
- `add r2, r0, r0 lsl #2`:
  - `R2 = R0 + (3 << 2)`
  - `R2 = 3 + (3 << 2)`
  - `R2 = 3 + 12`
  - `R2 = 15`

`add r2, r0, r0 lsl #2`에서 `r0 lsl #2`만큼이 두번째 피연산자인 시프트 피연산자입니다.
그렇기 때문에 `add`동작 이전에 `R0`를 2만큼 왼쪽 시프트한 이후에 그 값을 덧셈에 사용했습니다.

## 2. SUB : 빼기
```
어셈블리 문법: sub <Rd>, <Rn>, <Shifter-Operand>

수학적 표현: C = A - B
  C: <Rd>
  A: <Rn>
  B: <Shifter-Operand>
```

`sub`는 첫번째 피연산자(`<Rn>`)에서 두번째 피연산자(`<Shifter-Operand>`)를 빼는 동작입니다.

### SUB 예제
**sub.s**
{% highlight text mark_lines="5" %}
  .text
  .global _start:
_start:
  mov r0, #4
  sub r1, r0, #0x1
  b .
{% endhighlight %}

주요 라인 해석:
- `mov r0, #4`:
  - `R0 = 4`
- `sub r1, r0, #0x1`:
  - `R1 = R0 - 0x1`
  - `R1 = 4 - 1`
  - `R1 = 3`

## 3. RSB : 역방향 빼기
```
어셈블리 문법: rsb <Rd>, <Rn>, <Shifter-Operand>

수학적 표현: C = B - A
  C: <Rd>
  A: <Rn>
  B: <Shifter-Operand>
```

`rsb`는 **두번째** 피연산자(`<Shifter-Operand>`)에서 첫번째 피연산자(`<Rn>`)를 빼는 동작입니다.
`sub`의 반대 방향 뺄셈이기 때문에 *Reverse SuB* 입니다.

### RSB 예제
**rsb.s**
{% highlight text mark_lines="6" %}
  .text
  .global _start:
_start:
  mov r0, #5
  mov r1, #2
  rsb r2, r1, r0
  b .
{% endhighlight %}

주요 라인 해석:
- `mov r0, #5`:
  - `R0 = 5`
- `mov r1, #2`:
  - `R1 = 2`
- `rsb r2, r1, r0`:
  - `R2 = R0 - R1`
  - `R2 = 5 - 2`
  - `R2 = 3`

만약, `rsb`가 아닌 `sub`를 사용했다면, `R2 = R1 - R0`이기 때문에 `-3`이 될 것입니다.

## 4. ADC/SBC/RBC : 캐리를 포함한 연산
기본적인 `add`, `sub`, `rsb`만으로도 대부분의 연산이 가능합니다.
하지만 컴퓨터는 항상 “**올림(Carry)**”과 “**빌림(Borrow)**”을 고려해야 합니다.

캐리를 고려한 연산 명령어는 다음과 같습니다:
- `adc` = `add` + 캐리(올림)
- `sbc` = `sub` + 캐리(빌림)
- `rbc` = `rsb` + 캐리(빌림)
 
### ADC 예제
**adc.s**
```text
  .text
  .global _start
_start:
  mov r0, #0x3
  movs r0, r0, lsr #1
  mov r1, #2
  adc r2, r0, r1
  b .
```
 
주요 라인 해석:
- `mov r0, #0x3`
  - `R0 = 3`
- `movs r0, r0, lsr #1`
  - `R0 = R0 >> 1`
  - `R0 = 3 >> 1`
  - `R0 = 1`
  - 단, `movs`이므로 밀려난 비트가 캐리플래그로 설정
- `mov r1, #2`
  - `R1 = 2`
- `adc r2, r0, r1`
  - `R2 = R0 + R1 + Carry`
  - `R2 = 1 + 2 + 1`
  - `R2 = 4`
 
디버깅:
```bash
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

## 마무리
이번 포스팅에서는 ARMv4에서 사용되는 기본 산술 연산 명령어들을 살펴봤습니다.

| 종류 | 명령어 | 특징 | 
|:-:|:-:|:-:|
|덧셈 |`add`, `adc` | `adc` 는 캐리를 더함 |
|뺄셈 |`sub`, `sbc` | `sbc`는 캐리(혹은 빌림)을 고려 | 
|역방향 뺄셈 | `rsb`, `rsc` | 순서를 뒤집은 뺄셈 |