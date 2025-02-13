+++
date = '2025-02-12'
title = "Understanding Variable Assignment in Common Lisp: A Practical Guide"
tags = ["common lisp", "variable", "setq", "let", "defparameter", "defvar"]
+++
Learn how variable assignment works in Common Lisp, including global and local variables, `setq`, `let`, and `defvar`. This practical guide covers key differences from modern languages and best practices for efficient Lisp programming.

<!--more-->

Using variables is one of the most fundamental aspects of programming. However, Common Lisp differs from other languages in how it handles variable assignment. In this post, we will explore the various ways to assign variables in Common Lisp, examine real-world examples, and discuss important considerations for each method.

# Defining Global Variables: `defvar` and `defparameter`
The first variable assignment methods we'll explore are ***global variable*** declarations using `defvar` and `defparameter`. These are the easiest ways to create variables in a Lisp program, and both define variables with global scope.

```lisp
(defvar *a*)    ; Declare variable *a* without assigning a value
(defvar *b* (list 1 2 3))  ; Assign a list (1 2 3) to variable *b*
(defvar *my-array* (make-array 2))  ; Create a 2D array and assign it to *my-array*
```
You can use `defparameter` in a similar way:

```lisp
(defparameter *a*)  ; Error: defparameter requires an initial value
(defparameter *b* '(1 2 3))
(defparameter *c* nil)
```
Unlike `defvar`, `defparameter` **requires an initial value** when defining a variable.

> As shown in the example, Lisp conventionally surrounds global variable names with `*` (asterisks).

One of the key differences between Lisp and other languages like C-based languages is how variable scope is determined. In many languages, whether a variable is global or local depends on its syntactic position in the code. However, in Lisp, scope is **explicitly controlled** using specific keywords such as `defvar` and `let`.

# Defining Local Variables: `let`
In addition to global variables, you can explicitly create ***local variables***. As in other programming languages, local variables are only valid within a specific scope. In Lisp, you can use let to define variables with a local scope.

```lisp
(let ((a 1)   ; Initialize local variable a with 1
       c)     ; Declare local variable c (initialized to nil)
  (format t "~d~%" a))
(format t "~d~%" a)  ; Error: a is only valid within the let scope
```

Here, `a` and `c` exist only inside the let block. Once execution leaves the block, these variables are no longer accessible. This ensures that local variables do not interfere with other parts of the program.

## Using `let` Inside Functions
In Lisp, function parameters are also created as ***local variables*** within the function's scope.

```lisp
(defun foo (number)   ; Function parameter number, scoped locally
  (incf number)
  (format t "number: ~a~%" number))

(foo 20)  ; Call foo function with argument 20
(format t "number: ~a~%" number)   ; Error: number is only valid inside the function
```

Just like in other programming languages, function arguments are assigned as local variables within the function. This means that once the function execution is complete, these parameters cannot be accessed from outside the function.

Now, what if you need to create additional local variables inside a function? In such cases, you can use `let` within the function.

```lisp
(defun foo (number)  ; Function parameter number, scoped locally
  (let ((plus-one 0))   ; Declare local variable plus-one
    (setq plus-one (+ number 1))  ; Assign number + 1 to plus-one
    (format t "plus: ~d~%" plus-one)))

(foo 20)  ; Call foo function
```
In this example, we define a new local variable `plus-one` inside the function foo using `let`. This allows us to use it exclusively within the function without affecting the global environment.

## Relationship Between Existing Variables and `let`
Since let creates a new local variable, it can **shadow (hide)** an existing variable with the same name.

```lisp
(defvar a 10)   ; Bind a to 10 globally
(let ((a 5))    ; Create a new local variable a, bound to 5
  (format t "~d~%" a))
(format t "~d~%" a)  ; Outside let, prints 10 (global a)
```

Here, `(defvar a 10)` defines a global variable a bound to the value `10`. When we introduce `let`, a new local variable named a is created within the `let` block, but this is ***not*** the same as the global a. Instead, the local a exists only within the let block and temporarily **shadows the global `a`**.

This is one of the reasons why global variables in Lisp are conventionally named with surrounding `*` (e.g., `*a*`). This naming convention helps distinguish them from local variables and avoid unintended shadowing.

# Modifying Variable Values: `setq`
So far, we've explored `defvar`, `defparameter`, and `let`, which are used to **create** new variables. (More precisely, they create new bindings for variables.) Once a variable is created, you can **modify** its value using `setq`.

```lisp
(defparameter *a* nil)
(setq *a* 'a)
```
Here, `defparameter` initializes `*a*` with `nil`, and later, `setq` updates `*a*` to `'a`. Unlike `defvar` and `defparameter`, `setq` does **not** create a new variable—it only assigns a new value to an existing one.

## Always Use `setq` with `let`, `defvar`, or `defparameter`
`setq` **does not create a new variable**—it only modifies the value of an existing one. For this reason, it’s best to use `setq` together with a variable declaration, such as `let`, `defvar`, or `defparameter`, rather than using it alone.

```lisp
(let ((num 10)  ; Variable num initialized to 10
      (n nil))  ; Variable n initialized to nil
  (format t "~d~%" num)    ; Prints 10
  (setq num (+ num 2))     ; num = 12
  (setq n (+ num 2))       ; n = 14
  (format t "~d ~d~%" num n))  ; Prints 12 14
```
> **What happens if you use `setq` alone?**
> 
> If you use `setq` without first declaring the variable, Lisp automatically assigns it as a **global variable**.  
> This goes against Lisp’s explicit variable scoping rules and can easily lead to unintended mistakes.
> In fact, in [SBCL (Steel Bank Common Lisp)](https://www.sbcl.org/), using `setq` without a prior declaration will trigger a warning message.

# Defining Constants: `defconstant`
Just like other programming languages, Lisp allows you to create **constant variables** that cannot be modified.

```lisp
(defconstant +name+ "seojun")   ; Define a string constant
(format t "~a~%" +name+)

(setq +name+ "who")  ; Error: Constants cannot be modified
```

> By convention, constant variable names are surrounded by `+` signs.

Using `defconstant`, you can declare a variable whose value **cannot be changed** after its initial assignment. If you attempt to modify a constant using setq, Lisp will raise an error.

# Object and Symbol Binding
In Lisp, variables follow the concept of ***binding***. Instead of directly storing values, variables store **references to objects**. In Lisp, **everything**, including numbers and characters, is treated as either an *object* or a *symbol*.

> Concepts like objects, symbols, and s-expressions will be covered in a separate post.

For example, when you declare a global variable using `(defvar a 1)`, Lisp creates a **variable-to-value binding** as shown below:

```
 +---+      +---+
 | 1 | <--- | a |
 +---+      +---+
Object     Variable(Symbol)
```

# Differences from Modern Programming Languages
As we have explored so far, Lisp provides a more **explicit approach** to variable assignment and scope compared to modern languages like Python and C. In most modern programming languages, a variable's scope (local or global) is automatically determined based on its **contextual placement** within the code.

For example, in C:

```c
int a = 10;     // Global variable a
void foo() {
  int a = 20;   // Local variable a
  printf("%d", a);  // Prints 20
}
printf("%d", a);  // Prints 10
```

And in Python:

```python
a = 10    # Global variable a
def foo():
  a = 20  # Local variable a
print(a)  # Prints 20
```

However, Lisp explicitly specifies how variables are assigned and scoped using **operators and macros**, rather than relying on implicit scoping rules. This **declarative approach** allows for better **compiler optimization**, **debugging efficiency**, and **enhanced flexibility—features** that are not easily achievable in many modern languages.

# Conclusion
Recently, while working on the [*From the Transistor*](https://github.com/seojuncha/fromthetransistor-fork) project, I’ve been writing a compiler using Common Lisp. Since Lisp differs significantly from the languages I’m more familiar with, I initially struggled with several concepts—one of the major challenges being variable assignment and usage.

In this post, I focused on **practical aspects** of variable handling in Lisp, leaving out more complex topics, so that you can apply these concepts immediately in real programming scenarios.

In the next post, we’ll dive into another essential part of programming: **conditional statements** and **loops**. Stay tuned!

# Related Resources
- **LispWorks HyperSpec**
  - [defvar and defparameter](https://www.lispworks.com/documentation/HyperSpec/Body/m_defpar.htm)
  - [let](https://www.lispworks.com/documentation/HyperSpec/Body/s_let_l.htm#let)