---
layout: post
lang: ko 
ref: "ptrace-read-memory"
title: "PTRACE_PEEKDATA로 메모리 데이터 추출하기"
date: 2026-01-24 19:20:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall", "PTRACE_PEEKDATA"]
---

`PTRACE_GETREGS`를 사용하면 시스템콜 호출 시 레지스터에 저장된 시스템콜 번호, 인자, 반환값을 확인할 수 있다.
하지만 일부 시스템콜은 단순한 정수형 값이 아니라 메모리 주소를 가리키는 포인터를 인자로 사용한다.
이 경우, 레지스터에 저장된 값은 “의미 있는 데이터”가 아니라 tracee 가상 메모리 공간의 주소일 뿐이다.
따라서 해당 주소가 가리키는 실제 메모리 내용을 추가로 읽어야 한다.
대표적인 예가 `write` 시스템콜이다.

{% highlight c %}
ssize_t write(int fd, const void *buf, size_t count);
{% endhighlight %}

`write`의 두 번째 인자인 `buf`는 출력할 데이터를 저장한 메모리 주소이며, 레지스터에는 문자열 자체가 아니라 그 주소값만 저장된다.

이번 글에서는 `PTRACE_PEEKDATA`를 사용하여 tracee 프로세스의 가상 메모리로부터 데이터를 추출하는 방법을 정리한다.

## PTRACE_PEEKDATA 개요
{% highlight c %}
long ptrace(PTRACE_PEEKDATA, pid_t pid, void *addr, void *data);
{% endhighlight %}

`PTRACE_PEEKDATA`는 tracee 프로세스의 가상 메모리에서 `addr`가 가리키는 주소로부터 `sizeof(long)` 바이트(= word 크기) 를 읽어 반환한다.

> PTRACE_PEEKTEXT, PTRACE_PEEKDATA  
> Read a word at the address `addr` in the tracee's memory, returning the word as the result of the `ptrace()` call.  
>  — man 2 ptrace

여기서 중요한 점은 다음과 같다.
- 반환 단위는 바이트가 아니라 word `(sizeof(long))`
- x86_64 환경에서는 `sizeof(long) == 8`
- 반환값은 리틀 엔디안(Little Endian) 형식으로 해석

## write(buf) 인자를 PTRACE_PEEKDATA로 확인하기
다음은 `"Hello, World\n"`를 출력하는 `write` 시스템콜의 `buf` 인자를 시스템콜 진입 시점에서 읽는 예제이다.

> 본 예제에서는 `PTRACE_O_TRACESYSGOOD` 옵션을 설정했기 때문에
> 시스템콜 stop은 `SIGTRAP | 0x80` 형태로 들어온다고 가정한다.

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
    if (!in_syscall) {                /* syscall entry */
        peek_word(pid, regs.rsi);     /* write(fd, buf, count) */
    }
    break;
{% endhighlight %}

**실행 결과:**
{% highlight text %}
peekdata: 0x57202c6f6c6c6548
{% endhighlight %}

x86_64 아키텍처에서 word 크기는 8바이트이므로, `buf`가 가리키는 메모리로부터 8바이트가 한 번에 반환된다.

### 리틀 엔디안 해석
반환된 값 `0x57202c6f6c6c6548`을 바이트 단위로 풀어보면 다음과 같다.

| 메모리 주소   | 값              |
| -------- | -------------- |
| addr + 0 | 0x48 (`H`)     |
| addr + 1 | 0x65 (`e`)     |
| addr + 2 | 0x6c (`l`)     |
| addr + 3 | 0x6c (`l`)     |
| addr + 4 | 0x6f (`o`)     |
| addr + 5 | 0x2c (`,`)     |
| addr + 6 | 0x20 (`SPACE`) |
| addr + 7 | 0x57 (`W`)     |

즉, 반환된 word 값은 문자열 `"Hello, W"`의 첫 8바이트에 해당한다.

## 문자열 전체를 읽는 가장 단순한 방법 (비권장 예제)
아래 방식은 개념 설명용 예제이며, 성능 및 정렬 관점에서는 권장되지 않는다.

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

- 매 바이트마다 `ptrace()` 호출
- syscall 트레이서 입장에서는 비효율적
- 주소 정렬(alignment)에 대한 암묵적 가정 포함

## 권장 방식: word 단위로 읽어 버퍼에 복사
읽어야 할 바이트 수 와 word 크기를 기준으로 `ptrace` 호출 횟수를 최소화할 수 있다.

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

이 방식의 특징은 다음과 같다.
- `ptrace` 호출 횟수는 `ceil(size / WORD_BYTES)`
- 메모리 정렬에 의존하지 않음
- 문자열이 아닌 임의의 바이너리 데이터에도 안전

## 중요한 주의사항: write는 문자열 시스템콜이 아니다
`write` 시스템콜은 문자열 출력용 API가 아니다.

> `write()` writes up to count bytes from the buffer starting at `buf`  
>   — man 2 write

즉,
- `buf`는 NULL 종료 문자열이라는 보장이 없음
- 바이너리 데이터일 수 있음
- `printf("%s", buf)` 는 논리적으로 잘못된 가정

문자열이라고 단정할 수 있는 경우는 제한적이다.

## 포인터 인자 처리의 일반적인 분류
`ptrace` 기반 시스템콜 트레이서에서 포인터 인자는 보통 다음 세 가지로 나뉜다.

1. 버퍼 + 길이
- `read`, `write`
- syscall entry / exit 시점에 따라 의미가 달라짐

2. NULL-terminated 문자열
- `open`, `unlink`, `execve`

3. 구조체 / 배열 / 포인터 배열
- `stat`, `uname`, `execve(argv)`

`write`는 이 중 가장 단순한 형태에 해당한다.
