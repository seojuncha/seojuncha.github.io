---
layout: post
lang: ko
ref: "linux-process-memory"
title: "프로세스 메모리 구조와 가상 주소"
date: 2025-11-27 20:20:00 +0900
categories: ["linux"]
tags: []
published: false
---

리눅스에서 동작하는 모든 프로그램은 프로세스의 형태로 존재합니다.
프로세스는 프로그램을 실행하기 위해 필요한 모든 실행 문맥을 담는 커널 내부의 구조체입니다.
여기서 실행 문맥이란 것은 CPU 레지스터 상태, 프로그램 카운터, 스택/힙/전역변수 등 프로세스를 다시 실행시키는데 필요한 모든 정보를 포함합니다.
각각의 프로세스는 본인의 실행문맥을 가지고 있고, 리눅스 OS는 이것들을 제어하여 문맥전환, 스케쥴링 등에 활용할 수 있습니다.

이번 포스팅에서는 실행 문맥을 저장하는 프로세스의 가상 주소 공간에 대한 내용을 실제 예제와 함께 자세히 분석해보겠습니다.

> 본 포스팅은 리눅스 OS기반이며 테스트에 사용한 버전은 다음과 같습니다.
> OS: 
> 커널 버전:
> GCC 버전:

## 가상 주소과 물리 주소
리눅스의 프로세스와 주소 공간에 관해 다루기 위해서는 가장 먼저 가상 주소와 물리 주소의 개념을 이해해야 합니다.

현대의 컴퓨터에서 동작하는 프로그램은 기본적인 하드웨어 구성요소인 CPU, 메모리와 밀접한 연관이 있습니다.
CPU는 결국 메모리에 존재하는 데이터를 읽고, 연산하고, 쓰는 동작을 하기 때문입니다.

앞서 설명한 프로세스의 실행 문맥 역시 메모리에 저장되어 있습니다.
리눅스는 메모리 관리의 효율성, 안전성, 그리고 유연성을 위해 가상 주소와 물리 주소를 구분하여 사용합니다.

### 물리 주소(PA: Pysical Address)
물리 주소는 실제 하드웨어 메모리에 접근할 수 있는 주소를 의미합니다.
하드웨어 레벨에서 메모리는 우리가 흔히 사용하는 RAM이 물리 메모리이고, 이 물리 메모리에 접근하기 위한 주소가 바로 물리 주소입니다.

RAM 메모리는 물리적인 장치로 그 크기와 주소가 고정되어 있습니다.
예를 들어, 32비트 시스템에서 4GB RAM 메모리는 `0x0000_0000` 부터 `0xFFFF_FFFF`까지의 주소를 가질 수 있습니다.

> 32비트 시스템은 [메모리 버스]()의 크기가 32비트 이기 때문에 표현할 수 있는 주소의 범위는 0~(2^32-1) 입니다.

OS가 동작하는 환경에서, 사용자는 보안상/안전상의 문제를 야기할 수 있는 직접적인 물리 메모리 접근을 하지 않습니다.
그래서 가상 메모리 영역과 가상 주소의 개념이 등장합니다.

### 가상 주소(VA: Virtual Address)
가상 주소는 각각의 프로세스마다 독립된 메모리 주소를 의미합니다.
프로세스가 생성되면 본인이 사용할 가상의 메모리 공간을 할당 받고 해당 영역 안에서만 프로그램과 연관된 동작을 할 수 있습니다.

또한, 프로세스마다 독립된 *가상의 주소*를 할당 받으므로 각각의 프로세스가 동일한 가상 메모리 주소를 가질 수도 있습니다.
컴파일러는 이런 내용은 [ELF포맷]()의 이진 파일로 생성합니다.

> 향후 포스팅에서 ELF포맷의 프로세스 메모리 구조가 어떻게 가상주소 공간에 매핑되는지 검증해 볼 예정입니다!

### 프로세스의 가상주소 공간 확인
프로세스 생성 시에 리눅스 커널은 `PID`(Process ID)라는 프로세스 고유 번호를 부여합니다.
`PID`는 각각의 프로세스를 구분하기 위한 고유 번호로써 `/proc/[pid]/maps`파일에는 해당 프로세스의 가상주소 공간에 대한 정보가 들어있습니다.

```
$ cat /proc/[pid]/maps
```

여기서는 maps파일에 관한 자세한 내용은 다루지 않습니다.
실제로 우리가 작성한 코드가 어떻게 프로세스의 가상 메모리 공간에 할당되는지 이해하고 검증하는 것이 목표이기 때문입니다.

maps파일을 읽는 방법에 관한 내용은 `man 5 proc`페이지를 참고해주세요.

## 실험 예제
이 예제에서는 malloc으로 할당한 공간 혹은 지역변수 등이 어떤 가상 주소 공간에 위치하는지 직접 확인해 보겠습니다.

**main.c**
```c
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char global_var_1;
char global_var_2;
char global_var_init_1 = 'A';
char global_var_init_2 = 'B';

int print_maps(void) {
  FILE *f = fopen("/proc/self/maps", "r");
  char buf[256];

  if (!f) {
    perror("fopen");
    return -1;
  }

  while (fgets(buf, sizeof(buf), f)) {
      fputs(buf, stdout);
  }

  return 0;
}

void foo(char a, char b) {
  printf("[%p] address of the first parameter in function\n", &a);
  printf("[%p] address of the second parameter in function\n", &b);
}

int main(void) {
  int var_in_stack_1;
  int var_in_stack_2;
  int *addr_var_in_heap;
 
  print_maps();
  
  printf("[%p] address of the first local variable\n", &var_in_stack_1);
  printf("[%p] address of the second local variable\n", &var_in_stack_2);
  
  addr_var_in_heap = malloc(4 * sizeof(int));
  printf("[%p] address of the variable allocated by malloc()\n", addr_var_in_heap);
  
  printf("[%p] address of the function\n", foo);
  foo(1, 2);
 
  printf("[%p] address of the uninitialized global variable 1\n", &global_var_1);
  printf("[%p] address of the uninitialized global variable 2\n", &global_var_2);
  printf("[%p] address of the initialized global variable 1\n", &global_var_init_1);
  printf("[%p] address of the initialized global variable 2\n", &global_var_init_2);
 
  return 0;
}
```

컴파일 & 실행:
```bash
$ gcc -O0 main.c & ./a.out
```

출력 결과
<터미널 캡쳐화면>

### 실험 결과 분석
예제코드에서는 총 5개의 변수에 대한 가상 메모리 주소를 출력했습니다.
- main 함수내 지역변수 주소
- malloc으로 할당된 주소
- 함수 주소
- 초기화 되지 않은 전역 변수의 주소
- 초기화 된 전역 변수의 주소

`printf`로 출력한 주소를 `/proc/[pid]/maps`파일의 출력과 비교해보면 다음과 같습니다.
| 항목 | 변수/주소 | 할당 영역 |
| - | - | - |
| 지역변수 | | |
| malloc | | |
| 함수 주소 | | |
| 함수 인자 | | |
| 초기화 X 전역 변수 | | |
| 초기화 O 전역 변수 | | |


> malloc으로 할당한 공간이 [heap] 주소 범위에 포함되지 않는 경우도 있습니다.
> 보통 소량의 크기 일때만 [heap] 주소 영역에 할당하고 일정 크기()이상인 경우에는 별도의 메모리 공간을 할당합니다.
> 자세한 시뮬레이션은 [GitHub 링크]()를 참고해주세요.

#### 참고: 왜 2개의 지역변수의 주소는 증가할까?
쉽게 혼돈할 수 있는 부분이 하향식 스택의 주소 감소 개념과 스택에 할당되는 지역변수의 주소 증가에 대한 불일치입니다.
예제 코드에 2개의 지역변수가 있습니다.
각 지역변수의 주소를 출력했을 때, 뒤에 선언된 변수의 주소가 먼저 선언된 변수의 주소보다 큰 것을 확인할 수 있습니다.
이 때, 이것이 하향식 스택의 동작에 위배되는 것이 아닌가 하는 의문이 발생합니다.
 
프로세스 메모리 구조에서 스택은 주소가 감소하는 방향으로 자랍니다.
이것은 프로그램 실행 시 SP(스택 포인터)가 감소한다는 것을 의미할 뿐, 지역변수의 선언 순서대로 스택에 할당되는 것을 뜻하지 않습니다.
지역변수의 가상 주소 할당은 전적으로 컴파일러의 역할입니다.


## 참고
- [`/proc/<pid>/maps` 블로그 자료](https://www.baeldung.com/linux/proc-id-maps)
- [linux kernel doc]()
- [man 5 proc]()
- [glibc doc]()
- [ELF doc]()