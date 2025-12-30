---
layout: post
lang: ko
ref: "distinguish-syscall-signal"
title: "시스템콜 시그널은 일반 시그널과 어떻게 구분할까?"
date: 2025-12-29 21:10:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall", "sigtrap"]
published: false
---

`ptrace`를 활용해 시스템콜을 검출할 때, 시스템콜의 진입과 복귀는 `SIGTRAP` 시그널 형태로 tracer에 전달된다.
tracer는 tracee로부터 `SIGTRAP` 시그널을 받으면 시스템콜이 발생했음을 추측할 수 있다.
하지만 `SIGTRAP` 시그널은 시스템콜 호출 시에만 발생하는 것은 아니다.
그렇기 때문에 tracer가 `SIGTRAP` 시그널만을 기준으로 이벤트를 처리할 경우, 어떤 원인으로 발생한 시그널인지 구분하기 어려운 상황이 발생한다.

이번 글에서는 `SIGTRAP` 시그널이 발생하는 다양한 경우를 살펴보고, 시스템콜 이벤트를 다른 이벤트와 구분하는 방법을 알아본다.
