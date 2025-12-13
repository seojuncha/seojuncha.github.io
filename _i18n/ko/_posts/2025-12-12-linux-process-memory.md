---
layout: post
lang: ko
ref: "linux-process-memory"
title: "프로세스 메모리 구조와 가상주소의 기본"
date: 2025-12-12 20:20:00 +0900
categories: ["linux"]
tags: ["process", "virtual address", "maps"]
published: false
---

리눅스에서 동작하는 모든 프로그램은 프로세스의 형태로 존재합니다.
프로세스는 프로그램을 실행하기 위해 필요한 모든 실행 문맥을 커널 내부 자료 구조를 통해 관리되는 실행 단위입니다.
여기서 실행 문맥이란 CPU 레지스터 상태, 프로그램 카운터(PC), 스택/힙/전역변수 등 프로세스를 다시 실행하는 데 필요한 모든 정보를 포함합니다.
각각의 프로세스는 본인의 실행 문맥을 가지고 있으며, 리눅스 OS는 이를 제어하여 문맥 전환(context switch), 스케줄링(scheduling) 등에 활용합니다.

이번 포스팅에서는 실행 문맥을 담는 프로세스의 **가상 주소 공간**을 실제 예제와 함께 분석해 보겠습니다.

본 포스팅은 리눅스 OS 기반이며 테스트에 사용한 버전은 다음과 같습니다.

- OS: 우분투 24.04
- 커널: 6.8.0
- GCC: 13.3.0

> 이 글은 리눅스 프로세스의 가상 주소 공간을 실제 실험을 통해 분석한 기록입니다.
> 기본 개념을 설명하지만, 증명 과정과 한계도 함께 다루기 때문에 입문자보다는 시스템 프로그래밍에 관심 있는 독자를 대상으로 합니다.

## 가상 주소와 물리 주소
리눅스의 프로세스와 주소 공간을 이해하려면 먼저 가상 주소와 물리 주소의 개념을 구분해야 합니다.
현대 컴퓨터에서 프로그램은 CPU와 메모리와 밀접하게 연결되어 있습니다.
CPU는 결국 메모리에 존재하는 데이터를 읽고, 연산하고, 다시 쓰는 방식으로 동작하기 때문입니다.
앞서 언급한 실행 문맥 역시 메모리에 저장됩니다.
**리눅스는 메모리 관리의 효율성, 안전성, 유연성을 위해 가상 주소와 물리 주소를 구분하여 사용합니다**.

### 물리 주소 (PA: Physical Address)
물리 주소는 **실제 하드웨어 메모리에 접근할 수 있는 주소**를 의미합니다.
하드웨어 레벨에서 우리가 흔히 말하는 RAM이 물리 메모리이며, 이 물리 메모리에 접근하기 위한 주소가 물리 주소입니다.
예를 들어 32비트 시스템에서는 주소를 표현하는 데 32비트를 사용하므로, 이론적으로 `0x0000_0000`부터 `0xFFFF_FFFF`까지의 주소 범위를 표현할 수 있습니다.
OS 환경에서 사용자는 보안 및 안정성 문제를 야기할 수 있는 직접적인 물리 메모리 접근을 일반적으로 수행하지 않습니다.
그래서 가상 메모리와 가상 주소의 개념이 필요해집니다.

### 가상 주소 (VA: Virtual Address)
가상 주소는 프로세스마다 **독립적으로 보이는 메모리 주소**를 의미합니다.
프로세스가 생성되면 자신이 사용할 가상의 주소 공간을 할당받고, 그 범위 안에서만 프로그램과 관련된 동작을 수행합니다.
또한 프로세스마다 독립된 가상 주소를 가지므로, 서로 다른 프로세스가 동일한 가상 주소 값을 가질 수도 있습니다.
**ASLR**(Address Space Layout Randomization)이 비활성화된 환경이나 특정 조건에서는 이러한 현상이 더 쉽게 관찰될 수 있습니다.

중요한 점은, **가상 주소가 같아 보이더라도 실제로 매핑되는 물리 주소는 서로 다를 수 있다는 것**입니다.
예를 들어 프로세스 A에서 사용하는 변수 `a`의 주소와 프로세스 B에서 사용하는 변수 `b`의 주소가 동일하게 출력될 수도 있습니다.

하지만 이 경우에도 두 변수가 실제 물리 메모리에서 같은 위치를 가리킨다고 단정할 수는 없습니다.

**process_A.c**
```c
int a;
printf("%p", &a);
```

**process_B.c**
```c
int b;
printf("%p", &b);
```

#### 참고: GDB로 출력하는 주소는 가상 주소
유저 영역에서 실행되는 프로세스가 출력하는 주소 값은 모두 가상 주소입니다.
GDB로 디버깅할 때 출력하는 주소 또한 동일합니다.

예를 들어 GDB 세션에서 다음과 같이 변수의 주소를 출력할 때,

```bash
(gdb) p/32x &local_var
```

여기서 `&local_var`는 해당 프로세스의 가상 주소 공간 안에서의 주소(가상 주소)입니다.

## 프로세스 메모리 구조
리눅스에서 생성된 모든 프로세스는 OS로부터 할당받은 가상 주소 공간에 프로그램 코드와 데이터를 로드합니다.
이 가상 주소 공간은 일반적으로 몇 개의 대표적인 구역(세그먼트/영역)으로 구분하여 설명합니다.
일반적으로 각 프로세스는 독립된 가상 주소 공간을 가지지만, `text` 영역이나 공유 라이브러리처럼 물리 메모리를 공유하는 경우도 있습니다.

| 세그먼트  | 설명                                       |
| ----- | ---------------------------------------- |
| text  | 컴파일된 실행 코드를 저장하는 영역                      |
| data  | 초기화된 전역변수와 정적(static) 변수를 저장하는 영역        |
| bss   | 초기화되지 않은 전역변수와 정적 변수를 저장하는 영역            |
| stack | 지역변수, 함수 인자 등 LIFO 구조로 동작하는 데이터를 저장하는 영역 |
| heap  | `malloc()` 등을 이용한 동적 메모리 할당 영역           |

프로그램을 컴파일하면 컴파일러와 링커가 코드/데이터의 배치와 크기 정보를 포함한 바이너리를 생성합니다.
리눅스에서는 보통 이 바이너리가 ELF 포맷(Executable and Linkable Format)으로 생성됩니다.

> 향후 포스팅에서 ELF 포맷의 구조와, ELF의 섹션/세그먼트가 실제 가상 주소 공간에 어떻게 매핑되는지 검증해볼 예정입니다.

## 프로세스의 가상 주소 공간 확인
프로세스가 생성되면 리눅스 커널은 PID(Process ID)라는 고유 번호를 부여합니다.
`/proc/[pid]/maps` 파일에는 해당 프로세스의 가상 주소 공간 매핑 정보가 기록되어 있습니다.

```bash
$ cat /proc/[pid]/maps
```

이번 포스팅의 목표는 maps 파일 전체 문법을 설명하는 것이 아니라, **실제 코드가 가상 주소 공간에 어떻게 배치되는지 “관찰하고 검증”**하는 것입니다.
그럼에도 이해에 필요한 핵심 포인트 3가지만 간단히 확인하겠습니다.

아래 명령어는 현재 쉘에서 실행되는 `cat` 프로세스 기준으로 `maps`를 출력합니다.

```bash
$ cat /proc/self/maps
```

### 가상 주소 범위 (시작 주소 ~ 끝 주소)
<figure style="text-align: center;">
  <img src="/assets/img/first-column-of-stack-area-in-maps-.png"
    alt="스택 영역의 가상 주소 범위가 표시된 /proc/self/maps의 첫 번째 열"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    스택영역의 범위는 `` 부터 ``입니다.
    </figcaption>
</figure>

maps 파일의 첫 번째 열은 시작주소-끝주소 형태로 가상 주소 범위를 나타냅니다.

### 실행 권한

<figure style="text-align: center;">
  <img src="/assets/img/second-column-of-stack-area-in-maps.png"
    alt="스택 영역의 접근 권한(rw-p)이 표시된 /proc/self/maps의 권한 열"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    스택 영역의 권한은 `rw-p`이며, 읽기/쓰기 가능하고 private 매핑입니다.
    </figcaption>
</figure>


각 가상 주소 구간마다 고유한 접근 권한이 부여됩니다.
권한은 보통 rwxp 형태로 표기되며, 각각 read, write, execute, private을 의미합니다.
예를 들어 r--p는 읽기만 가능한 영역이며, 문자열 리터럴과 같은 read-only 데이터가 위치할 수 있습니다.

### 실행 경로 및 영역 타입

<figure style="text-align: center;">
  <img src="/assets/img/last-column-of-exec-binary-in-maps.png"
    alt="실행 파일 경로(/usr/bin/cat)와 라이브러리 경로가 표시된 /proc/self/maps의 마지막 열"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    실행 파일과 공유 라이브러리 매핑은 마지막 열에 경로로 표시됩니다.
    </figcaption>
</figure>

마지막 열은 실행 파일 경로, 공유 라이브러리 경로, 또는 [heap], [stack] 같은 영역 타입을 나타냅니다.

---
maps 파일을 읽는 방법을 포함해 /proc 디렉토리에 대한 더 자세한 내용은 man 페이지를 참고하시면 됩니다.
```bash
$ man 5 proc
```

<figure style="text-align: center;">
  <img src="/assets/img/part-of-linux-man-page-for-proc.png"
    alt="프로세스 관련 정보가 /proc에 제공됨을 설명하는 man 5 proc의 일부"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    `/proc`는 프로세스 관련 정보를 제공하며, maps 외에도 다양한 파일을 포함합니다.
    </figcaption>
</figure>

## 실험 예제
이 예제에서는 프로그램에서 자주 등장하는 요소들이 가상 주소 공간의 어느 영역에 위치하는지 직접 확인해 보겠습니다.

```c
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char global_var;
char init_global_var = 'A';

void print_maps(void) {
  FILE *f;
  char buf[256];

  f = fopen("/proc/self/maps", "r");
  while (fgets(buf, sizeof(buf), f))
      fputs(buf, stdout);
}

void foo(char a) {
  printf("[%p] address of the parameter in function\n", &a);
}

int main(void) {
  int local_var;
  int *malloc_var;

  print_maps();

  printf("[%p] address of the local variable\n", &local_var);

  malloc_var = malloc(sizeof(int));
  printf("[%p] address of the variable allocated by malloc()\n", malloc_var);

  printf("[%p] address of the function\n", foo);
  foo(1);

  printf("[%p] address of the uninitialized global variable\n", &global_var);
  printf("[%p] address of the initialized global variable\n", &init_global_var);

  return 0;
}
```

**컴파일 & 실행:**
```bash
$ gcc main.c & ./a.out
```

**출력 결과:**
<figure style="text-align: center;">
  <img src="/assets/img/output-of-code-sample-to-print-the-various-address.png"
    alt="maps 출력 이후 지역변수, malloc, 함수, 전역변수 주소를 출력한 터미널 결과"
    style="display: block; margin: auto;" />
    <figcaption style="margin-top: 0.5em; font-size: 0.9em; color: #666;">
    먼저 `/proc/self/maps`를 출력한 뒤, 프로그램 내부의 다양한 주소 값을 출력합니다.
    </figcaption>
</figure>


## 실험 결과 분석
예제 코드에서는 다음 항목의 주소를 출력합니다.
- `main()` 함수의 지역변수 주소
- `malloc()`으로 할당된 주소
- 함수 주소
- 초기화되지 않은 전역 변수 주소
- 초기화된 전역 변수 주소

`printf`로 출력한 주소를 `/proc/[pid]/maps`의 범위와 비교하면, 대략 다음과 같이 정리할 수 있습니다.

| 항목          | 변수/주소             | 추정 영역     |
| ----------- | ----------------- | --------- |
| 지역변수        | `local_var`       | stack     |
| malloc      | `malloc_var`      | heap      |
| 함수 주소       | `foo`             | text      |
| 함수 인자       | `a`               | stack     |
| 초기화 X 전역 변수 | `global_var`      | bss (추정)  |
| 초기화 O 전역 변수 | `init_global_var` | data (추정) |

다만 `/proc/[pid]/maps`만으로는 전역 변수가 `.bss`인지 `.data`인지 직접적으로 구분할 수는 없습니다.
실행 권한과 주소 범위를 통해 “실행 파일 매핑 영역에 포함된다” 정도까지는 관찰할 수 있지만, 섹션 단위의 증명은 ELF 분석이 필요합니다.
이 부분은 이후 ELF 분석 포스팅에서 보완할 예정입니다.
또한 함수 인자의 주소는 컴파일러와 ABI에 따라 레지스터 전달 이후 스택에 저장될 수 있습니다.

> `malloc()`으로 할당한 공간이 항상 [heap] 범위에 포함되는 것은 아닙니다.
> 일반적으로 작은 크기는 `brk` 기반 힙 확장으로 처리되는 경우가 많지만, 일정 크기 이상은 `mmap`으로 별도의 매핑 영역을 할당하기도 합니다.

## 참고
- [Linux Kernel Documentation – Memory Management](https://docs.kernel.org/admin-guide/mm/concepts.html)
- [Linux man-pages: proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html)
- [GNU C Library Manual – Memory Allocation](https://www.gnu.org/software/libc/manual/html_node/Memory-Allocation.html)
- [System V ABI – ELF Specification](https://refspecs.linuxbase.org/LSB_3.1.0/LSB-Core-generic/LSB-Core-generic/elf-generic.html)