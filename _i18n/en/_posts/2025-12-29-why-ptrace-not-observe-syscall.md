---
layout: post
lang: en
ref: "why-ptrace-not-observe-syscall"
title: "Why a Tracer Cannot Observe System Calls by Default in ptrace"
date: 2025-12-29 21:10:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall"]
---

When observing system calls with `ptrace`, the first problem to resolve is **when and how the tracer becomes aware of a system call**.
The tracer and tracee are in a parent–child process relationship, and signals are used as the IPC mechanism between them.
In other words, the tracer must receive and handle a signal when the tracee performs a system call.
However, `ptrace` does not generate signals for system calls by default.

This post examines the reason for this behavior and how to enable system call observation.

## Initializing the tracer–tracee relationship
This section examines the minimal initialization steps required to establish a tracer–tracee relationship.
The tracer creates the process to be observed as a child process using a system call such as `fork`.
The child process notifies the kernel that it is traceable by calling the following function.

```c
ptrace(PTRACE_TRACEME, pid, 0);
```

By calling ptrace with the `PTRACE_TRACEME` option, the child marks itself as traceable.

The example below shows the simplest form of **tracer–tracee initialization**.
The child process (*tracee*), created via `fork`, calls `ptrace` to become traceable.
The parent process (*tracer*) waits using `waitpid` **to observe signal state changes in the child**.

{% highlight c linenos mark_lines="10 14 18" %}
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

## System calls are not observed yet
The execution result is as follows.

```text
I'm TRACER: 23716
I'm TRACEE: 23717
status of 23717: 0
```

The child process exits immediately without invoking any system calls.
Therefore, the parent observes a status value of `0`, indicating normal termination.

Now, let us modify the code so that the child invokes the `write` system call.

{% highlight c linenos mark_lines="15 21 23 24 25" %}
#include <sys/ptrace.h>  /* ptrace() */
#include <sys/wait.h>    /* waitpid() */
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

  return 0;
}
{% endhighlight %}

The parent process uses `waitpid` to observe signal state changes of the child.
The child process performs the `write` system call after calling `PTRACE_TRACEME`.

## Why is the write system call not observed?
From the output, the tracer fails to observe the `write` system call and terminates immediately.

```
I'm TRACER: 24948
I'm TRACEE: 24949
Hello, World
TRACEE has terminated by exited: 0
TRACER has terminated
```

This happens **because the `write` system call does not generate a signal**.

## Why the tracer does not observe system calls
The default behavior of `ptrace` is ***signal observation, not system call observation***.
Calling `PTRACE_TRACEME` means that all signal-related state changes of the tracee are reported to the tracer.
However, this does *not* imply that execution stops at system call boundaries.
A system call merely transfers control between user mode and kernel mode and does not generate a signal by default.

> While being traced, the tracee will stop each time a signal is delivered…  
>
> — man ptrace

## How the tracer observes system calls (PTRACE_SYSCALL)
Stopping execution at system call entry or exit is not the default behavior of `ptrace`; it is the tracer’s responsibility.
In other words, **the tracer must explicitly configure the tracee to stop at system call boundaries**.
The request that performs this role is `PTRACE_SYSCALL`.

```c
ptrace(PTRACE_SYSCALL, pid, 0, 0);
```

`PTRACE_SYSCALL` resumes the tracee while arranging for it to stop at the next system call entry or exit.
From the tracer’s perspective, the tracee appears to have stopped due to receipt of `SIGTRAP`.

> Restart the stopped tracee as for PTRACE_CONT, but arrange for the tracee to be stopped at the next entry to or exit from a system call…
From the tracer's perspective, the tracee will appear to have been stopped by receipt of a SIGTRAP.
> 
> — man ptrace

The tracer needs an appropriate timing point to invoke `PTRACE_SYSCALL`.
To achieve this, the tracee uses `SIGSTOP` as a trigger.
When the tracee raises `SIGSTOP`, the tracer is notified of the stop event via `waitpid`.
At that point, the tracer can invoke `PTRACE_SYSCALL` to configure the tracee to stop with `SIGTRAP` events on subsequent system calls.
As a result, the kernel delivers `SIGTRAP` for subsequent system calls.

The example below shows how to acquire initial control using `SIGSTOP` and then stop at system call boundaries using `PTRACE_SYSCALL`.

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

The result is as follows.
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

The tracer detects the first stop event caused by `raise(SIGSTOP)`.
It then calls `ptrace(PTRACE_SYSCALL, pid, 0, 0)` to make the tracee stop with `SIGTRAP` events at system call entry/exit.
After that, when the tracee calls write, `SIGTRAP` events occur at the system call boundary, and the tracer can observe them.

### Why multiple SIGTRAPs occur for a single system call
From the logs, we can observe multiple `SIGTRAP` events.
This occurs because additional system calls are executed during process termination after write.

## Note: execve-family calls automatically generate SIGTRAP
When not using the previously described method (`SIGSTOP` + `PTRACE_SYSCALL`),
calling a system call such as `execve`, which loads a new program image, causes the kernel to automatically generate a `SIGTRAP` event.
As a result, the tracer can gain control without any additional configuration.

> A process can initiate a trace by calling fork(2) and having the resulting child do a PTRACE_TRACEME, followed (typically) by an execve(2).
>
> — man ptrace

Typically, a child process calls `execve` after `PTRACE_TRACEME` to load a new program image.

If the `PTRACE_O_TRACEEXEC` option is not in effect,
a successful `execve` call by the tracee causes the kernel to deliver a `SIGTRAP` signal.
This allows the parent process (the tracer) to gain control before the new program begins execution.

To observe what actually happens when `execve` is called, the following test was performed.

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

**Execution result:**
```
I'm TRACER: 10126
I'm TRACEE: 10127
[10127] stopped, 5
[10127] traced stop

TRACER has terminated
```

As shown in the output, the kernel delivers a `SIGTRAP(5)` signal immediately after the `execve` call.
At this point, the child process is stopped due to a system call,
the parent process can resume the child’s execution by calling `ptrace(PTRACE_SYSCALL, pid, 0, 0)`.

> To simplify the output, the signal detection loop was removed and the program terminates after the first signal.
>
> Since the tracee executes `echo`, a large number of signals are generated in practice.

## References
- man 2 ptrace
- man 2 waitpid
- man 2 write