---
layout: post
lang: en
ref: "create-and-use-c-libraries"
title: "Creating and Using Static/Dynamic (Shared) Libraries"
date: 2026-07-21 22:00:00 +0900
categories: ["programming"]
tags: ["c library", "static library", "shared library", "dynamic library"]
---

In this post, I want to talk about libraries.
Let's compare **how to create static and dynamic libraries in C on Linux**, and look at **the key differences between them**.
What a library is, and what the static/dynamic concepts even mean — there's already plenty of material out there, so I won't cover that here.
Building on the basic creation and usage, I'll focus mainly on what differs at the binary level and why those differences arise.

The test and analysis environment is GNU/Linux-based. Terms and commands differ on other platforms, so please refer to other articles for those. For the basics of object file creation and the compilation pipeline, see [this post]({% post_url 2026-07-01-c-build-step %}).

Here's the simple example code we'll use for the analysis.

**my_math.h**
```c
#ifndef MY_MATH_H
#define MY_MATH_H

#include <stdbool.h>

// Arithmetic operations
int add(int a, int b);
int sub(int a, int b);

// Compare
bool equal(int a, int b);
bool greater(int a, int b);
bool less(int a, int b);

#endif
```

**my_arithmetic.c**
```c
#include "my_math.h"

int add(int a, int b) {
        return a + b;
}

int sub(int a, int b) {
        return a - b;
}
```

**my_compare.c**
```c
#include "my_math.h"

bool equal(int a, int b) {
        return (a == b);
}

bool greater(int a, int b) {
        return (a > b);
}

bool less(int a, int b) {
        return (a < b);
}
```

**main.c**
```c
#include "my_math.h"

int main(void) {
        int a = add(1, 3);   // 4
        int b = sub(3, 1);   // 2
        return a + b;        // 6
}
```

## Creating a Static Library
```bash
$ gcc -c my_arithmetic.c my_compare.c   # without -o, the output is auto-named {name}.o
$ ar crs libmymath.a my_arithmetic.o my_compare.o  
```

Just two lines and you've got a static library.
The second argument to the `ar` command (`libmymath.a`) is exactly the ***static library*** we just created.

`ar` is a GNU utility for creating, modifying, and extracting archive files.
The "archive" here just means it bundles the given files into one — nothing special happens to them.
In particular, it does not compress the object files!

The `crs` options we used, one by one:
- `c` : create a new archive if one doesn't exist yet.
- `r` : insert the object files into the archive. If a member with the same name already exists, replace it.
- `s` : generate an index (symbol table).

## Creating a Dynamic/Shared Library
```bash
$ gcc -fPIC -c my_arithmetic.c my_compare.c
$ gcc -shared my_arithmetic.o my_compare.o -o libmymath.so
```

This one also takes just two lines.
The ***dynamic/shared library*** created with GCC is `libmymath.so`.

But here it's worth splitting things into two stages.
First, the stage that creates the object files used as the raw material for the library, and second, the stage that builds the library out of those object files.

---

In the first stage, on top of the usual option for producing a relocatable object file, `-fPIC` has been added.
***PIC*** stands for *Position-Independent Code* — literally, **code that doesn't depend on its position**.

The binary-level analysis and practical use of this option is something I'll cover in another post, so here I'll only explain it briefly.
I found it easiest to understand PIC by breaking it down word by word.

- **Position** : the position — more precisely, the position in memory. In other words, a memory address.
- **Independent** : has no dependency on the Position above, i.e., the memory address.
- **Code** : the code here means the code (text) section of the object file. This section holds the machine instructions the CPU executes.

So, putting those meanings together, we can read it as: **"the machine-code instructions in a (relocatable) object file produced with this option don't run at any single fixed memory address."**

> You can also use -fpic instead of -fPIC. For the difference, check the gcc man page.

In the second stage — building the library — we used `-shared`.
Again, nothing dramatic: it's just the option that says "combine these relocatable object files into a dynamic/shared library."

## Static vs. Dynamic/Shared Library: Comparing How They're Built
Here's a table summarizing how each library is built, and the differences.

| Type | Extension | Object file option | Built by | Library option | 
| - | - | - | - | - | 
| Static | .a | -c | ar | crs | 
| Dynamic/Shared | .so | -c -fPIC | gcc | -shared |

Both libraries conventionally take the *lib* prefix.

---
Now that we've created the libraries, let's look at how to use them from an executable.

## Linking a Static Library
```bash
$ gcc main.o -L. -lmymath -o math.out
```
When linking without libraries, we simply listed the relocatable object files. But when linking a library, we need two command-line arguments.

**The `-L` option specifies the directory where the library to link lives**, and **the `-l` option specifies the library name**. 

So `-L. -lmymath` means the linker will try to find a library named `libmymath.a` in the current directory (.).

With those options, the linker finds the library, links it with main.o, and produces the final executable, math.out.

## Linking a Dynamic/Shared Library
```bash
$ gcc main.o -L. -lmymath -o math.out
```

Fundamentally, it's exactly the same as linking a static library.
The `-L. -lmymath` options look for `libmymath.so` in the current directory.

Whether it's a static library or a dynamic/shared one, from the linker's point of view what matters first is where the library is and what it's called.

### When a Static and a Dynamic/Shared Library Live in the Same Path
As we saw, the linking options are identical for both. So what happens when two different libraries with the same name sit in the same location?

For example, what if both `libabc.a` and `libabc.so` are under the `/opt` directory?
```bash
$ gcc main.o -L/opt -labc -o abc.out   # does -labc mean libabc.a? or libabc.so?
```
In this case, the linker looks for `libabc.so` under `/opt` — **that is, it prefers the dynamic/shared library.**

If you want to link the static library instead, use the `-static` option.
```bash
$ gcc main.o -static -L/opt -labc -o abc.out   # links libabc.a
```

### Linking a Library Without the lib Prefix
```bash
$ gcc main.o -L./libs -l:mmm.a -o try.out
```
After the `-l:` option you can write the library file name directly. (Might depend on your compiler version...)

That said, if you try to link a library without the prefix, modern compilers will kindly tell you how to fix it, so don't worry too much about it.

## Running an Executable Linked Against a Static Library
```bash
$ ./math.out || echo $?
6
```
We can check `math.out`'s exit code (its return value), `6`, by printing `$?`.

When you link a static library, the final executable runs exactly the same as one that uses no library at all. Really — not a single difference. You just run the executable, and that's it.

## Running an Executable Linked Against a Dynamic/Shared Library
```bash
$ LD_LIBRARY_PATH=. ./math.out || echo $?
6
```
The important thing here is setting the `LD_LIBRARY_PATH=` environment variable.

This variable holds the location of the dynamic/shared library (.so) files that the dynamic loader will load at runtime.
The key point is that **when the process starts, the dynamic loader has to know where to find the libraries it needs to load into memory.**

> There are actually a few more ways to use dynamic/shared libraries.
> But for now the basic usage and concepts are enough, so I'll leave those out of this post.

---

So far we've looked at how to build and use libraries on the surface.
But what I really want to dig into is what's underneath.
When you link a static library, does the binary code actually get embedded straight into the executable object file? If so, how do a dynamic library's symbols get written into the executable object file? How does the dynamic loader actually work? There's still a lot I'm curious about and want to see for myself.
