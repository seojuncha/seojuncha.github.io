---
layout: post
lang: en
ref: "distinguish-syscall-signal"
title: "How Can We Distinguish System Call Signals from Ordinary Signals?"
date: 2025-12-31 16:10:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall", "sigtrap"]
---

When detecting system calls using `ptrace`, system call entry and exit events are delivered to the tracer in the form of a `SIGTRAP` signal.
When the tracer receives a `SIGTRAP` signal from the tracee, it may infer that a system call has occurred.
However, **`SIGTRAP` is not generated exclusively by system call events**.
Therefore, if the tracer relies solely on `SIGTRAP` to classify events, it becomes difficult to determine the actual cause of each stop.

In this post, we examine the various situations in which `SIGTRAP` can be generated and explore how to distinguish system call events from others.

## Controlling ptrace Behavior with PTRACE_SETOPTIONS
The tracer can configure various options to control the behavior of `ptrace`.
`PTRACE_SETOPTIONS` is used by the tracer to configure these `ptrace` options.

{% highlight c %}
ptrace(PTRACE_SETOPTIONS, pid, NULL, PTRACE_O_*)
{% endhighlight %}

The final argument is a bitmask created by OR-ing the desired options together.
To reproduce `SIGTRAP` generation in a simple way, we enable `PTRACE_O_TRACEEXIT` to receive tracee exit events.

{% highlight c %}
ptrace(PTRACE_SETOPTIONS, pid, NULL, PTRACE_O_TRACEEXIT);
{% endhighlight %}

## Receiving Tracee Exit Events (PTRACE_O_TRACEEXIT)
The tracer must configure `ptrace` options at the correct timing.
Typically, the `SIGSTOP` signal raised before the first system call is used as the trigger.

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

In this example, the tracee raises `SIGSTOP` before invoking the `write` system call.
The tracer sets `PTRACE_O_TRACEEXIT` upon receiving `SIGSTOP`, then uses `PTRACE_SYSCALL` to trigger `SIGTRAP` stops at system call boundaries.

## SIGTRAP Alone Is Not Sufficient to Identify Event Causes
The execution result is shown below.

{% highlight text linenos mark_lines="7 11 14 17" %}
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

As shown above, most stop events—including those around `Hello, World`—appear as `SIGTRAP`.
In this state, it is impossible to determine whether a `SIGTRAP` was caused by a system call or by another `ptrace`-related event.

### EXECVE Is Also Observed as a SIGTRAP Stop
As another example, an `execve` call after `fork` is also observed by the tracer as a `SIGTRAP` stop.
The same ambiguity arises in this case as well.
Although `SIGTRAP` is delivered, the tracer cannot tell whether it originated from a system call or an exec event.

## Distinguishing System Call Stops with PTRACE_O_TRACESYSGOOD
To solve this problem, `ptrace` provides the `PTRACE_O_TRACESYSGOOD` option.
When this option is enabled, stops caused by system call entry or exit are reported as `SIGTRAP | 0x80`.
This extra bit is a kernel-level marker used to distinguish system call stops from ordinary `SIGTRAP` events.

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

**Execution Result:**
{% highlight text linenos mark_lines="7 11 14" %}
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

Compared to the previous run, after the initial `SIGSTOP`,
we observe three system call stops (`SIGTRAP | 0x80`) and one ordinary `SIGTRAP`.

## Identifying System Calls More Precisely
So far, we have only confirmed how many system call–related stops occurred.
However, identifying the exact system call requires more information.
A system call transitions execution from user mode to kernel mode, with its number and arguments passed via CPU registers.
Therefore, the next step is to inspect register values to determine the system call number and name.

## References
- man 2 ptrace
- man 2 syscalls
