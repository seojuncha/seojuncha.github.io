+++
date = '2025-02-17'
title = "Common Lisp Character & String Guide: Comparison, Conversion, and Manipulation"
tags = ["common lisp", "common lisp string", "common lisp character"]
description = "Learn how to handle characters and strings in Common Lisp with this complete guide. Covers character representation, ASCII conversion, string comparison, substring extraction, number conversion, and various string manipulation techniques with practical examples"
draft = true
+++

In Common Lisp, **characters are represented using the `#\` prefix**, allowing support for ASCII and Unicode character sets. Characters are **self-evaluating objects**, meaning they can be used without `quote`. Additionally, special characters such as **whitespace (`#\space`) and newline (`#\newline`)** are available. Various functions are provided for character comparison, with case-sensitive (`char=`) and case-insensitive (`char-equal`) options. Common Lisp also supports conversion between characters and their ASCII codes.

Strings are simply sequences of characters and are **enclosed in double quotes (`" "`)**. String comparison works similarly to character comparison, where appropriate functions must be chosen based on case sensitivity. **A variety of string manipulation functions** (`length`, `subseq`, `concatenate`, `string-upcase`, `string-downcase`, `search`, etc.) **are available**, and `coerce` allows **conversion between strings and lists**. Additionally, `parse-integer` is used to convert strings to numbers, while `princ-to-string` converts numbers to strings.

## Character Representation in Common Lisp
In Common Lisp, characters are represented by prefixing them with `#\`. The following 94 characters can be represented:

```
! " # $ % & ' ( ) * + , - . / 0 1 2 3 4 5 6 7 8 9 : ; < = > ?
@ A B C D E F G H I J K L M N O P Q R S T U V W X Y Z [ \ ] ^ _
` a b c d e f g h i j k l m n o p q r s t u v w x y z { | } ~
```

Characters in Common Lisp are self-evaluating objects, just like numbers. This means they can be used directly in the REPL without parentheses or `quote`.

### Whitespace and Newline
Common Lisp provides specific characters for whitespace and newline representation:

- `#\space`
- `#\newline`

> Special characters such as `#\tab` and `#\linefeed` may be available depending on the Lisp implementation. Ensure their availability before using them!

Unlike in C, where escape sequences (`\t`, `\n`) are used, Common Lisp explicitly defines such characters.

Example of printing whitespace to standard output:

```lisp
(format t "~a~a" #\space #\a)
```

**Output:**
```
 a
NIL
```

> The keywords `space` and `newline` in `#\space` and `#\newline` are case-insensitive, meaning `#\space` and `#\spACe` are equivalent.

### Character Comparison
Comparing characters is a common operation in programming. Unlike numbers, character comparison in Common Lisp requires special functions since case sensitivity is a factor.

| Number | Case-Sensitive | Case-Insensitive |
|:-:|:-|:-|
|`=`|`char=`|`char-equal`|
|`/=`|`char/=`|`char-not-equal`|
|`>`|`char>`|`char-greaterp`|
|`<`|`char<`|`char-lessp`|
|`>=`|`char>=`|`char-not-lessp`|
|`<=`|`char<=`|`char-not-greaterp`|

Example:
```lisp
(char= #\a #\A)       ; NIL (`a` and `A` are different)
(char-equal #\a #\A)  ; T
```

> Character comparison follows the ASCII table order.

### Character-to-Number Conversion
#### Convert to ASCII Code
Common Lisp distinguishes character objects from number objects, requiring explicit conversion using `char-code`:
```lisp
(char-code #\A)  ; 65
```

#### Convert Number to Character
To convert numbers to characters, use `code-char`:
```lisp
(code-char 65)  ; #\A
```

#### Convert Digit Characters to Numbers
To check if a character is a digit and retrieve its numeric value, use `digit-char-p`:
```lisp
(digit-char-p #\1)  ; 1
(digit-char-p #\a)  ; NIL
```
This function can also determine if a character is numeric:
```lisp
(let ((is-digit (digit-char-p #\a)))
  (if is-digit
    (format t "is digit~%")
    (format t "is not digit~%")))
```
**Output:**
```
is not digit
```

## String Representation in Common Lisp
Strings are sequences of characters enclosed in double quotes. Escape sequences (e.g., `\"`) are supported:
```lisp
(defparameter *my-string* "this is a string")
(format t "~a~%" *my-string*)  ; this is a string
```

### String Comparison
String comparison functions mirror character comparison functions but work on entire strings.

| Number | Case-Sensitive | Case-Insensitive |
|:-:|:-|:-|
|`=`|`string=`|`string-equal`|
|`/=`|`string/=`|`string-not-equal`|
|`>`|`string>`|`string-greaterp`|
|`<`|`string<`|`string-lessp`|
|`>=`|`string>=`|`string-not-lessp`|
|`<=`|`string<=`|`string-not-greaterp`|

Example:
```lisp
(string= "ab" "ab")   ; T
(string/= "ab" "aa")  ; 1
```
Unlike character comparison, string comparison functions (except `string=` and `string-equal`) return a **mismatch index** instead of `T` or `NIL`.

### Basic String Functions
| Functionality | Function |
|:-|:-|
|String length|`length`|
|Character at index|`char`|
|Concatenation|`concatenate`|
|Substring extraction|`subseq`|
|Case conversion|`string-upcase`, `string-downcase`|
|String search|`search`|
|String-list conversion|`coerce`|
|String-number conversion|`parse-integer`, `princ-to-string`|

Examples:
```lisp
(length "abc")  ; 3
(subseq "hello world" 0 5)  ; "hello"
(string-upcase "hello")  ; "HELLO"
(string-downcase "WORLD")  ; "world"
(search "store" "fruit store")  ; 6
```

### Converting Strings to Numbers
Convert a numeric string to an integer:
```lisp
(parse-integer "123")  ; 123
```
With additional options:
```lisp
(parse-integer "0xFF" :radix 16 :start 2)  ; 255
```
Convert a number to a string:
```lisp
(princ-to-string 123)  ; "123"
```

## Conclusion
Common Lisp provides a powerful set of functions for handling characters and strings. By understanding these functions, developers can efficiently manipulate text data, making Lisp programming more effective and intuitive.

