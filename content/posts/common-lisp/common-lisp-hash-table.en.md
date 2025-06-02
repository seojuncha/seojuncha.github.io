---
title: "Understanding Hash Tables in Common Lisp"
date: 2025-03-14
tags: [Common Lisp, Common Lisp Hash Table]
description: ""
draft: true
---

Recently, while working on the "From the Transistor" project and building a compiler, I encountered a need for a [**hash table**](https://en.wikipedia.org/wiki/Hash_table) to store keywords in C. In Common Lisp, a **hash table is an efficient data structure for storing key-value pairs**. Unlike arrays or lists, hash tables are designed for **fast lookups based on keys**.

This post explains how to create and use hash tables in Common Lisp.

## 1. Creating a Hash Table (`make-hash-table`)

To create a hash table in Common Lisp, use the `make-hash-table` function.

```lisp
(defparameter *my-table* (make-hash-table))
```

By default, this creates a hash table that compares keys using **eq test**. If you want to use a different comparison method, specify the `:test` option.

```lisp
(defparameter *my-table-equal* (make-hash-table :test 'equal))
```

### Comparison Methods

- `eq` (default): Compares object identity.
- `eql`: Can compare numbers and characters.
- `equal`: Compares lists or strings.
- `equalp`: Case-insensitive and structural comparison.

For a more detailed explanation of these comparison methods, see my post on [Understanding Equality in Common Lisp](/posts/common-lisp/equality-predicate-in-common-lisp).

## 2. Data Lookup and Insertion

### Data Lookup (`gethash`)

Use `gethash` to retrieve values.

```lisp
(gethash "key1" *my-table*) ; => "value1"
(gethash "key2" *my-table*) ; => 42
(gethash "key3" *my-table*) ; => NIL
```

The `gethash` accessor not only retrieves a key’s value but also provides an indicator of its existence. 

To check whether a key exists, use the second return value of `gethash` with `multiple-value-bind`:

```lisp
(multiple-value-bind 
  (value present)   ; Bind returned values to variables
  (gethash "key1" *my-table*)  ; Evaluation
  (if present   ; Use predicate for condition check
    (format t "There is ~a.~%" value)
    (format t "There isn't ~a.~%" value)))
```

However, note that `gethash` alone is **not a reliable way to check for key existence** because it returns `NIL` if a key is not present **or** if the key exists but is assigned `NIL`. Let's see this in an example:

```lisp
;;; Using gethash directly in an if statement
(defun check-from-gethash (key table)
  (if (gethash key table)
    (format t "exist~%")
    (format t "not exist~%")))

;;; Using multiple-value-bind with predicate
(defun check-from-value-bind (key table)
  (multiple-value-bind (value present) (gethash key table)
    (if present
      (format t "exist~%")
      (format t "not exist~%"))))

;;; Create a hash table
(defparameter *my-table* (make-hash-table))

;;; Assign NIL to key1 and key2
(setf (gethash 'key1 *my-table*) nil)
(setf (gethash 'key2 *my-table*) nil)

;;; Table has two keys
(format t "size: ~a~%" (hash-table-count *my-table*))

;;; Compare different key lookup methods
(check-from-gethash 'key1 *my-table*)
(check-from-value-bind 'key1 *my-table*)
```

**Output:**
```shell
size: 2
not exist
exist
```

The table contains two keys, both assigned `NIL`. `gethash` alone cannot reliably determine key existence.

### Data Insertion (`setf` + `gethash`)

To insert values into a hash table, use `setf` with `gethash`:

```lisp
(setf (gethash "key1" *my-table*) "value1")
(setf (gethash "key2" *my-table*) 42)
```

> `setf` is an extended version of `setq`, as explained in my post on [Variable Assignment in Common Lisp](/posts/common-lisp/variable-assignment-in-common-lisp/).

Compared to other languages like Python or C++, which support hash maps natively, Lisp's syntax might seem a bit complex:

```python
my_dict = dict()
my_dict["key1"] = 1
my_dict["key2"] = 2
```

```c++
#include <map>
std::map<std::string, int> my_map;
my_map["key1"] = 1;
my_map["key2"] = 2;
```

In Lisp, `gethash` plays the role of bracket notation in these languages.

## 3. Data Deletion

### Deleting Data (`remhash`)

To remove a specific key, use `remhash`:

```lisp
(remhash "key1" *my-table*)
```

---

## 4. Iterating Through a Hash Table (`loop` and `maphash`)

To iterate over all key-value pairs in a hash table, use `maphash` or `loop`.

### Using `maphash`

```lisp
(maphash (lambda (key value)
           (format t "~A => ~A~%" key value))
         *my-table*)
```

### Using `loop`

```lisp
(loop for key being the hash-keys of *my-table*
      using (hash-value value)
      do (format t "~A => ~A~%" key value))
```

## 5. Checking Size and Clearing Hash Tables

### Checking Size (`hash-table-count`)

Get the number of keys in a hash table:

```lisp
(hash-table-count *my-table*)
```

### Clearing a Hash Table (`clrhash`)

To remove all entries from a hash table, use `clrhash`:

```lisp
(clrhash *my-table*)
```

> Clearing a table does not delete the variable itself, only its contents.

## Conclusion

Hash tables are useful for efficient data storage and fast lookups. Since Common Lisp provides built-in support for hash tables, they can be easily utilized. By experimenting with the `:test` option and different comparison methods, you can gain a deeper understanding of key comparisons in Common Lisp.
