---
layout: post
lang: ko
ref: "arm-rotate-shift"
title: "[ARM32] ROR, RRX로 비트를 회전하는 방법"
date: 2025-09-17 22:40:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["ror", "rrx", "arm", "assembly", "rotate shift"]
---

이번 글에서는 시프트 연산의 마지막 주제인 **회전 시프트 (Rotate Shift)** 를 다룹니다. 
ARMv4에서는 두 가지 회전 시프트 명령어를 제공합니다: `ror` (Rotate Right), `rrx` (Rotate Right with eXtend)

이 명령어들은 단순히 비트를 밀어내는 것이 아니라, **밀려난 비트를 다시 반대편에서 채워 넣는** 방식으로 동작합니다.

## ARM 에서 제공하는 회전 시프트

| 명령어 | 설명 |
|:---:|:---:|
| `ror` | 오른쪽 회전 |
| `rrx` | 오른쪽 회전 확장|

`ror`은 다른 시프트 연산자들과 동일하게 즉시값이나 레지스터로 시프트양을 지정할 수 있습니다.
- `ror #<imm>` : 즉시값 만큼 시프트 연산
- `ror <Rs>` : 레지스터값 만큼 시프트 연산

`rrx`는 피연산자를 가지지 않으며, **항상 1비트**를 시프트합니다. 대신 **CPSR의 캐리(Carry) 플래그**를 포함하여 33비트 시프트처럼 동작합니다.

## ROR: Rotate Right
`ror` 명령은 오른쪽으로 시프트되는 비트를 **최상위 비트로 되돌려** 삽입합니다.  
즉, **모든 비트를 보존하면서 위치만 바꾸는** 연산입니다.

**ror.s**
```
  .text
  .global _start
_start:
  mov r0, #0x96
  mov r1, r0, ror #4
```

`R0`의 초기값은 `0x96`(`0b10010...0110`)입니다.
`ror #4`를 실행하면 오른쪽 4비트를 잘라낸 후, 그것을 다시 상위 비트에 붙입니다.

```
Before:   0b0000_0000_0000_0000_0000_0000_1001_0110 = 0x96
After:    0b0110_0000_0000_0000_0000_0000_0000_1001 = 0x60000009
```

## RRX: Rotate Right with Extend
`rrx`는 `ror #1`과 거의 같지만, **CPSR의 캐리 플래그**와 함께 작동하여 32비트가 아닌 **33비트 시프트처럼** 동작합니다.
- 오른쪽으로 1비트를 밀면, LSB는 캐리 플래그로 이동
- 동시에 현재 캐리 플래그의 값이 MSB로 채워짐

**rrx.s**
```
  .text
  .global _start
_start:
  mov r0, #0x69
  movs r1, r0, rrx
  movs r2, r1, rrx
  b .
```

```
0b0000_0000_0000_0000_0000_0000_0110_1001 = 0x69

[rrx/C=0]: 0b0000_0000_0000_0000_0000_0000_0011_0100 -> R1=0x34, C=1
[rrx/C=1]: 0b1000_0000_0000_0000_0000_0000_0001_1010 -> R2=0x8000001a, C=0
```

### GDB로 캐리 플래그 확인
```bash
$ arm-none-eabi-gcc -Ttext=0x10000 -nostdlib rrx.s -o rrx.elf
$ qemu-system-arm -machine versatilepb -nographic -S -s -kernel rrx.elf
$ gdb-multiarch rrx.elf
```

{% highlight bash mark_lines="11 17" %}
(gdb) target remote :1234
(gdb) si                          # mov r0, #0x69 실행
0x00010004 in _start ()
(gdb) i r r0
r0             0x69     105
(gdb) p ($cpsr >> 29) & 1         # 최초 캐리플래그 확인
$3 = 0
(gdb) si                          # movs r1, r0, rrx 실행
0x00010008 in _start ()
(gdb) p ($cpsr >> 29) & 1
$4 = 1
(gdb) i r r1
r1             0x34     52
(gdb) si                          # movs r2, r1, rrx 실행
0x0001000c in _start ()
(gdb) p ($cpsr >> 29) & 1
$5 = 0
(gdb) i r r0 r1 r2
r0             0x69     105
r1             0x34     52
r2             0x8000001a       -2147483622
{% endhighlight %}

## 왜 왼쪽 회전 시프트는 없을까?
ARMv4T에는 ROR(우회전)만 있고 ROL(좌회전)은 없습니다. 이유는 두 가지입니다.
첫째, 32비트 회전에서는 좌회전과 우회전이 서로 같습니다. `n`비트 좌회전은 `32 - n`비트 우회전과 결과가 완전히 같기 때문입니다.
따라서 `ror` 하나만 있으면 좌회전도 표현할 수 있고, `rol`을 따로 둘 이유가 없습니다.

둘째, ARM은 시프트를 독립 명령이 아니라 barrel shifter를 통해 데이터 처리 명령의 두 번째 오퍼랜드에 통합해 처리합니다.
이때 시프트 종류를 지정하는 필드는 2비트뿐이라 네 가지(`lsl`, `lsr`, `asr`, `ror`)만 담을 수 있습니다.
좌방향은 이미 `lsl`이 차지하고 있고 회전은 좌우가 동치이므로, 제한된 인코딩 공간을 효율적으로 사용하기 위해 `ror`하나만 채택했습니다.


## 마무리
이번 포스팅에서는 ARM의 회전 시프트 명령어인 `ror`과 `rrx`를 살펴봤습니다.
- `ror`: 비트를 오른쪽으로 밀고 최상위 비트로 다시 삽입
- `rrx`: 캐리 플래그를 포함한 33비트 순환 시프트
- rol은 명령어로 존재하지 않지만 `lsl` + `ror` 조합으로 구현 가능

다음 글에서는 이 시프트 연산들이 실제 연산 명령어(`add`, `sub`)와 어떻게 결합되어 쓰이는지,
그리고 Carry/Overflow 플래그의 의미와 역할에 대해 자세히 다룰 예정입니다.
