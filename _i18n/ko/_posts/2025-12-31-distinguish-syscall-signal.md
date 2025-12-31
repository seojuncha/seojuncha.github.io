---
layout: post
lang: ko
ref: "distinguish-syscall-signal"
title: "시스템콜 시그널은 일반 시그널과 어떻게 구분할까?"
date: 2025-12-31 16:10:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall", "sigtrap"]
---

`ptrace`를 활용해 시스템콜을 검출할 때, 시스템콜의 진입과 복귀는 `SIGTRAP` 시그널 형태로 tracer에 전달된다.
tracer는 tracee로부터 `SIGTRAP` 시그널을 받으면 시스템콜이 발생했음을 추측할 수 있다.
하지만 `SIGTRAP` 시그널은 시스템콜 호출 시에만 발생하는 것은 아니다.
그렇기 때문에 tracer가 `SIGTRAP` 시그널만을 기준으로 이벤트를 처리할 경우, 어떤 원인으로 발생한 시그널인지 구분하기 어려운 상황이 발생한다.

이번 글에서는 `SIGTRAP` 시그널이 발생하는 다양한 경우를 살펴보고, 시스템콜 이벤트를 다른 이벤트와 구분하는 방법을 알아본다.


## tracer의 ptrace 설정 제어 (PTRACE_SETOPTIONS)
tracer는 `ptrace`의 동작을 제어하기 위해 여러 가지 옵션을 설정할 수 있다.
`PTRACE_SETOPTIONS`는 tracer가 `ptrace` 동작 옵션을 설정하기 위해 사용하는 요청이다.

{% highlight c %}
ptrace(PTRACE_SETOPTIONS, pid, NULL, PTRACE_O_*)
{% endhighlight %}

마지막 인자로 설정할 옵션들을 OR 연산으로 비트 마스킹하여 전달할 수 있다.
간단히 `SIGTRAP` 시그널 발생을 재현하기 위해, tracee의 종료 이벤트를 수신하는 `PTRACE_O_TRACEEXIT` 옵션을 추가한다.

{% highlight c %}
ptrace(PTRACE_SETOPTIONS, pid, NULL, PTRACE_O_TRACEEXIT);
{% endhighlight %}

## tracee의 종료 이벤트 받기 (PTRACE_O_TRACEEXIT)
tracer가 `ptrace` 옵션을 설정하려면 적절한 타이밍이 필요하다.
일반적으로 tracee가 최초 시스템콜에 진입하기 전에 발생하는 `SIGSTOP` 시그널을 옵션 설정의 트리거로 사용한다.

{% highlight c linenos mark_lines="16 33" %}
#include <sys/ptrace.h>
#include <sys/wait.h>
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
    raise(SIGSTOP);
    write(1, "Hello, World\n", 13);
  } else {
    printf("I'm TRACER: %d\n", tracer_pid);

    for (;;) {
      int ws;
      waitpid(pid, &ws, 0);

      if (WIFEXITED(ws) || WIFSIGNALED(ws))
        break;

      if (WIFSTOPPED(ws)) {
        printf("[%d] stopped, %d\n", pid, WSTOPSIG(ws));
        switch (WSTOPSIG(ws)) {
          case SIGSTOP:
            printf("[%d] sigstop\n", pid);
            ptrace(PTRACE_SETOPTIONS, pid, NULL, PTRACE_O_TRACEEXIT);
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
  return 0;
}
{% endhighlight %}

위 예제에서 tracee는 `write` 시스템콜 호출 전에 `SIGSTOP` 시그널을 발생시킨다.
tracer는 `SIGSTOP`을 수신한 뒤 `PTRACE_O_TRACEEXIT` 옵션을 설정하고, `PTRACE_SYSCALL`을 호출하여 시스템콜 지점에서 `SIGTRAP`이 발생하도록 한다.

## SIGTRAP 하나로는 이벤트 원인을 구분할 수 없다
실행 결과는 다음과 같다.

{% highlight text mark_lines="7 11 14 17" %}
I'm TRACER: 14580
I'm TRACEE: 14581
[14581] stopped, 19
[14581] sigstop

[14581] stopped, 5
[14581] traced stop

Hello, World
[14581] stopped, 5
[14581] traced stop

[14581] stopped, 5
[14581] traced stop

[14581] stopped, 5
[14581] traced stop

TRACER has terminated
{% endhighlight %}

출력 결과를 보면 `Hello, World` 출력 전후를 포함해 대부분의 stop 이벤트가 모두 `SIGTRAP`으로 관측된다.
이 상태에서는 해당 `SIGTRAP`이 시스템콜로 인해 발생한 것인지, 아니면 다른 `ptrace` 이벤트로 인한 것인지 구분할 수 없다.

### EXECVE 역시 SIGTRAP stop으로 관측된다
또 다른 예로, `fork` 이후 호출되는 `execve` 역시 tracer 입장에서는 `SIGTRAP` stop 형태로 관측된다.
이 경우에도 동일한 문제가 발생한다.
`SIGTRAP`이 발생했지만, 이것이 시스템콜 때문인지 아니면 exec 이벤트로 인한 것인지 구분할 수 없다.

## PTRACE_O_TRACESYSGOOD을 사용한 시스템콜 시그널 구분
이 문제를 해결하기 위해 `ptrace`는 `PTRACE_O_TRACESYSGOOD` 옵션을 제공한다.
이 옵션이 설정되면, 시스템콜 진입 및 복귀로 인한 stop은 `SIGTRAP | 0x80` 형태로 전달된다.
이는 일반적인 `SIGTRAP`과 시스템콜 이벤트를 구분하기 위한 커널 레벨의 표시 비트이다.

{% highlight c linenos mark_lines="5 7" %}
switch (WSTOPSIG(ws)) {
  case SIGSTOP:
    printf("[%d] sigstop\n", pid);
    ptrace(PTRACE_SETOPTIONS, pid, NULL,
           PTRACE_O_TRACESYSGOOD | PTRACE_O_TRACEEXIT);
    break;
  case SIGTRAP | 0x80:
    printf("[%d] syscall stop\n", pid);
    break;
  case SIGTRAP:
    printf("[%d] traced stop\n", pid);
    break;
}
{% endhighlight %}


**실행 결과:**
{% highlight text mark_lines="7 11 14" %}
I'm TRACER: 14608
I'm TRACEE: 14609
[14609] stopped, 19
[14609] sigstop

[14609] stopped, 133
[14609] syscall stop

Hello, World
[14609] stopped, 133
[14609] syscall stop

[14609] stopped, 133
[14609] syscall stop

[14609] stopped, 5
[14609] traced stop

TRACER has terminated
{% endhighlight %}

이전 결과와 비교하면, 첫 번째 `SIGSTOP` 이후 3번의 시스템콜 stop(`SIGTRAP | 0x80`)과 1번의 일반 `SIGTRAP` stop이 발생한 것을 확인할 수 있다.

## 시스템콜을 더 정확히 확인하려면?
지금까지는 시스템콜 관련 stop이 몇 번 발생했는지만 확인했다.
하지만 시스템콜의 종류까지 확인하려면 추가 정보가 필요하다.
시스템콜은 유저 모드에서 커널 모드로 전환되며, 시스템콜 번호와 인자들은 CPU 레지스터를 통해 전달된다.
따라서 다음 단계에서는 레지스터 값을 읽어 시스템콜 번호와 이름을 확인해야 한다.

## 참고 자료
- man 2 ptrace 
- man 2 syscalls