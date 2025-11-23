---
layout: post
lang: ko
ref: "arm-basic-branch"
title: "ARM 어셈블리 #13 - B 명령어로 실행 흐름 분기하기"
date: 2025-11-18 22:00:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["arm b", "arm branch instruction"]
published: false
---

이번 포스팅에서는 명령어 분기를 위한 B 명령어의 사용법을 실제 예제와 함께 정리해보겠습니다.
B 명령어는 조건 없이 실행 흐름을 특정 위치로 이동시키는 명령어이며, 조건문·반복문·함수 구조 같은 큰 흐름을 만드는 기본 재료입니다.

- [예제 코드](https://github.com/seojuncha/arm-assembly-tutorial/blob/main/05_branching/branch.s)
- [YouTube 영상]()

## 왜 B 명령어를 사용할까?
B 명령어는 단순히 코드의 흐름을 다른 위치로 옮기기 위한 명령어입니다.
함수 호출처럼 되돌아오는 개념은 없으며, C 언어의 goto와 거의 동일한 수준의 동작을 수행합니다.

사용 예는 다음과 같습니다:
- 루프 구현
- 특정 코드 블록 건너뛰기
- 단순 흐름 전환

## 실행 흐름 분기의 기본 원리
CPU는 프로그램 카운터(PC)에 다음 실행할 명령어의 주소를 저장합니다.
분기 명령을 실행하면 PC를 다른 주소로 변경하여, CPU가 다음 명령어를 가져오는 위치를 바꿉니다.

> 참고: ARMv4 파이프라인에서는 현재 명령어 기준 PC가 +8 오프셋으로 보입니다.
> 하지만 분기 오프셋 계산은 어셈블러가 처리하므로, 코드 작성 시 신경 쓸 필요는 없습니다.

## 브랜치 명령어: B
```armasm
  b label
```
`b label`은 레이블의 주소로 바로 점프하라는 의미입니다.
이 동작은 내부적으로 PC 값을 레이블 주소로 교체하는 것과 동일합니다.

## 레이블(Label)의 의미
레이블은 특정 명령어 위치에 이름을 붙인 것입니다.
```armasm
  label:
    instructions
```
레이블 이름 뒤에는 반드시 콜론(:)을 붙여야 합니다.
레이블의 주소는 레이블 아래 첫 번째 명령어의 주소입니다.

예:
```armasm
  foo:
    mov r0, #1   @ address = 0x10000
    mov r0, #2   @ address = 0x10004
```

여기서 foo의 주소는 `0x10000`입니다.

## 예제 코드
```armasm
  .text
  .global _start
_start:
  mov r0, #2
  b   foo
  mov r1, r0     @ never executed
foo:
  add r0, r0, #3
  b   _start
```

**실행 흐름**
1. `mov r0, #2`
2. `b foo` → foo로 이동
3. `add r0, r0, #3`
4. `b _start` → 다시 처음 위치로 이동
5. `mov r0, #2` 다시 실행

이렇게 R0 값은 2 → 5 → 2 → 5 로 반복됩니다.
`mov r1, r0`는 분기 때문에 절대 실행되지 않습니다.

## 디버깅: GDB로 PC 변화 관찰하기
```bash
(gdb) target remote :1234
(gdb) b _start
(gdb) b foo
(gdb) display/i $pc
(gdb) c
(gdb) si
```
`display/i $pc`는 한 단계씩 실행할 때 PC가 어디로 이동하는지 확인하기 좋습니다.
특히 b foo 명령어가 실행될 때, PC가 정확히 레이블의 주소로 변경되는 것을 확인할 수 있습니다.

## 마무리
이번 포스팅에서는 ARMv4에서 가장 기본적인 분기 명령어인 `B`를 살펴보았습니다.
다음 포스팅에서는 함수 호출에 사용되는 `BL` 명령어를 다뤄보겠습니다.
BL은 되돌아올 주소를 자동으로 `LR` 레지스터에 저장한다는 점에서 구조적으로 중요한 차이를 가집니다.