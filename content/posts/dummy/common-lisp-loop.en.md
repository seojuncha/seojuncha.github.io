+++
date = '2025-02-24'
title = "Mastering Loop in Common Lisp: A Beginner's Guide"
tags = ["common lisp", "common lisp loop", "lisp iteration", "lisp programming"]
description = "Learn how to use the powerful `loop` construct in Common Lisp to simplify iteration and control flow. This guide covers fundamental usage with examples."
draft = true
+++

## Introduction

Common Lisp provides multiple ways to implement loops, but the `loop` construct stands out as a powerful and intuitive iteration mechanism. In this guide, we will explore the fundamental uses of `loop`, covering essential syntax and examples to help you master its usage effectively.

## Basic `loop` Usage
The simplest form of the `loop` statement creates an infinite loop.

```lisp
(loop
  (format t "This is a loop!~%"))
```

### Specifying Iteration Count (`repeat`)
To execute a loop a fixed number of times, use `repeat`.

```lisp
(loop repeat 5 do
  (format t "This is a loop!~%"))
```

### Iterating Over a List (`for...in`)
To iterate over elements in a list:

```lisp
(loop for item in '(1 2 3) do
  (format t "~a~%" item))
```

### Iterating Over a Range (`for...from/to/by`)
Use `from` and `to` to define numeric ranges:

```lisp
(loop for i from 0 to 5 do
  (format t "~d~%" i))
```

Use `downto` for descending values:

```lisp
(loop for i from 5 downto 0 do
  (format t "~d~%" i))
```

Use `by` to set custom increments:

```lisp
(loop for i from 0 to 5 by 2 do
  (format t "~d~%" i))
```

## Controlling Loop Execution
### Using `while` for Conditional Execution

```lisp
(defparameter *number* 5)
(loop while (< *number* 10) do
  (format t "~d~%" *number*)
  (incf *number*))
```

### Using `always` for Condition Verification

```lisp
(loop for i in '(2 4 6 8 10)
  always (evenp i))
```

### Using `thereis` for Early Termination

```lisp
(let ((check-ret (loop for i in '(1 3 5 7 8 9)
                    thereis (if (evenp i) i))))
  (format t "Return: ~d~%" check-ret))
```

## Using Conditions Inside `loop`
### Using `when`

```lisp
(loop for i from 0 upto 6
  when (evenp i) do
    (format t "~d is even~%" i))
```

### Using `if`

```lisp
(loop for i from 1 to 10 do
  (if (oddp i)
    (format t "~A is odd.~%" i)
    (format t "~A is even.~%" i)))
```

## Conclusion
The `loop` construct in Common Lisp provides a concise and expressive way to handle iteration. Mastering `loop` will help you write cleaner and more readable Lisp code.

- Use `repeat` for fixed iteration counts.
- Use `for...in` for list iteration.
- Use `for...from/to/by` for numerical ranges.
- Use `when` and `if` to introduce conditional logic.

For further reading, refer to the [HyperSpec documentation](https://www.lispworks.com/documentation/HyperSpec/Body/m_loop.htm#loop) and experiment with different loop constructs!

