---
layout: post
lang: ko
ref: "why-ptrace-not-observe-syscall"
title: "ptrace에서 TRACER가 시스템 콜을 관측하지 못하는 이유"
date: 2025-12-29 21:10:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall"]
published: false
---

`ptrace`를 사용한 시스템 콜 관측에서 가장 먼저 해결해야 할 문제는 tracer가 **“언제, 그리고 어떻게 시스템 콜을 인지하는가”**이다.
tracer와 tracee는 부모–자식 프로세스 관계이며, 이들 사이의 IPC 수단으로 시그널을 사용한다.
즉, tracer는 tracee의 시스템 콜 호출 시점에 시그널을 받아 이를 처리해야 한다.
하지만 `ptrace`의 기본 동작은 시스템 콜 자체에 대해 시그널을 발생시키지 않는다는 점을 확인할 수 있다.

이 글에서는 그 이유와, 시스템 콜 관측을 가능하게 만드는 방법을 살펴본다.

## tracer–tracee 초기화
이 절에서는 tracer–tracee 관계를 설정하기 위한 최소 초기화 단계를 살펴본다.

tracer는 `fork`와 같은 시스템 콜을 사용해 관측 대상 프로세스(tracee)를 자식 프로세스로 생성한다.
생성된 자식 프로세스는 자신이 관측 대상임을 커널에 알리기 위해 다음 함수를 호출한다.

```c
ptrace(PTRACE_TRACEME, pid, 0);
```

자식 프로세스는 `PTRACE_TRACEME` 옵션으로 `ptrace`를 호출함으로써 자기 자신을 관측 대상으로 설정한다.

아래 예제 코드는 가장 단순한 형태의 tracer–tracee 초기화 단계이다.
`fork`로 생성된 자식 프로세스(tracee)는 `ptrace`를 호출하여 관측 대상으로 설정된다.
부모 프로세스(tracer)는 자식의 시그널 상태 변화를 감지하기 위해 `waitpid`로 대기한다.

{% highlight c mark_lines="10 14 18" %}
#include <sys/ptrace.h> /* ptrace()  */
#include <sys/wait.h>   /* waitpid() */
#include <unistd.h>
#include <stdio.h>

int main(void) {
  pid_t tracer_pid = getpid();
  pid_t pid;

  pid = fork();
  if (pid == 0) {
    pid_t me = getpid();
    printf("I'm TRACEE: %d\n", me);
    ptrace(PTRACE_TRACEME, me, 0);
  } else {
    int ws;
    printf("I'm TRACER: %d\n", tracer_pid);
    waitpid(pid, &ws, 0);
    printf("status of %d: %d\n", pid, ws);
  }

  return 0;
}
{% endhighlight %}

## 아직 시스템 콜은 관측되지 않는다
실행 결과는 다음과 같다.

```
I'm TRACER: 23716
I'm TRACEE: 23717
status of 23717: 0
```

자식 프로세스는 어떤 시스템 콜도 호출하지 않고 곧바로 종료한다.
따라서 부모 프로세스가 관측한 상태 값은 정상 종료를 의미하는 `0`이다.
이제 자식 프로세스가 `write` 시스템 콜을 호출하도록 코드를 수정해 보자.

{% highlight c linenos mark_lines="15 21 23 24 25" %}
#include <sys/ptrace.h>  /* ptrace() */
#include <sys/wait.h>   /* waitpid() */
#include <unistd.h>
#include <stdio.h>

int main(void) {
  pid_t tracer_pid = getpid();
  pid_t pid;

  pid = fork();
  if (pid == 0) {
    pid_t me = getpid();
    printf("I'm TRACEE: %d\n", me);
    ptrace(PTRACE_TRACEME, 0, 0, 0);
    write(1, "Hello, World\n", 13);
  } else {
    printf("I'm TRACER: %d\n", tracer_pid);

    for (;;) {
      int ws;
      waitpid(pid, &ws, 0);

      if (WIFEXITED(ws)) {
        printf("TRACEE has terminated by exited: %d\n",
               WEXITSTATUS(ws));
        break;
      }

      if (WIFSIGNALED(ws)) {
        printf("TRACEE has terminated by signaled: %d\n",
               WTERMSIG(ws));
        break;
      }

      if (WIFSTOPPED(ws)) {
        printf("[%d] stopped, %d\n", pid, WSTOPSIG(ws));
      }
    }

    printf("TRACER has terminated\n");
  }
}
{% endhighlight %}

부모 프로세스는 `waitpid`를 통해 자식 프로세스의 시그널 상태 변화를 감지하고,
자식 프로세스는 `PTRACE_TRACEME` 호출 이후 `write` 시스템 콜을 수행한다.

## write 시스템 콜은 왜 관측되지 않는가
출력 결과를 보면 tracer는 `write` 시스템 콜을 관측하지 못하고 바로 종료된다.

```
I'm TRACER: 24948
I'm TRACEE: 24949
Hello, World
TRACEE has terminated by exited: 0
TRACER has terminated
```

이는 **`write` 시스템 콜이 시그널을 발생시키지 않기 때문**이다.

## TRACER가 시스템 콜 관측을 시작하지 않는 이유
**`ptrace`의 기본 동작은 시스템 콜 관측이 아니라 시그널 관측**이다.
`PTRACE_TRACEME`를 호출한다는 것은 tracee의 모든 시그널 상태 변화가 tracer에게 전달된다는 의미이다.
하지만 이것이 시스템 콜 단위에서 실행이 중단됨을 의미하지는 않는다.
시스템 콜은 유저 모드와 커널 모드 사이에서 제어권을 전환할 뿐, 별도의 시그널을 발생시키지 않는다.

> While being traced, the tracee will stop each time a signal is delivered…
>
> — man ptrace

## TRACER가 시스템 콜을 관측하는 방법 (PTRACE_SYSCALL)
시스템 콜 진입 또는 종료 시점에서 멈추도록 만드는 것은 `ptrace`의 기본 동작이 아니라 tracer의 역할이다.
즉, tracer는 tracee가 시스템 콜 호출 시점에 멈추도록 명시적으로 설정해야 한다.
이 역할을 수행하는 요청이 `PTRACE_SYSCALL`이다.

```c
ptrace(PTRACE_SYSCALL, pid, 0, 0);
```

`PTRACE_SYSCALL`은 tracee를 재개시키되, 다음 시스템 콜 진입(entry) 또는 종료(exit) 시점에 멈추도록 설정한다.
이때 tracer 관점에서는 tracee가 `SIGTRAP`을 받은 것처럼 보인다.

> Restart the stopped tracee as for PTRACE_CONT, but arrange for the tracee to be stopped at the next entry to or exit from a system call…
From the tracer's perspective, the tracee will appear to have been stopped by receipt of a SIGTRAP.
>
> — man ptrace

tracer는 `PTRACE_SYSCALL`을 호출하기 위한 적절한 타이밍이 필요하다.
이를 위해 tracee는 `SIGSTOP` 시그널을 트리거로 사용한다.
tracee가 `SIGSTOP`을 발생시키면 tracer는 `waitpid`를 통해 해당 정지 이벤트를 통지받는다.
그리고 tracer는 이 시점에서 `PTRACE_SYSCALL`을 호출하여, 이후 시스템 콜에서 `SIGTRAP` 이벤트로 멈추도록 설정할 수 있다.
그 결과 커널은 이후 발생하는 시스템 콜에 대해 `SIGTRAP`을 전달한다.

아래 예제는 `SIGSTOP`으로 초기 제어권을 확보한 뒤 `PTRACE_SYSCALL`로 시스템 콜 경계에서 멈추게 만드는 과정을 보여준다.

{% highlight c linenos mark_lines="6 18 21 24 32" %}
  if (pid == 0) {
    pid_t me = getpid();
    printf("I'm TRACEE: %d\n", me);

    ptrace(PTRACE_TRACEME, 0, 0, 0);
    raise(SIGSTOP);
    write(1, "Hello, World\n", 13);
  } else {
    printf("I'm TRACER: %d\n", tracer_pid);

    for(;;) {
      int ws;
      waitpid(pid, &ws, 0);

      if (WIFEXITED(ws) || WIFSIGNALED(ws))
        break;

      if (WIFSTOPPED(ws)) {
        printf("[%d] stopped, %d\n", pid, WSTOPSIG(ws));
        switch (WSTOPSIG(ws)) {
          case SIGSTOP:
            printf("[%d] sigstop\n", pid);
            break;
          case SIGTRAP:
            printf("[%d] traced stop\n", pid);
            break;
          default:
            printf("[%d] unknown stop\n", pid);
            break;
        }
        printf("\n");
        ptrace(PTRACE_SYSCALL, pid, 0, 0);
      }
    }

    printf("TRACER has terminated\n");
  }
{% endhighlight %}

결과는 다음과 같다.
```
I'm TRACER: 27025
I'm TRACEE: 27026
[27026] stopped, 19
[27026] sigstop

[27026] stopped, 5
[27026] traced stop

Hello, World
[27026] stopped, 5
[27026] traced stop

[27026] stopped, 5
[27026] traced stop

TRACER has terminated
```

tracer는 `raise(SIGSTOP)`에 의해 발생한 첫 번째 정지 이벤트를 검출한다.
그 다음 `ptrace(PTRACE_SYSCALL, pid, 0, 0)`을 호출하여 시스템 콜 진입/종료 시점에서도 `SIGTRAP` 이벤트로 멈추도록 설정한다.
이후 tracee가 `write`를 호출하면 시스템 콜 경계에서 `SIGTRAP` 이벤트가 발생하며, tracer는 이를 관측할 수 있다.

### 시스템 콜 하나에서 SIGTRAP이 여러 번 발생하는 이유
로그를 보면 `SIGTRAP` 이벤트가 여러 번 발생한다.
이는 `write` 호출 진입/종료 시점과 프로세스 종료 과정에서 추가 시스템 콜이 발생하기 때문이다.

## 참고: EXECVE 계열 호출에서는 SIGTRAP을 자동으로 보내준다
위에서 설명한 방식(`SIGSTOP` + `PTRACE_SYSCALL`)을 사용하지 않더라도,
`execve`와 같이 프로그램 이미지를 로딩하는 시스템 콜을 호출하면 커널이 자동으로 `SIGTRAP` 이벤트를 발생시킨다.
이로 인해 tracer는 별도의 설정 없이도 제어권을 획득할 수 있다.

> A process can initiate a trace by calling fork(2) and having the resulting child do a PTRACE_TRACEME, followed (typically) by an execve(2).
>
> — man ptrace

일반적으로 자식 프로세스는 `PTRACE_TRACEME` 호출 이후 `execve`를 호출하여 새로운 프로그램 이미지를 로딩한다.

`PTRACE_O_TRACEEXEC` 옵션이 설정되지 않은 경우, tracee가 성공적으로 `execve`를 호출하면 커널은 `SIGTRAP` 시그널을 전달한다.
이를 통해 새로운 프로그램이 실행되기 전에 부모 프로세스(tracer)가 제어권을 획득할 수 있다.

`execve` 호출 시 실제로 어떤 동작이 발생하는지 확인하기 위해 다음과 같은 테스트를 수행했다.

{% highlight c linenos mark_lines="6 22 23" %}
if (pid == 0) {
  pid_t me = getpid();
  char *argv[] = {"/usr/bin/echo", "Hello, World\n", NULL};
  printf("I'm TRACEE: %d\n", me);
  ptrace(PTRACE_TRACEME, 0, 0, 0);
  execve(argv[0], &argv[0], NULL);
  perror("execve");
} else {
  int ws;

  printf("I'm TRACER: %d\n", tracer_pid);
  waitpid(pid, &ws, 0);

  if (WIFEXITED(ws) || WIFSIGNALED(ws)) {
    printf("TRACEE has terminated\n");
  } else if (WIFSTOPPED(ws)) {
    printf("[%d] stopped, %d\n", pid, WSTOPSIG(ws));
    switch (WSTOPSIG(ws)) {
      case SIGSTOP:
        printf("[%d] sigstop\n", pid);
        break;
      case SIGTRAP:
        printf("[%d] traced stop\n", pid);
        break;
      default:
        printf("[%d] unknown stop\n", pid);
        break;
    }
    printf("\n");
    ptrace(PTRACE_SYSCALL, pid, 0, 0);
  }

  printf("TRACER has terminated\n");
}
{% endhighlight %}

**실행 결과:**
```
I'm TRACER: 10126
I'm TRACEE: 10127
[10127] stopped, 5
[10127] traced stop

TRACER has terminated
```

출력 결과에서 확인할 수 있듯이, 커널은 `execve` 호출 직후 `SIGTRAP` 시그널을 전달한다.
이 시점에서 자식 프로세스는 시스템 콜에 의해 멈춘 상태이며,
부모 프로세스는 `ptrace(PTRACE_SYSCALL, pid, 0, 0)`을 호출하여 자식 프로세스의 실행을 재개할 수 있다.

> 출력 결과의 단순화를 위해 시그널 검출 루프는 삭제하고, 첫 번째 시그널 이후에 프로그램이 종료되도록 구성하였다.
> 
> tracee는 `echo` 프로그램을 실행하므로, 실제로는 매우 많은 시그널이 발생한다.

## 참고 자료
- man 2 ptrace
- man 2 waitpid
- man 2 write
