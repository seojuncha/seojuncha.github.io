---
layout: post
lang: ko 
ref: "arm-cmp-instruction"
title: "ARM 어셈블리에서 CMP 명령은 어떻게 동작할까?"
date: 2025-12-04 18:00:00 +0900
categories: ["arm", "assembly"]
tags: ["arm cmp", "arm cpsr"]
---
ARM에서 `CMP` 명령은 두 값을 비교하기 위해 사용되는 명령어입니다.
**비교는 내부적으로 두 값을 뺄셈**하여 결과의 상태를 확인하는 방식으로 이루어집니다.
예를 들어, `5 - 3 = 2`는 양수이므로 `5`가 `3`보다 크다는 것을 알 수 있고, `3 - 5 = -2`는 음수이므로 `3`이 `5`보다 작다는 것을 알 수 있습니다.
또한 `5 - 5 = 0`이므로 두 값이 동일하다는 것도 확인할 수 있습니다.

`CMP` 명령은 이러한 산술 원리를 이용하여 실제 계산 결과 대신, 결과의 상태만 `CPSR`에 기록합니다.

[**GitHub 예제 코드**](https://github.com/seojuncha/arm-assembly-tutorial/tree/main/06_comparing-values)

## CPU 연산과 CPSR의 관계
ARM에서는 모든 산술 및 논리 연산의 상태가 `CPSR`에 저장됩니다.
`CPSR`의 최상위 4비트에는 `N`, `Z`, `C`, `V` 플래그가 위치하며, 계산 결과를 정리합니다.

> 연산 결과값 자체는 레지스터에 저장되며, `CPSR`에는 그 결과의 상태만 저장됩니다.

`CPSR`에 대한 자세한 내용은 [이 포스팅]({% post_url 2025-11-27-arm-cpsr-condition-flags %})을 참고해 주세요.

## CMP 명령
```text
    cmp Rn, Operand2
```

여기서 `Rn`은 반드시 레지스터여야 하며, `Operand2`는 시프트 피연산자(Shifter Operand)입니다.
앞서 설명한 것처럼 `CMP`는 실질적으로 뺄셈 연산(`Rn - Operand2`) 을 수행합니다.

하지만, 그 결과값을 `Rn`에 저장하지 않고, `CPSR` 플래그만 갱신합니다.
즉, `cmp Rn, Operand2`는 `subs Rn, Rn, Operand2`와 동일한 연산을 수행하지만 **레지스터 값은 변경되지 않습니다**.

> `CMP`를 비롯한 몇몇 비교 명령어는 접미사 `s` 없이도 `CPSR`을 **자동으로 갱신**하는 특별한 명령어입니다.

## 예제 코드
```text
  .text
  .global _start

_start:
    mov r0, #5
    mov r1, #5
    cmp r0, r1
    beq equal

equal:
    mov r2, #0xff
```

`cmp r0, r1` 실행 후 `CPSR`의 상태는 다음과 같습니다.

| CPSR 플래그 | 값 | 이유      |
| -------- | - | --------- |
| N        | 0 | 5 - 5 ≥ 0 |
| Z        | 1 | 5 - 5 = 0 |
| C        | 1 | 빌림 없음     |
| V        | 0 | 오버플로우 없음  |

> `CMP`에서 `C` 플래그(Bit[29])는 부호 없는(unsigned) 비교용입니다.  
> 빌림이 없으면 `C=1`, 빌림이 발생하면 `C=0`입니다.

## 디버깅
```bash
(gdb) i r cpsr
0x60000010
```
GDB에서 `i r cpsr` 명령을 실행하면 `CMP`로 인해 갱신된 `CPSR`값을 확인할 수 있습니다.
`CPSR`은 32비트 전체가 출력되지만, 우리가 관심 있는 영역은 최상위 4비트(`N`, `Z`, `C`, `V`) 입니다.

파이썬을 사용해 이 비트들을 쉽게 확인할 수 있습니다.
```python
cpsr = 0x60000010
flags = (cpsr >> 28) & 0xF
print("{:04b}".format(flags))
```

## 참고자료
- [ARM Architecture Reference Manual](https://github.com/seojuncha/arm-assembly-tutorial/blob/main/doc/arm-architecture-reference-manual_DDI0100.pdf)
- [Python3 Input and Output](https://docs.python.org/3/tutorial/inputoutput.html)
- [Python3 Format Specification](https://docs.python.org/3/library/string.html#formatspec)