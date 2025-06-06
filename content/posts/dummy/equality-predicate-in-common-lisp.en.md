+++
date = '2025-02-25'
title = "Understanding Equality in Common Lisp"
tags = ["common lisp", "common lisp equality"]
description = "Learn the key differences between eq, eql, equal, and equalp in Common Lisp. This guide helps you choose the right equality predicate for symbols, numbers, strings, and lists."
draft = true
+++

## Introduction

**Equality comparison** is a crucial concept in all programming languages. However, in **Common Lisp**, there are multiple **equality predicates**, making it essential to understand their differences and use them correctly. In this post, we will explore the **four primary equality predicates provided by Common Lisp**, their characteristics, and how to use them effectively.

## Types of Equality in Common Lisp

Common Lisp provides four primary comparison operators:

1. **`eq`** – Reference Equality (Identity Comparison)
2. **`eql`** – `eq` with additional support for numbers and characters
3. **`equal`** – Structural Equality for lists and strings
4. **`equalp`** – Case-insensitive and type-flexible equality comparison

Let's examine how each of these operators works in detail.

---

### `eq` – Reference Equality (Identity Comparison)
```lisp
(eq 'a 'a) ;; ⇒ T
(eq (list 1 2) (list 1 2)) ;; ⇒ NIL
(eq 100 100) ;; ⇒ Implementation-dependent, T in SBCL
(eq "hello" "hello") ;; ⇒ Implementation-dependent, NIL in SBCL
```
- `eq` checks whether two values **point to the same memory location**.
- It is reliable for **symbol comparisons** due to Lisp's **interning** mechanism.
- **Numbers and strings** may yield implementation-dependent results when compared using `eq`.

> **What is interning?**
>
> Interning is an optimization technique where the compiler or runtime system stores identical values in a single memory location, ensuring that references to the same symbol share memory. This allows symbols with the same name to have the same memory address.

---

### `eql` – `eq` with Number and Character Comparison Support
```lisp
(eql 100 100) ;; ⇒ T
(eql #\A #\A) ;; ⇒ T
(eql (list 1 2) (list 1 2)) ;; ⇒ NIL
```
- `eql` behaves like `eq` but correctly compares **numbers and characters**.
- **Numbers with the same value and type** are considered equal under `eql`.
- **Characters with the same value** also return `T`.

---

### `equal` – Structural Equality for Lists and Strings
```lisp
(equal '(1 2 3) '(1 2 3)) ;; ⇒ T
(equal "hello" "hello") ;; ⇒ T
(equal #(1 2 3) #(1 2 3)) ;; ⇒ NIL (different array instances)
```
- `equal` **recursively compares lists**.
- It **compares strings character by character** to determine equality.
- Unlike `equalp`, **arrays and different numeric types may not be considered equal**.

---

### `equalp` – Case-Insensitive and Type-Flexible Equality
```lisp
(equalp 1 1.0) ;; ⇒ T
(equalp "hello" "HELLO") ;; ⇒ T
(equalp #(1 2 3) #(1 2 3)) ;; ⇒ T
```
- `equalp` **ignores case differences in strings**.
- It treats `1` and `1.0` as equal, making it **type-flexible** for numeric comparisons.
- Unlike `equal`, **arrays are compared element-wise**.

---

### Choosing the Right Equality Predicate

| Operator  | Use Case |
|------------|----------|
| `eq`       | Check if two objects refer to the same memory location (best for symbols). |
| `eql`      | Similar to `eq`, but correctly handles numbers and characters. |
| `equal`    | Compare lists and strings structurally. |
| `equalp`   | Case-insensitive string comparison and type-flexible numeric comparison. |

> The higher an operator appears in this table, the **stricter** its comparison criteria.

---
### Code for Testing Each Equality Predicate
```lisp
(defun check (func)
  (let ((ret func))
    (format t "return: ~a~%" ret)))

(format t "eq~%")
(check (eq 'a 'a))
(check (eq 100 100))
(check (eq 100 100.0))
(check (eq #\a #\a))
(check (eq #\a #\A))
(check (eq "aa" "aa"))
(check (eq (list 1 2) (list 1 2)))

(format t "eql~%")
(check (eql 'a 'a))
(check (eql 100 100))
(check (eql 100 100.0))
(check (eql #\a #\a))
(check (eql #\a #\A))
(check (eql "aa" "aa"))
(check (eql (list 1 2) (list 1 2)))

(format t "equal~%")
(check (equal 'a 'a))
(check (equal 100 100))
(check (equal 100 100.0))
(check (equal #\a #\a))
(check (equal #\a #\A))
(check (equal "aa" "aa"))
(check (equal (list 1 2) (list 1 2)))

(format t "equalp~%")
(check (equalp 'a 'a))
(check (equalp 100 100))
(check (equalp 100 100.0))
(check (equalp #\a #\A))
(check (equalp "aa" "aa"))
(check (equalp (list 1 2) (list 1 2)))
```

## Conclusion
Understanding the various equality predicates in Common Lisp is essential for **writing predictable and optimized code**.

- Use `eq` for **object identity comparisons**.
- Use `eql` for **numbers and characters**.
- Use `equal` for **structural comparison of lists and strings**.
- Use `equalp` for **case-insensitive string comparison and type-flexible numeric comparison**.

By choosing the appropriate equality predicate, you can write **robust and efficient Lisp programs**.

