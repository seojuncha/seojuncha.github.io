---
layout: post
lang: en
ref: "syscall-entry-and-exit"
title: "Distinguishing System Call Entry and Exit: Extracting Arguments and Return Values"
date: 2026-01-06 22:20:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall"]
---

When a tracer uses `PTRACE_SYSCALL` to trace system calls,
the tracer receives two signals per system call: one before execution and one after.
This happens at the transition points between user mode and kernel mode, which are fundamental to system call execution.
**The tracer must distinguish these two signals as system call entry and system call exit, and read the appropriate register values at each point**.
For example, if the tracer reads the return-value register at system call entry,
it will observe a kernel-initialized placeholder rather than the actual result, leading to incorrect behavior.

This post explains how to distinguish system call entry and exit,
briefly analyzes how this mechanism is implemented in strace,
and demonstrates how to extract system call arguments and return values from registers.

## Concept of System Call Entry and Exit
A system call can be divided into two distinct phases.

**Syscall Entry**
- User mode → Kernel mode
- System call arguments are passed

**Syscall Exit**
- Kernel mode → User mode
- Return value is delivered

This is the fundamental execution model of a system call.
Under `PTRACE_SYSCALL`, two stops are observed per system call, the first corresponding to entry and the second to exit.
From the tracer’s perspective, the first observed signal for each system call can be treated as the system call entry point.

### Implementing Entry/Exit Distinction
Based on this behavior, the simplest and most efficient way to distinguish entry and exit is to use a **toggle-style flag**.

{% highlight c %}
int in_syscall = 0;

case SIGTRAP | 0x80:
  if (in_syscall) {      /* system call exit */
    in_syscall = 0;
  } else {               /* system call entry */
    in_syscall = 1;
  }
{% endhighlight %}

The `in_syscall` flag represents whether the tracee is currently inside a system call.

- `0`: system call entry
- `1`: system call exit

The initial value is set to `0`.
It is toggled to `1` at entry, and reset to `0` after handling the exit stop.

> The `SIGTRAP | 0x80` signal is generated only when `PTRACE_O_TRACESYSGOOD` is enabled.

### Reference: How strace Implements This
`strace` uses the same conceptual approach to distinguish system call entry and exit.

**defs.h**
{% highlight c %}
#define TCB_INSYSCALL      0x04
#define entering(tcp)      (!((tcp)->flags & TCB_INSYSCALL))
#define exiting(tcp)       ((tcp)->flags & TCB_INSYSCALL)
{% endhighlight %}

**syscall.c**
{% highlight c %}
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


The `TCB_INSYSCALL` flag indicates whether the thread is currently inside a system call.
It is maintained per `tcb` (*Trace Control Block*).
This is necessary because a tracer may follow multiple tracees and threads,
and must track the system call state of each independently.

## Extracting System Call Arguments and Return Values
System call arguments and return values are obtained by reading registers via `PTRACE_GETREGS`.
By understanding how system calls work, it becomes clear which registers to read at which point.

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

> The full code template is available in the [previous post]({% post_url 2025-12-31-distinguish-syscall-signal %}).

### Example Output Analysis

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

The `write` system call has the following signature.

{% highlight c %}
ssize_t write(int fd, const void *buf, size_t count);
{% endhighlight %}

As shown in the output, the `rax` value at entry is not the actual return value.

| System Call |  Arguments    | Register |
| ------- | ----------------- | ----- |
| Entry  | `int fd`          | `rdi` |
|        | `const void *buf` | `rsi` |
|         | `size_t count`    | `rdx` |
| Exit    | Return value      | `rax` |


## Why Is the Return Value -ENOSYS at System Call Entry?
{% highlight asm mark_lines="2" %}
SYM_CODE_START(entry_SYSCALL_64)
  PUSH_AND_CLEAR_REGS rax=$-ENOSYS
SYM_CODE_END(entry_SYSCALL_64)
{% endhighlight %}

On x86-64, the kernel initializes `rax` to `-ENOSYS` on system call entry.
This serves as a default failure value in case the system call handler is missing or dispatch fails.
Therefore, the `rax` value at entry is not meaningful, and tracers must only process the value at system call exit.

## Limitation of Pointer Arguments
The second argument of `write`, `buf`, is a pointer.
Since registers hold only the address, not the data itself, reading register values alone is insufficient.
For this purpose, `ptrace` provides the `PTRACE_PEEKDATA` interface **to read tracee memory**.

## References
- [Linux Kernel System Call Entry](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S)
- man 2 ptrace
- [GitHub strace](https://github.com/strace/strace)