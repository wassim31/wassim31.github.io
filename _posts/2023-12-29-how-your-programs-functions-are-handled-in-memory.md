---
layout: post
title: "How your program's functions are handled in memory?"
date: 2023-12-29 15:40:54 +0000
description: "Because I always forget how the kernel handles a program's functions in the call stack, I wrote it down. Function prologue, %rbp, %rsp, and calling conventions."
tags: [linux, c, x86, memory]
canonical_devto: https://dev.to/wassim31/how-your-programs-functions-are-handled-in-memory--afl
image: /images/posts/how-your-programs-functions-are-handled-in-memory/cover.png
---

Because I always forget how the kernel is handling the program's functions in the call stack, I'll write about it so I can come back after months, or someone else will :)

So hello, let's do it in C on an x86-64 bits machine. Why? Because it's my favorite language.

## From C to assembly

First, when you compile your C code:

```bash
gcc -c main main.c
```

you get an object file `main.o` that will be linked later (manually or dynamically) with other already compiled standard code (like your famous `printf()`), and you get an executable file, ELF64 in my case.

When you disassemble it, either with objdump or gdb, you can see the assembly code of your program:

```bash
gdb ./main
disassemble function
```

![Disassembly of main and function showing the function prologue](/images/posts/how-your-programs-functions-are-handled-in-memory/functions.png)

You'll find some sections like `.text`, `.bss` or `.data`, the heap, and the stack. These are the sections of the virtual address space of your program.

What concerns us is the `.text` section: it contains the code, the instructions to be executed. For example `int x = 3` will have the equivalent of `movl $0x3,-0x4(%rbp)`.

Okay, good. What concerns us now is how the functions `main()` and `function()` are handled in the stack, the section where the functions reside.

## The registers involved

The CPU has some special registers called `%rbp` and `%rsp` (the base pointer and the stack pointer) to manipulate the scope of the functions:

- `%rbp` is used to access the memory words by adding values to it, like the example we had: `movl $0x3,-0x4(%rbp)`
- `%rsp` always points to the top of the stack.

## The function prologue

Here is the process, step by step:

1. The first thing we need to do is push the value of the old `%rbp`, because it's used by other functions to do the same thing. In this case that's the `main()` function, which called `function()`.

2. Then we push the return address to the stack, so the `main()` function resumes its execution where it stopped (in this case it will return to line 13). This information is retrieved from register `r14` in ARM processors, for example.

3. Then we'll set the new value of the base pointer `%rbp` (the one we said is used for accessing memory words in the scope) to the value of `%rsp`, so the top of the stack.

This process of handling `%rbp` and `%rsp` is called the **function prologue**, and it's done at each function start, as you can see in the assembly code.

This is how the initialization of the function scope is done, similarly across multiple architectures and systems.

## Calling conventions

Now we'll use the function call stack by putting the local variables, the function parameters, etc. These specific details follow calling conventions:

<https://lnkd.in/eUNNAiXH>

One of the conventions to follow on x64 architectures, for example, is **System V**. For function parameter handling, the registers are used like this: the first parameter is placed in `rdi`, the second in `rsi`, the third in `rdx`, and then `rcx`, `r8` and `r9`. Only the 7th argument and onwards are passed on the stack. The left most parameter is passed first on the stack, then the old value of the base pointer, and then it's followed by the local variables of the function.
