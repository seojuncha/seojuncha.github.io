---
layout: post
lang: en
ref: "ptrace-read-memory"
title: "Extracting Memory Data with PTRACE_PEEKDATA"
date: 2026-01-24 19:00:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall", "PTRACE_PEEKDATA"]
---

Using `PTRACE_GETREGS`, we can inspect the system call number, arguments, and return value stored in registers at the time of a system call.
However, some system calls take pointers to memory addresses rather than simple integer values as arguments.
In such cases, the value stored in the register is not the actual data, but merely an address in the tracee’s virtual memory space.
Therefore, we **must explicitly read the memory contents pointed to by that address**.
A representative example is the `write` system call.

{% highlight c %}
ssize_t write(int fd, const void *buf, size_t count);
{% endhighlight %}

The second argument `buf` is a memory address pointing to the data to be written,
and the register stores only that address, not the string itself.

In this article, we will use `PTRACE_PEEKDATA` to demonstrate **how to extract data from the tracee process’s virtual memory**.

## Overview of PTRACE_PEEKDATA
{% highlight c %}
long ptrace(PTRACE_PEEKDATA, pid_t pid, void *addr, void *data);
{% endhighlight %}

`PTRACE_PEEKDATA` reads from the tracee process’s virtual memory,
returning `sizeof(long)` bytes (i.e., one machine word) starting at the given address(`addr`).

> PTRACE_PEEKTEXT, PTRACE_PEEKDATA  
> Read a word at the address `addr` in the tracee's memory, returning the word as the result of the `ptrace()` call.  
>  — man 2 ptrace

The important points to note are:
- The return unit is a word, not a single byte (`sizeof(long)`)
- On x86_64 systems, `sizeof(long) == 8`
- The returned value must be interpreted in little-endian format

## Inspecting write(buf) with PTRACE_PEEKDATA
Below is a simple example that inspects the `buf` argument of a `write` system call printing `"Hello, World\n"`,
at the system call entry point.

> In this example, `PTRACE_O_TRACESYSGOOD` is enabled,
> so system call stops are delivered as `SIGTRAP | 0x80`.

{% highlight c mark_lines="3" %}
static void peek_word(pid_t pid, unsigned long addr) {
    errno = 0;
    long ret = ptrace(PTRACE_PEEKDATA, pid, addr, NULL);
    if (ret == -1 && errno) {
        perror("PTRACE_PEEKDATA");
        return;
    }
    printf("peekdata: 0x%lx\n", ret);
}
{% endhighlight %}

{% highlight c mark_lines="3" %}
case SIGTRAP | 0x80:
    if (!in_syscall) { /* syscall entry */
        peek_word(pid, regs.rsi); /* write(fd, buf, count) */
    }
    break;
{% endhighlight %}

**Output:**
{% highlight text %}
peekdata: 0x57202c6f6c6c6548
{% endhighlight %}

Since the word size on x86_64 is 8 bytes,
8 bytes are returned at once from the memory pointed to by `buf`.

### Little-Endian Interpretation
Breaking down the returned value `0x57202c6f6c6c6548` byte by byte yields:

| Memory Address  | Value   |
| -------- | -------------- |
| addr + 0 | 0x48 (`H`)     |
| addr + 1 | 0x65 (`e`)     |
| addr + 2 | 0x6c (`l`)     |
| addr + 3 | 0x6c (`l`)     |
| addr + 4 | 0x6f (`o`)     |
| addr + 5 | 0x2c (`,`)     |
| addr + 6 | 0x20 (`SPACE`) |
| addr + 7 | 0x57 (`W`)     |

In other words, the returned word corresponds to the first 8 bytes of `"Hello, W"`.

## The Simplest Way to Read the Entire String (Not Recommended)
The following approach is for demonstration purposes only and is not recommended due to performance and alignment concerns.

{% highlight c %}
static void peek_bytewise(pid_t pid, unsigned long addr, size_t sz) {
    for (size_t i = 0; i < sz; i++) {
        errno = 0;
        long ret = ptrace(PTRACE_PEEKDATA, pid, addr + i, NULL);
        if (ret == -1 && errno) {
            perror("PTRACE_PEEKDATA");
            return;
        }
        putchar(ret & 0xff);
    }
}
{% endhighlight %}

- Calls `ptrace()` once per byte
- Inefficient for a syscall tracer
- Implicitly assumes safe unaligned access

## Recommended Approach: Read by Words and Copy to a Buffer
By using the byte count and the word size, we can minimize the number of `ptrace` calls.

{% highlight c %}
#define WORD_BYTES (sizeof(long))

static void peek_buffer(pid_t pid,
                        unsigned long addr,
                        size_t size)
{
    size_t off;
    char *buf = calloc(size, 1);

    for (off = 0; off < size; off += WORD_BYTES) {
        errno = 0;
        long word = ptrace(PTRACE_PEEKDATA, pid, addr + off, NULL);
        if (word == -1 && errno) {
            perror("PTRACE_PEEKDATA");
            break;
        }

        size_t n = size - off;
        if (n > WORD_BYTES)
            n = WORD_BYTES;

        memcpy(buf + off, &word, n);
    }

    fwrite(buf, 1, size, stdout);
    free(buf);
}
{% endhighlight %}

This approach has the following characteristics:
- Number of ptrace calls is `ceil(size / WORD_BYTES)`
- Does not rely on memory alignment
- Safe for arbitrary binary buffers, not just strings

## Important Note: write Is Not a String System Call
The `write` system call is not a string-output API.

> `write()` writes up to count bytes from the buffer starting at `buf`  
>    — man 2 write

This means:
- `buf` is not guaranteed to be NULL-terminated
- It may contain binary data
- Using `printf("%s", buf)` is logically incorrect

Only a limited set of system calls can safely assume string arguments.

## General Classification of Pointer Arguments
In ptrace-based syscall tracers, pointer arguments typically fall into three categories:

1. Buffer + Length
- `read`, `write`
- Meaning depends on syscall entry vs exit

2. NULL-terminated Strings
- `open`, `unlink`, `execve`

3. Structs / Arrays / Pointer Arrays
- `stat`, `uname`, `execve(argv)`

`write` belongs to the simplest category.

