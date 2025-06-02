+++
date = '2025-02-13'
title = "Mastering Conditionals in Common Lisp: when, unless, if, cond"
tags = ["common lisp", "common lisp if", "common lisp if-else", "common lisp cond", "common lisp when", "common lisp unless"]
description = "Learn how to use conditionals in Lisp. This guide covers when, unless, if, and cond statements to control logic flow and return values effectively. Improve code readability and minimize errors by mastering these conditional structures."
draft = true
+++

Today, we'll explore how to use **branching statements** to control program logic flow. A **branch statement** determines the next action based on whether a given condition is **true or false**. Common Lisp (hereafter, Lisp) provides various ways to perform condition checks, and understanding these approaches will help you **write more readable code** while **minimizing errors**.

## True (`t`) or False (`nil`): Boolean Logic in Lisp
Like most programming languages, Lisp evaluates conditions to determine **truth or falsity**. 
For example, in Python, we use `True` and `False` to represent boolean values:

```python
is_true = True
is_false = False
```

Lisp also distinguishes between true and false, but it does not have an explicit `bool` type. Instead:
- **`t`** represents true.
- **`nil`** represents false (and also the empty list `()` in Lisp).

```lisp
(defparameter *is-true* t)
(defconstant +empty+ nil)
```

## Checking a Single Condition: `when` and `unless`
To check a single condition, Lisp provides the `when` and `unless` macros:

- [`when`|`unless`] *test-form* form* => result*

- `test-form` is an expression that evaluates to true or false.
- `form*` represents one or more expressions that execute if the condition is met.

For example:

```lisp
(defconstant +pi+ 3.14)
(defparameter *my-value* +pi+)

;; (= *my-value* +pi+) evaluates to t, so `when` executes its body
(when (= *my-value* +pi+)
  (format t "my-value is PI!~%"))

;; Changing *my-value*
(setq *my-value* 3)

;; (= *my-value* +pi+) evaluates to nil, so `unless` executes its body
(unless (= *my-value* +pi+)
  (format t "my-value is not PI!~%"))
```

> Like other keyword symbols in Lisp, `when` and `unless` are designed to read naturally as English sentences, making them intuitive.

In Python, similar logic can be expressed using `if` and `if not`:

```python
pi = 3.14
my_value = pi

if pi == my_value:
  print("my_value is PI!")

my_value = 3
if not pi == my_value:
  print("my_value is not PI!")
```

### Numeric Comparison Operators: `=`, `/=`, `>`, `<`, `>=`, `<=`
Before moving on, let's briefly cover numeric comparison operators in Lisp.
Lisp provides six standard comparison operators:
- `=`
  - Checks if two numbers are equal
- `/=`
  - Checks if two numbers are not equal
- `>`, `<`, `>=`, `<=`
  - Compare numerical values

A key point is that these operators perform **mathematical equality** and comparisons.

```lisp
(= 10 10)    ; t
(= 10 10.0)  ; t (mathematically equal)
(eql 10 10.0) ; nil (different data types: integer vs. float)
(/= 4 8/3)   ; t, because 4 is not equal to 8/3 (≈ 2.666...)
```

> We'll cover `eql`, `equalp`, and string comparison operators in a future post.

## Checking Both True and False Cases: `if`

The `if` expression in Lisp works like `if-else` in Python.

```python
num = 10
if num < 10:
  print("num is less than 10")
else:
  print("num is greater than or equal to 10")
```

Translated to Lisp:

```lisp
(defparameter *number* 10)
(if (< *number* 10)   ; The condition (< *number* 10) evaluates to nil
  (format t "number is less than 10~%")  ; Executes if condition is true
  (format t "num is greater than or equal to 10~%"))  ; Executes if condition is false
```

If you don’t need an `else` branch, you can omit it:

```lisp
(if t (format t "true"))
```

## Checking Even/Odd: `evenp` & `oddp`
Lisp provides built-in predicates to check whether a number is even or odd:

```lisp
(let ((is-even (evenp 4))   ; Returns t
      (is-odd (oddp 7)))    ; Returns t
  (format t "is-even: ~a , is-odd: ~a~%" is-even is-odd))
```

> The `p` at the end of `evenp` and `oddp` follows Lisp’s convention for predicate functions.

## Checking Multiple Conditions: `cond`
`cond` is similar to Python's `if-elif-else` structure.

```python
number = 8

if number < 0:
  print("is negative")
elif number >= 0 and number < 10:
  print("is positive and less than or equal to 10")
else:
  print("is greater than 10")
```

Using `cond` in Lisp:

```lisp
(let ((number 8))
  (cond 
    ((< number 0) 
      (format t "is negative~%"))
    ((and (>= number 0) (< number 10)) 
      (format t "is positive and less than or equal to 10~%"))
    (t  ; Always executes if no previous condition is met
      (format t "is greater than 10~%"))))
```
`cond`'s final `(t ...)` clause acts like an `else` statement, executing only if none of the previous conditions are met.

## All Conditionals Return a Value
All conditionals in Lisp return the last evaluated expression.

```lisp
(when t
  (format t "Hello, World!~%")
  42)  ; => 42
```

This behavior allows us to use conditionals to assign values:

```lisp
(defparameter *is-set* t)
(let ((check-set (when *is-set* 99)))  ; Assigns 99 if *is-set* is true
  (format t "check-set: ~a~%" check-set))
```

## Conclusion
We've explored various conditional structures in Lisp. `when` and `unless` simplify single-condition checks, `if` provides two-way branching, and `cond` offers a structured way to handle multiple conditions.

By combining these conditionals effectively, you can write cleaner, more readable Lisp code. Since Lisp uses `t` and `nil` as truth values, understanding this behavior is crucial for writing robust conditionals.

Stay tuned for our next post on `case` and [equality comparison functions](/posts/common-lisp/equality-predicate-in-common-lisp/) like `eql`, `equal`, and `equalp`!

**For more details on variable assignments in Common Lisp, check out this post:** [**Understanding Variable Assignment in Common Lisp**](/posts/variable-assignment-in-common-lisp/)
