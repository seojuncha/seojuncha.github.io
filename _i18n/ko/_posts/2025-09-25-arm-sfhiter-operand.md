---
layout: post
lang: ko
ref: "arm-shifter-operand"
title: "ARM 어셈블리 #5 - 시프트 피연산자(Shifter Operand) 총정리"
date: 2025-09-25 22:40:00 +0900
categories: ["arm", "assembly", "tutorial"]
tags: ["shifter-operand"]
---
오늘은 ARM 데이터 처리 명령어(Data Processing Instructions)의 **시프트 피연산자(Shifter Operand)**를 총정리해보겠습니다.  
시프트 피연산자는 모든 데이터 처리 명령어에서 **필수적으로 사용되는 중요한 개념**입니다.

이번 글에서는  
1. 시프트 피연산자의 종류와 문법  
2. 즉시값이 표현되는 방식과 제약  
3. ARM에서 시프트 피연산자를 제공하는 이유  

이 세 가지를 중심으로 설명하겠습니다.

## 시프트 피연산자의 활용
`mov`, `add`, `sub` 같은 **데이터 처리 명령어**는 항상 하나 이상의 시프트 피연산자를 사용합니다.  
그리고 최소한 다음과 같은 요소들이 필요합니다:
- 목적지 레지스터(`Rd`)
- 입력 레지스터(`Rn`)
- 시프트 피연산자(`Shifter Operand`)

<figure>
  <img src="/assets/img/arm-data-processing-table.png" style="display: block; margin: auto;" alt="ARM 데이터 처리 명령어 테이블">
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    ARM 아키텍처의 <b>데이터 처리 명령어(Data-processing instructions)</b> 요약.  
    모든 명령어는 공통적으로 <code>shifter_operand</code>를 사용하며, 
    필요에 따라 <code>Rn</code>과 결합해 결과를 <code>Rd</code>에 저장하거나 
    조건 플래그만 갱신합니다.
  </figcaption>
</figure>

### 데이터 처리 명령어 포맷
- `mov`, `mvn` (단항 연산자):

```
<instruction> <Rd>, <Shifter Operand>
```

- 그 외(`add`, `sub`, `and` 등) (이항 연산자):

```
<instruction> <Rd>, <Rn>, <Shifter Operand>
```

즉,
- `mov` / `mvn` 은 `<Shifter Operand>`의 값을 그대로(또는 반전해서) `Rd`에 저장
- 나머지 연산은 `<Rn>`과 `<Shifter Operand>`를 연산해 `Rd`에 저장  

## 시프트 피연산자의 종류
시프트 피연산자는 크게 다음으로 나눌 수 있습니다:

- **즉시값**  
  - `#<immediate>`
- **레지스터 값**  
  - `<Rm>`
- **시프트 연산 적용**  
  - 논리 시프트 왼쪽: `<Rm>, lsl #<shift_imm>` / `<Rm>, lsl <Rs>`  
  - 논리 시프트 오른쪽: `<Rm>, lsr #<shift_imm>` / `<Rm>, lsr <Rs>`  
  - 산술 시프트 오른쪽: `<Rm>, asr #<shift_imm>` / `<Rm>, asr <Rs>`  
  - 순환 시프트: `<Rm>, ror #<shift_imm>` / `<Rm>, ror <Rs>` / `<Rm>, rrx`  

여기서 `<Rm>`은 데이터 처리 명령에 사용되는 **피연산자 레지스터** 중 하나입니다.

데이터 처리 명령어 포맷에서 `<Shifter Operand>`를 위 내용으로 치환할 수 있습니다.

예를 들어:
```
# mov r0, r1
  <instruction> <Rd>, <Shifter Operand>
  <instruction> <Rd>, <Rm>
       mov       r0    r1
 
# add r0, r1, r0, lsl #2
  <instruction> <Rd>, <Rn>, <Shifter Operand>
  <instruction> <Rd>, <Rn>, <Rm> lsl #<shift_imm>
       add       r0    r1    r0         #2
```

### 문법 예제

```armasm
mov r0, r1           @ 유효(<Rd> = r0, <Rm> = r1)
mov r0, #0xff        @ 유효(<Rd> = r0, <Shifter Operand> = 0xff)
mov r0, r0, lsl #2   @ 유효(<Rd> = r0, <Sfhiter Operand> = r0, lsl #2)
add r0, #1           @ 무효(<Rd> = r0, <Rn>이 필요)
```

## 즉시값의 범위
시프트 피연산자 값으로 즉시값(`#<immediate>`)을 사용할 때 주의해야 할 것이 한가지 있습니다.
32비트 ARM프로세서이기 때문에 `0x0` 부터 `0xffffffff` 까지의 데이터를 다룰 수 있다고 생각하지만,
모든 32비트 값을 시프트 피연산자의 유효한 즉시값으로 사용할 수 는 없다는 것입니다.

```armasm
  .text
  .global _start
_start:
  mov r0, #0x101
  b .
```
위의 코드에서 즉시값 `0x101`은 `257`이지만 컴파일 오류가 발생합니다:

```
imm.s: Assembler messages:
imm.s:4: Error: invalid constant (101) after fixup
```

### 즉시값 인코딩

즉시값에 32비트를 모두 할당할 수 없기 때문에, ARM은 회전 시프트와 결합한 특수한 인코딩 방식을 사용합니다.

우선, 즉시값은 12비트로 인코딩됩니다:
- 8비트 값 (imm8)
- 4비트 회전 값 (rotate_imm) -> 실제 회전 각도는 rotate_imm × 2 (예: 0, 2, 4, ... 28, 30)

8비트 즉시값과 4비트 회전값을 사용해서 즉시값은 다음과 같이 계산합니다.
```
#<immediate> = #<imm8> ROR (2 * #<rotate_imm>)
```

**즉, 8비트 값 하나를 최대 30비트까지 오른쪽 순환시켜 만들 수 있는 값만 표현 가능합니다.**

> 참고: 8비트 값을 2의 배수만큼 오른쪽 회전 시프트(ROR)를 했을 때:
> ```
> #<imm_8> ROR <rotate_amount>
> <rotate amount> = 2 * #<rotate_imm>
> 
> 0x00 <= #<imm_8> <= 0xff(255)
> 0x0 <= #<rotate_imm> <= 0xf(15)
> ```

### 예제: 유효한 값
```armasm
mov r0, #0x104
```
- `0x104` = `0b0000_0000_0000_0000_0000_0001_0000_0100`
- 직접 8비트 범위에 담을 수 없으므로 회전시프트 검사
- `0x104`를 왼쪽으로 2비트 회전시프트: `0b0000_0000_0000_0000_0000_0100_0001_0000`
  - 비트[11:4]의 `0b0100_0001` 만큼은 8비트로 표현 가능
- 따라서 `0x104`를 오른쪽으로는 30비트 회전시프트해야 하므로, `imm_8=0x401`이고 `rotate_imm=15`

### 예제: 유효하지 않은 값
```armasm
mov r0, #0x101
```
- `0x101`=`0b0000_0000_0000_0000_0000_0001_0000_0001`
- 직접 8비트 범위에 담을 수 없으므로 회전시프트 검사
- 모든 `rotate_imm`값으로 회전해도 8비트 값을 만들 수 없음
- 따라서 인코딩 불가능 -> 오류 발생

### 표현 불가능한 상수 처리 방법
앞의 예제에서와 같이 `0x101`과 같은 수는 유효하지 않은 상수이므로 이진데이터로 인코딩 되지 않습니다.
이럴 때는 보통 리터럴 풀(Literal Pool) 을 사용합니다.
즉, 상수를 코드 근처에 저장하고 ldr 명령으로 불러옵니다.

**literal-pool.s**
```armasm
  .text
  .global _start
_start:
  ldr r0, =0x101   @ 어셈블러가 리터럴 풀에 0x101을 저장하고 로드
  b .
```

위 코드를 objdump로 보면, 실제로는 `ldr r0, [pc, #offset]` 형식으로 변환됩니다.
```bash
# 컴파일
$ arm-none-eabi-gcc \
    -Ttext=0x10000 \
    -nostdlib \
    literal-pool.s \
    -o literal-pool.elf

# ELF 디스어셈블
$ arm-none-eabi-objdump -D literal-pool.elf
00010000 <_start>:
   10000:       e51f0000        ldr     r0, [pc, #-0]   @ 10008 <_start+0x8>
```

> 메모리 접근과 리터럴 풀에 관한 내용은 향후 포스팅에서 다룰 예정입니다.
> 
> 그 외에는 다른 시프트 명령어와 조합하여 사용할 수도 있지만, 상세한 내용은 건너뛰겠습니다.
> 혹시 궁금하신 분은 댓글로 알려주세요.

#### 인코딩 불가능한 상수를 사용하는 C 코드 예제
그럼 실제 C코드에서 인코딩 불가능한 상수값을 사용할 때, 어떤 어셈블리 코드가 생성되는지 궁금해집니다.
아래의 예제코드를 보겠습니다.
 
**invalid-imm.c**
```c
int main(void) {
  int x = 0x101;
  return x;
}
```
이를 `arm-none-eabi-gcc`로 컴파일하면:
```bash
$ arm-none-eabi-gcc -fomit-frame-pointer -nostdlib -S invalid-imm.c
```

```armasm
main:
  sub sp, sp, #8
  ldr r3, .L3
  @...
.L3:
  .word 257
```
와 같은 형태로 어셈블리 코드가 생성됩니다.

### 즉시값에 회전(ROR) 인코딩을 쓰는 이유
32비트 ARM프로세서에서 모든 명령어 크기는 32비트입니다.
32비트 영역안에서 opcode와 레지스터 정보등을 모두 사용해야 하죠.
만약 즉시값에 32비트를 모두 사용해서 표현하면 명령어 전체가 64비트가 되어야 합니다.
그럼 다른 명령어들과 일관성이 떨어지고 명령어가 복잡해져 ARM의 설계철학과 맞지 않기 때문에 절충안으로 사용하는 것 입니다.

또한 우리가 프로그래밍할 때 사용하는 대부분의 상수값을 생각해보면 아래의 3가지 정도입니다.
- 작은 상수(예: 0, 1, 2, 255)
- 비트 마스크(예: 0xff00, 0x8000)
- 포인터 오프셋

이런 값들은 회전 인코딩으로 대부분 표현 가능합니다.
즉, 공간 절약 + 충분한 표현력을 동시에 얻기 위한 방식입니다.

## 참고: 왜 시프트 피연산자가 존재할까?
ARM 아키텍처의 설계 철학은 간결함과 효율성입니다.
데이터 처리 명령어 하나만으로도 흔히 필요한 비트 이동 + 연산을 동시에 수행할 수 있도록 만들었습니다.

<figure>
  <img src="/assets/img/arm-barrel-shifter.png" style="display: block; margin: auto;" alt="ARM 배럴시프터 구조">
  <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    ARM7TDMI 내부 구조 중 일부. 데이터 처리 명령어의 두 번째 피연산자는 
    ALU로 전달되기 전에 항상 <b>배럴 시프터(Barrel Shifter)</b>를 거칩니다. 
    이 덕분에 추가 명령어 없이도 시프트 연산과 산술/논리 연산을 동시에 수행할 수 있습니다.
  </figcaption>
</figure>

ARM7TDMI와 같은 ARMv4T CPU 구조를 보면,
ALU 입력단에 배럴 시프터(Barrel Shifter) 가 연결되어 있습니다.

즉, ALU에 값을 넣기 전에 자동으로 시프트/회전을 처리할 수 있어서:

- 불필요한 추가 명령어 감소
- 코드 밀도(code density) 향상
- 성능 최적화

라는 장점을 가집니다.
