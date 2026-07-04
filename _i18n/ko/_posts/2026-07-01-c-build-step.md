---
layout: post
lang: ko
ref: "c-build-step"
title: "C 컴파일 4단계"
date: 2026-07-01 21:00:00 +0900
categories: ["programming"]
tags: ["c compile", "c programming", "c build"]
---

간단한 샘플코드를 사용해서 C코드가 실행가능한 파일로 변환되는 과정을 자세히 살펴본다.

사실 이 과정을 모르지 않지만, 스스로 설명하는 과정에서 각 과정의 개념과 분석방법에 더 익숙해 지기 위함이다.

실습환경은 AMD64/리눅스 기반이다. GNU 프로그램과 LLVM 프로그램을 혼용해서 사용하고 있음을 참고하자.
아주 오래된 툴체인을 사용하지 않는다면 대부분 환경에서 무리 없이 실습가능할 것 이다.

## 주요 용어
**이식성(Portable)**  
코드의 수정없이 다른 플랫폼에서 실행가능한 것을 의미한다.
**C 코드자체는 portable이다**. 동일한 소스코드를 AMD64 아키텍쳐와 arm64 아키텍쳐 모두에서 사용할 수 있기 때문이다.
하지만 **컴파일된 실행파일은 portable이 아니다**. AMD64용으로 컴파일된 실행파일을 AArch64에서 실행할 수는 없기 때문이다.

**해석유닛(Translation Unit)**  
컴파일러가 해석하는 하나의 소스파일을 의미한다. 소스파일 마다 각각 Translation Unit(TU)이 존재.
의미그대로 컴파일러 입장에서의 **해석 단위**이다.

**명령어 셋(Instruction Set)**  
CPU가 처리하는 명령어 집합이다. **CPU는 이진코드 형태의 각종 산술/논리 연산 등을 해석해서 어떤 작업을 해야할지 알 수 있다**.
예를 들어, 이진코드 `1111`은 덧셈연산을, 이진코드 `1100`은 뺄셈연산을 해야한다고 미리 정의해 놓는 것 과 같다.
문제는 CPU 종류마다 이 맵핑이 모두 다르다는 것이다. 

**오브젝트 파일 포맷(Object File Format)**  
오브젝트 파일은 그 종류와 무관하게 결국 하드웨어가 실행가능한 이진코드 명령어 혹은 데이터 집합이다.
현대 리눅스에서는 **ELF(Executable and Linkable Format)**를 기본 오브젝트 파일 포맷으로 사용한다. 


## 컴파일(빌드) 파이프라인
C 소스파일이 실행가능한 오브젝트 파일로 변환되는 과정은 일종의 파이프라인 형태다. 
크게 총 4개의 단계로 분류할 수 있으며 각 단계는 이전 단계의 출력을 입력으로 사용하기 때문에 파이프라인으로 정의할 수 있다.

**컴파일 파이프라인 4단계:**
- 전처리(Preprocessing)
- 컴파일(Compilation)
- 어셈블리(Assembly)
- 링킹(Linking)

아래의 간소화된 샘플코드를 사용해서 어떻게 컴파일 전 과정이 이루어지는지 확인해보자.

**sum.h**
{% highlight c linenos %}
#ifndef SUM_H
#define SUM_H

int sum(int a, int b);

#endif
{% endhighlight %}

**sum.c**
{% highlight c linenos %}
#include "sum.h"

#define WORK 1
#define PLUS +

typedef int i32;

/* block comment */
i32       sum(i32 a,    i32 b) {   // line comment
#if WORK
    return a PLUS b;
#else
    return 0;
#endif
}
{% endhighlight %}

**main.c**
{% highlight c linenos %}
#include "sum.h"

int main(void) {
  return sum(3, 5);
}
{% endhighlight %}

## 단계 1. 전처리
- **입력**: C 소스 파일
- **출력**: 전처리된 소스 파일
- **전처리기**: `cpp`, `clang-cpp`

첫번째 단계는 전처리 과정이다. 다른 대부분의 언어에서는 제공하지 않는 C/C++ 고유의 특징이다.

전처리 과정에서는 크게 다음 4가지 일을 수행한다.
1. 헤더파일 포함
2. 매크로 확장
3. 조건부 컴파일
4. 소스 코드 정리

보통 `gcc`와 같은 툴을 사용할 경우 크게 신경쓰지 않을 수 있지만, `gcc`는 컴파일 파이프라인의 각 단계를 편리하게 처리하기 위한 통합 툴 정도이고
실제로는 각 단계마다 별도의 툴이 사용된다. 이 중, 전처리 단계에서 사용되는 것이 `cpp` 다.

{% highlight bash %}
$ gcc -E sum.c -o sum.i   # gcc 를 사용할 경우 -E 옵션으로 전처리된 결과 확인 가능
$ cpp sum.c -o sum.i
{% endhighlight %}

`sum.c`파일을 전처리한 결과로 생성한 `sum.i` 을 살펴보고 전처리 과정에서 처리한 것들을 보자.

**sum.i**
{% highlight c linenos %}
# 0 "sum.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 0 "<command-line>" 2
# 1 "sum.c"
# 1 "sum.h" 1



int sum(int a, int b);
# 2 "sum.c" 2




typedef int i32;


i32 sum(i32 a, i32 b) {

    return a + b;



}
{% endhighlight %}

**헤더파일 포함**
- `#include "sum.h"` 문이 해당 헤더파일의 내용으로 치환되었다.

**매크로 확장**
- `#define PLUS +` 에 의해서 `return a PLUS b;` 문이 `return a + b;` 로 확장되었다.

**조건부 컴파일**
- `#define WORK 1` 과 `#if WORK, #else, #endif` 조건에 의해 `return 0;` 가 제거되었다.

**소스코드 정리**
- `i32       sum(i32 a,    i32 b)` 에 포함된 토큰사이 불필요한 공백이 제거되었다.
- 주석도 제거되었다. 주석은 프로그래머를 위한 것이지 실행에 관련이 없기 때문에 미리 제거한다.

참고로 `# 2 "sum.c" 2`와 같은 줄은 라인마커(linemarkers)라고 한다.

유의해야 할 점은, 전처리 과정은 C 문법 기반의 파싱을 포함하지 않는다.

### 전처리는 문법기반 파싱을 하지 않는다는 증거
아래의 코드를 전처리한 결과를 살펴보면, 전처리기는 C문법은 안보지만 전처리 지시어 문법은 검사한다는 것을 볼 수 있다.

**pp.h**
{% highlight c linenos %}
#include <stdio.h>

#define text 100

This is my text.

{% endhighlight %}

{% highlight bash %}
$ cpp pp.h -o pp.i
{% endhighlight %}

`pp.h` 헤더파일은 `#include`문과 `#define`, 그리고 하나의 일반 문장을 포함하고 있다.
`This is my text.` 와 같은 단일 문장은 파싱되지 않는 문법이지만, 전처리 과정에서는 C 문법을 파싱하지 않으므로 별다른 오류메세지를 생성하지 않는다.

다만, 전처리 동작에 따라 `text`가 `100`으로 치환되는 것은 `pp.i` 파일의 가장 아래부분에서 확인할 수 있다.

## 단계 2. 컴파일
- **입력**: 전처리된 소스 파일
- **출력**: 어셈블리 파일
- **컴파일러**: `gcc`, `clang`

컴파일 단계에서는 전처리된 소스파일을 호스트 아키텍쳐에 맞는 어셈블리 파일로 변환해준다. 
이 말은, 최종 실행파일이 실행되는 하드웨어에 따라서 다른 어셈블리 파일을 생성한다는 것이다.

왜 어셈블리 파일은 아키텍쳐 별로 구분되어야 할까?

C언어로 작성된 프로그램은 이식성이 있다. 이식성이 있다는 것은 소스코드를 변경하지 않고도 다른 아키텍쳐(예: AMD64, AArch64, 등)에서 실행가능하다는 것이다.
이런 이식성은 바로 각각의 아키텍쳐 구분되어 생성되는 어셈블리 파일 덕분이다.

{% highlight text %}
 C Source   ────┬─  Assembly (AMD64)    ─────   Machine Code (AMD64)
                └─  Assembly (AArch64)  ─────   Machine Code (AArch64)
{% endhighlight %}

보통 지시어를 제외한 어셈블리 명령어는 하나의 CPU 머신명령에 대응한다.
플랫폼(아키텍쳐)마다 명령어 셋(Instruction Set)이 다르고 제공하는 어셈블리 명령어가 다르다.
그 뿐 아니라 CPU 레지스터 구조와 이름도 다르다. 

이것은 결과적으로 이식성을 위해서는 플랫폼마다 서로 다른 어셈블리 코드를 생성해야한다는 의미가 된다.
여기에 대한 자세한 내용은 아래 "단계 3. 어셈블리"에서 다루겟다.

어셈블리 코드를 생성하기 위해선 `gcc -S` 옵션을 사용할 수 있다.

{% highlight bash %}
$ gcc -S sum.i -o sum.s
$ gcc -S main.i -o main.s
{% endhighlight %}

파이프라인의 입출력 연결을 표현하기 위해 `.i`파일을 입력으로 사용했지만 보통 `.c`파일을 인자로 사용하여 어셈블리 파일을 생성하는 것이 일반적이다.

컴파일러는 전처리된 소스파일을 개별적인 ***TU(Translation Unit)***으로 다룬다.
TU는 컴파일러가 처리하는 하나의 단위로 생각하면 편하다.
즉, 컴파일러가 컴파일하는 하나의 소스파일 자체가 하나의 TU가 되는 것 이다.

**sum.s**
{% highlight text linenos %}
        .file   "sum.c"
        .text
        .globl  sum
        .type   sum, @function
sum:
.LFB0:
        .cfi_startproc
        endbr64
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        movl    %edi, -4(%rbp)
        movl    %esi, -8(%rbp)
        movl    -4(%rbp), %edx
        movl    -8(%rbp), %eax
        addl    %edx, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   sum, .-sum
        .ident  "GCC: (Ubuntu 11.4.0-1ubuntu1~22.04.3) 11.4.0"
        .section        .note.GNU-stack,"",@progbits
        .section        .note.gnu.property,"a"
        .align 8
        .long   1f - 0f
        .long   4f - 1f
        .long   5
0:
        .string	"GNU"
1:
        .align 8
        .long   0xc0000002
        .long   3f - 2f
2:
        .long   0x3
3:
        .align 8
4:

{% endhighlight %}

생성된 어셈블리 파일에서는 `sum()` 함수의 동작을  AMD64 어셈블리어로 기술하고 있다.
- `.text`
  - 텍스트(코드) 섹션에 위치. 즉, 정적 데이터가 아닌 실행 가능한 코드
- `.globl sum`
  - 링킹 가능한 글로벌 심볼 sum
- `pushq %rbp`, `popq %rbp`
  - 함수 호출 및 반환을 위한 `rbp` 레지스터 저장과 불러오기
- `movq %rsp, %rbp`
  - 스택프레임을 생성
- `movl %edi, -4(%rbp)`, `movl %esi, -8(%rbp)`
  - `edi`, `esi` 레지스터에 저장된 2개의 함수인자를 스택에 추가
- `movl -4(%rbp), %edx`, `movl -8(%rbp), %eax`
  - 스택에 저장된 함수 인자를 `edx`, `eax`레지스터로 읽기
- `addl %edx, %eax`
  - 각 레지스터의 값을 더한다.
- `ret`
  - main 함수로 반환


그럼 컴파일러는 C소스코드를 어떻게 어셈블리 코드로 변환할 수 있었을까?
이 과정이 바로 컴파일러 이론의 핵심이다. 하지만, 이번 포스팅의 범위를 벗어나기 때문에 다른 포스팅에서 더 자세히 다뤄보겠다.

**C 문법에 따르는지는 어떻게 알까?**  
예전에 [C컴파일러](https://github.com/seojuncha/ftt-c-compiler)를 만들면서, 이론적인 부분을 제외하고, 첫번째로 궁금했던 점은 C문법에 따라 토큰을 나누고 분류하는 방법이었다.

우리는 강의 혹은 교재등을 통해서 C문법을 익히고 C 프로그램을 작성하지만 직접 컴파일러를 만들기 위해서는 이걸로 충분하지 않았다.
C언어는 [ISO 문서](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf)에 그 문법이 상세히 기술되어있다.

### 플랫폼 의존적인 어셈블리 코드
동일한 C코드와 동일한 컴파일러를 사용하더라도 서로 다른 어셈블리 코드를 생성할 수 있음을 확인할 수 있다.

예를 들어, AArch64 아키텍쳐를 타겟한 어셈블리 코드를 보자.

{% highlight bash %}
$ aarch64-none-linux-gnu-gcc -S sum.c -o sum_arm64.s

$ cat sum_arm64.s

        .arch armv8-a
        .file   "sum.c"
        .text
        .align  2
        .global sum
        .type   sum, %function
sum:
.LFB0:
        .cfi_startproc
        sub     sp, sp, #16
        .cfi_def_cfa_offset 16
        str     w0, [sp, 12]
        str     w1, [sp, 8]
        ldr     w1, [sp, 12]
        ldr     w0, [sp, 8]
        add     w0, w1, w0
        add     sp, sp, 16
        .cfi_def_cfa_offset 0
        ret
        .cfi_endproc
.LFE0:
        .size   sum, .-sum
        .ident  "GCC: (Arm GNU Toolchain 15.2.Rel1 (Build arm-15.86)) 15.2.1 20251203"
        .section        .note.GNU-stack,"",@progbits

{% endhighlight %}

동일한 `sum.c` 파일을 사용하더라도 다른 어셈블리 코드를 출력한다.


## 단계 3. 어셈블리
- **입력**: 어셈블리 파일
- **출력**: 재배치 가능한 오브젝트 파일(Relocatable Object File)
- **어셈블러**: `as`, `llvm-as`

컴파일러 벡엔드의 CodeGen모듈에 의해 생성된 어셈블리 파일은 플랫폼에 맞는 어셈블러(`as`)가 머신코드로 해석해준다.
머신코드는 플랫폼마다 서로 다른 명령어 셋(instruction set)을 표현한다.

{% highlight bash %}
$ as sum.s -o sum.o
$ as main.s -o main.o
{% endhighlight %}

지금까지 다루었던 전처리, 컴파일, 어셈블리 단계까지 모두 한번에 할 수 있는 것이 `gcc -c` 명령어다.
{% highlight bash %}
$ gcc -c sum.c -o sum.o
$ gcc -c main.c -o main.o

$ file main.o

main.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
{% endhighlight %}

생성된 재배치 가능한 오브젝트 파일은 어셈블리 명령에 따라 타겟 아키텍쳐와 호환되는 이진 코드를 포함한다.

`objdump -d` 명령으로 `sum.o` 와 `main.o` 오브젝트 파일의 디스어셈블리 결과를 확인해보자.

{% highlight bash %}
$ objdump -d sum.o

sum.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <sum>:
   0:   f3 0f 1e fa             endbr64
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   89 7d fc                mov    %edi,-0x4(%rbp)
   b:   89 75 f8                mov    %esi,-0x8(%rbp)
   e:   8b 55 fc                mov    -0x4(%rbp),%edx
  11:   8b 45 f8                mov    -0x8(%rbp),%eax
  14:   01 d0                   add    %edx,%eax
  16:   5d                      pop    %rbp
  17:   c3                      ret
{% endhighlight %}

아래 표와 같이 각 어셈블리 라인은 하나의 이진 실행코드와 매핑된다.

| 어셈블리 | 이진 코드 |
| - | - |
| `endbr64` | `0xF30F1EFA` |
| `push %rbp` | `0x55` |
| `mov %rsp, %rbp` | `0x4889E5` |
| `mov %edi,-0x4(%rbp)` | `0x897DFC` |
| `mov %esi, -0x8(%rbp)` | `0x8975F8` |
| `mov -0x4(%rbp), %edx` | `0x8B55FC` |
| `mov -0x8(%rbp), %eax` | `0x8B45F8` |
| `add %edx, %eax` | `0x01D0` |
| `pop %rbp` | `0x5D` |
| `ret` | `0xC3` |


{% highlight bash %}
$ objdump -d main.o

main.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:   f3 0f 1e fa             endbr64
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   be 05 00 00 00          mov    $0x5,%esi
   d:   bf 03 00 00 00          mov    $0x3,%edi
  12:   e8 00 00 00 00          call   17 <main+0x17>
  17:   5d                      pop    %rbp
  18:   c3                      ret
{% endhighlight %}

main함수의 어셈블리 코드 `endbr64`나 `push %rbp`와 같은 이진코드는 sum함수에서 봤던 이진코드와 정확히 일치하는 것을 볼 수 있다.

main 함수의 동작도 간략히 살펴보자. (sum함수와 동일한 부분은 제외했다.)
- `mov    $0x5,%esi`
  - 정수 5를 `esi` 레지스터에 저장
- `mov    $0x3,%edi`
  - 정수 3을 `edi` 레지스터에 저장
- `call   17 <main+0x17>`
  - `call` 명령은 뒤에 나오는 주소의 함수를 호출하지만, 아직 재배치되지 않았으므로 다음 명령어 주소를 가리킴

AArch64용으로 컴파일된 `sum_arm64.o`도 디스어셈블리 해보자. 한눈에 봐도 AMD64 오브젝트 파일과 다른 것을 볼 수 있다.

{% highlight bash %}
$ aarch64-none-linux-gnu-objdump -d sum_arm64.o

sum_arm64.o:     file format elf64-littleaarch64


Disassembly of section .text:

0000000000000000 <sum>:
   0:   d10043ff        sub     sp, sp, #0x10
   4:   b9000fe0        str     w0, [sp, #12]
   8:   b9000be1        str     w1, [sp, #8]
   c:   b9400fe1        ldr     w1, [sp, #12]
  10:   b9400be0        ldr     w0, [sp, #8]
  14:   0b000020        add     w0, w1, w0
  18:   910043ff        add     sp, sp, #0x10
  1c:   d65f03c0        ret
{% endhighlight %}

이것은 동일한 하드웨어 동작이라도 아키텍쳐에 따라 호환되는 이진 실행코드와 어셈블리 코드가 다름을 증명해준다.

예를 들어,
AMD64의 덧셈연산은 `add %edx, %eax` 였지만, AArch64의 덧셈연산은 `add w0, w1, w0` 형태이다.
각각의 이진 실행 코드 역시, `0x01D0` 와 `0x0B000020` 로 차이점을 확인할 수 있다.

## 단계 4. 링킹
- **입력**: 재배치 가능한 오브젝트 파일
- **출력**: 실행 가능한 오브젝트 파일(Executable Object File), 공유 오브젝트 파일(Shared Object File)
- **링커**: `ld`, `lld`

지금까지 컴파일러가 하나의 소스파일을 하나의 TU로 처리하고 어셈블러가 각각 재배치 가능한 오브젝트 파일들을 생성했다.
이렇게 생성된 각각의 오브젝트 파일들은 이들 자체만으로는 실행할 수 없다. 
이 오브젝트 파일들은 단순히 각각의 소스코드가 어떤 머신코드로 번역되었는지만을 기술할 뿐이다.
각각의 오브젝트 파일을 하나로 묶어 프로세스 메모리에 로딩될 수 있도록 하는 것이 링킹이고 이것이 링커(Linker)의 역할이다.

그럼 여러 오브젝트 파일에 정의된 데이터를 어떻게 하나로 취합할 수 있을까?

링커는 각 오브젝트 파일이 가지고 있는 심볼(Symbol)이라는 일종의 라벨을 사용해서 실제 이진 코드를 연결한다.

이 외에도 링커는 링킹과정에서는 다양한 처리를 하므로, `ld`를 직접 사용해서 링킹하는 것은 다른 포스팅에서 다룰 예정이다.
지금은 간편하게 `gcc`만 사용해서 링킹해본다.

{% highlight bash %}
$ gcc main.o sum.o   # 링킹 후, a.out 생성

$ file a.out
a.out: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),  .. <생략>

$ ./a.out || echo $?
8                       # sum호출 결과인 8을 반환
{% endhighlight %}

{% highlight bash %}
$ objdump -d a.out

...<생략>...

0000000000001129 <sum>:
    1129:       f3 0f 1e fa             endbr64
    112d:       55                      push   %rbp
    112e:       48 89 e5                mov    %rsp,%rbp
    1131:       89 7d fc                mov    %edi,-0x4(%rbp)
    1134:       89 75 f8                mov    %esi,-0x8(%rbp)
    1137:       8b 55 fc                mov    -0x4(%rbp),%edx
    113a:       8b 45 f8                mov    -0x8(%rbp),%eax
    113d:       01 d0                   add    %edx,%eax
    113f:       5d                      pop    %rbp
    1140:       c3                      ret

0000000000001141 <main>:
    1141:       f3 0f 1e fa             endbr64
    1145:       55                      push   %rbp
    1146:       48 89 e5                mov    %rsp,%rbp
    1149:       be 05 00 00 00          mov    $0x5,%esi
    114e:       bf 03 00 00 00          mov    $0x3,%edi
    1153:       e8 d1 ff ff ff          call   1129 <sum>
    1158:       5d                      pop    %rbp
    1159:       c3                      ret
{% endhighlight %}

`sum`과 `main`에 정의된 이진 실행코드가 하나의 `a.out`파일에 모두 삽입된 것을 볼 수 있다.

### 오브젝트 파일 심볼
링킹 전에 생성된 재배치 가능한 오브젝트 파일들(`sum.o`, `main.o`)에 정의된 심볼을 `nm`명령어를 사용해서 확인해보자.

> `nm`은 오브젝트 파일에 정의된 심볼을 보여주는 GNU 유틸리티 프로그램이다. 아래 함수 이름 왼쪽의 알파벳의 의미는 `man nm`을 확인하자.

{% highlight bash %}
$ nm sum.o main.o

sum.o:
0000000000000000 T sum

main.o:
0000000000000000 T main
                 U sum
{% endhighlight %}

`main.o`(`main.c`를 컴파일한 오브젝트 파일) 에는 `main`과 `sum` 2개의 심볼이 있다.
하지만, `sum`심볼은 `U`이므로 *Undefined* 상태다. 정의되어 있지 않은 심볼이란 뜻이다.

왜냐면 **sum**이란 이름으로 라벨링(심볼)된 `sum`함수의 이진 실행 코드, 즉 함수의 정의는 `sum.o`파일에 정의되어있고 `main.o`파일에 정의된게 아니기 때문이다.

그래서 링커는 서로 다른 위치에 있는 이진 실행코드를 합치기 위해 심볼 정보들을 모아야 한다.
최종 생성된 실행가능한 오브젝트 파일에서 다시 심볼을 확인해보자.

{% highlight bash %}
$ nm a.out

0000000000001141 T main
0000000000001129 T sum
{% endhighlight %}

`sum`심볼이 `T`(`.text`섹션에 있는 실행가능한 코드)로 변경되었다.
