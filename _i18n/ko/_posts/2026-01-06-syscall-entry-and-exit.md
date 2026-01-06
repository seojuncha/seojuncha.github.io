---
layout: post
lang: ko
ref: "syscall-entry-and-exit"
title: "시스템콜 진입과 복귀 구분하기: 인자와 반환값 추출 방법"
date: 2026-01-06 22:20:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall"]
---

tracer가 `PTRACE_SYSCALL`을 사용하여 시스템콜을 추적하는 경우, tracee가 시스템콜을 호출할 때 tracer는 호출 전후로 총 두 번의 시그널을 받는다.
이는 시스템콜의 핵심 동작인 유저 모드와 커널 모드 간 전환 시점에서 발생한다.
tracer는 이 두 시그널을 각각 시스템콜 진입(entry) 과 시스템콜 복귀(exit) 로 구분하고,
각 시점에 맞는 레지스터 값을 읽어 처리해야 한다.
예를 들어 시스템콜 진입 시점에 반환값 레지스터를 읽으면, 실제 처리 결과가 아닌 커널이 설정한 초기값을 사용하게 되어 의도하지 않은 결과를 만들 수 있다.

이 글에서는 시스템콜의 진입과 복귀를 구분하는 방법을 설명하고, 해당 방식이 `strace`에서 어떻게 구현되어 있는지를 간단히 분석한다.
또한 실제 레지스터 값을 읽어 시스템콜 인자와 반환값을 출력하는 방법을 다룬다.

## 시스템콜 진입과 복귀의 개념
시스템콜은 크게 다음 두 단계로 나눌 수 있다.

**시스템콜 진입 (Entry)**
- 유저 모드 → 커널 모드
- 시스템콜 인자 전달

**시스템콜 복귀 (Exit)**
- 커널 모드 → 유저 모드
- 반환값 전달

이는 시스템콜의 기본 동작 모델이므로, `PTRACE_SYSCALL` 환경에서는 각 시스템콜마다 두 번의 시그널이 관측되며,
첫 번째는 진입, 두 번째는 복귀 시점에 해당한다.
따라서 tracer 입장에서는 각 시스템콜에 대해 관측되는 첫 번째 시그널을 시스템콜 진입으로 인식할 수 있다.

### 시스템콜 진입과 복귀 구분 방식
이러한 동작 특성을 이용하면, 시스템콜 진입과 복귀를 구분하는 가장 단순하고 효율적인 방법은
토글(toggle) 방식의 플래그를 사용하는 것이다.

{% highlight c %}
int in_syscall = 0;

case SIGTRAP | 0x80:
  if (in_syscall) {  /* system call exit  */
    in_syscall = 0;
  } else {           /* system call entry */
    in_syscall = 1;
  }
{% endhighlight %}

`in_syscall`은 현재 시스템콜 처리 상태를 나타내는 플래그이다.
- `0`: 시스템콜 진입
- `1`: 시스템콜 복귀

초기값은 `0`이며, 시스템콜 진입 시 `1`로 토글하고,
다음 시그널에서 복귀 시점임을 인식한 뒤 다시 `0`으로 되돌린다.

> `SIGTRAP | 0x80` 시그널은 `PTRACE_O_TRACESYSGOOD` 옵션이 설정된 경우에만 발생한다.

### 참고: strace의 구현 방식
`strace` 역시 동일한 개념을 사용해 시스템콜 진입과 복귀를 구분한다.

**defs.h**
{% highlight c %}
#define TCB_INSYSCALL      0x04
#define entering(tcp)      (!((tcp)->flags & TCB_INSYSCALL))
#define exiting(tcp)       ((tcp)->flags & TCB_INSYSCALL)
{% endhighlight %}

**syscall.c**
{% highlight c linenos mark_lines="1" %}
int syscall_entering_finish(struct tcb *tcp, int res)
{
  tcp->flags |= TCB_INSYSCALL;
}

void syscall_exiting_finish(struct tcb *tcp)
{
  tcp->flags &= ~(TCB_INSYSCALL |
                  TCB_TAMPERED |
                  TCB_INJECT_DELAY_EXIT |
                  TCB_INJECT_POKE_EXIT |
                  TCB_TAMPERED_DELAYED |
                  TCB_TAMPERED_POKED);
}
{% endhighlight %}

**strace.c**
{% highlight c mark_lines="3 5 8" %}
static int trace_syscall(struct tcb *tcp, unsigned int *sig)
{
  if (entering(tcp)) {
    int res = syscall_entering_decode(tcp);
    syscall_entering_finish(tcp, res);
  } else {
    int res = syscall_exiting_decode(tcp, &ts);
    syscall_exiting_finish(tcp);
  }
}
{% endhighlight %}

`TCB_INSYSCALL` 플래그는 “현재 해당 스레드가 시스템콜 처리 중인가”를 나타낸다.
이 플래그는 `tcb`(Trace Control Block) 단위로 유지된다.
이는 tracer가 여러 tracee 프로세스 및 스레드를 동시에 추적할 수 있으며, 각각의 시스템콜 상태를 독립적으로 관리해야 하기 때문이다.

## 시스템콜 인자와 반환값 추출
시스템콜 인자와 반환값은 `PTRACE_GETREGS`를 통해 레지스터 값을 읽어 추출한다.
시스템콜의 동작 원리를 이해하면, 어느 시점에 어떤 레지스터를 읽어야 하는지도 자연스럽게 결정된다.

{% highlight c %}
case SIGTRAP | 0x80:
  long long ret;
  ptrace(PTRACE_GETREGS, pid, NULL, &regs);

  ret = (long long) regs.rax;

  if (in_syscall) {
    /* system call exit */
    printf("[%d] syscall exit stop: nr=%llu\n", pid, regs.orig_rax);
    printf("[%d] arg1: %llu, arg2: 0x%0llx, arg3: %llu, ret: %lld\n",
           pid, regs.rdi, regs.rsi, regs.rdx, ret);
    in_syscall = 0;
  } else {
    /* system call entry */
    printf("[%d] syscall entry stop: nr=%llu\n", pid, regs.orig_rax);
    printf("[%d] arg1: %llu, arg2: 0x%0llx, arg3: %llu, ret: %lld (%s)\n",
           pid, regs.rdi, regs.rsi, regs.rdx, ret, strerror(abs(ret)));
    in_syscall = 1;
  }
  break;
{% endhighlight %}

> 전체 코드 템플릿은 [이전 포스팅]({% post_url 2025-12-31-distinguish-syscall-signal%})에서 확인할 수 있다.

### 예제 출력 결과 분석
{% highlight text mark_lines="8 13" %}
I'm TRACER: 39539
I'm TRACEE: 39540
[39540] stopped, 19
[39540] sigstop
 
[39540] stopped, 133
[39540] syscall entry stop: nr=1
[39540] arg1: 1, arg2: 0x5ea9fa4e9018, arg3: 13, ret: -38(Function not implemented)
 
Hello, World
[39540] stopped, 133
[39540] syscall exit stop: nr=1
[39540] arg1: 1, arg2: 0x5ea9fa4e9018, arg3: 13 ,ret: 13
 
[39540] stopped, 133
[39540] syscall entry stop: nr=231
[39540] arg1: 0, arg2: 0xe7, arg3: 60, ret: -38(Function not implemented)
 
[39540] stopped, 5
[39540] traced stop
 
TRACER has terminated
{% endhighlight %}

`write` 시스템콜의 시그니처는 다음과 같다.

{% highlight c %}
ssize_t write(int fd, const void *buf, size_t count);
{% endhighlight %}

출력 결과에서 볼 수 있듯이, 시스템콜 진입 시점의 `rax` 값은 실제 반환값이 아니다.

| 시스템콜 시점 | 인자                | 레지스터  |
| ------- | ----------------- | ----- |
| 진입      | `int fd`          | `rdi` |
|         | `const void *buf` | `rsi` |
|         | `size_t count`    | `rdx` |
| 복귀      | 반환값               | `rax` |


## 참고: 왜 시스템콜 진입 시 반환값은 -ENOSYS 인가?
{% highlight asm mark_lines="2" %}
SYM_CODE_START(entry_SYSCALL_64)
  PUSH_AND_CLEAR_REGS rax=$-ENOSYS
SYM_CODE_END(entry_SYSCALL_64)
{% endhighlight %}

x86-64 커널에서는 시스템콜 진입 시 `rax`를 `-ENOSYS`로 초기화한다.
이는 시스템콜 핸들러가 존재하지 않거나, 디스패치 중 오류가 발생한 경우를 대비한 디폴트 실패값이다.
따라서 시스템콜 진입 시의 `rax` 값은 의미 있는 반환값이 아니며, tracer는 반드시 시스템콜 복귀 시점의 값만을 사용해야 한다.

## 포인터 인자의 한계
`write` 시스템콜의 두 번째 인자인 `buf`는 포인터 값이다.
레지스터에는 실제 데이터가 아닌 메모리 주소만 저장되어 있으므로, 단순히 레지스터 값을 출력하는 것만으로는 의미 있는 정보를 얻을 수 없다.
이를 위해 `ptrace`는 메모리 내용을 읽기 위한 `PTRACE_PEEKDATA` 인터페이스를 제공한다.

## 참고 자료
- [Linux Kernel System Call Entry](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S)
- man 2 ptrace
- [GitHub strace](https://github.com/strace/strace)