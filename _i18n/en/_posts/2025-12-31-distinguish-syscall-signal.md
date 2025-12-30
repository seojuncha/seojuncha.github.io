---
layout: post
lang: en
ref: "distinguish-syscall-signal"
title: "How Can We Distinguish System Call Signals from Ordinary Signals?"
date: 2025-12-31 21:10:00 +0900
categories: ["linux"]
tags: ["ptrace", "syscall", "sigtrap"]
published: false
---

When detecting system calls using `ptrace`, system call entry and exit events are delivered to the tracer in the form of a `SIGTRAP` signal.
When the tracer receives a `SIGTRAP` signal from the tracee, it may infer that a system call has occurred.
However, **`SIGTRAP` is not generated exclusively by system call events**.
Therefore, if the tracer relies solely on `SIGTRAP` to classify events, it becomes difficult to determine the actual cause of each stop.

In this post, we examine the various situations in which `SIGTRAP` can be generated and explore how to distinguish system call events from others.