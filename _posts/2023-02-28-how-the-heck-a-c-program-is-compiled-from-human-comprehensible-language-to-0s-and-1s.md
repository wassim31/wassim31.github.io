---
layout: post
title: "How the heck a C program is compiled from human comprehensible language to 0's and 1's"
date: 2023-02-28 01:38:09 +0000
description: "You may wondered how the programming languages lexemes are turned into a series of 0's and 1's aka..."
canonical_devto: https://dev.to/wassim31/how-the-heck-a-c-program-is-compiled-from-human-comprehensible-language-to-0s-and-1s-2kkg
image: /images/posts/how-the-heck-a-c-program-is-compiled-from-human-comprehensible-language-to-0s-and-1s/cover.jpeg
---

You may wondered how the programming languages lexemes are turned into a series of 0's and 1's aka binary files ?

![Compilation pipeline from source code to running process](/images/posts/how-the-heck-a-c-program-is-compiled-from-human-comprehensible-language-to-0s-and-1s/pic.png)

Figure 2.11 - Book reference : Operating system concepts.

Let's discover that together step by step :

Usually programs a.out or program.exe ( games , word processors , web browsers ..) resides in your disk as a binary executable file , and in order to execute one of them , you need to fetch it from disk and put in memory , and it waits until it's scheduled to run and becomes a running program also known as a "process" which have a specific amount of memory addresses and executed by the units of cpu's core with the help of specific registers like program counters , IR ... and general-purpose registers .

but we don't write operating systems and Enterprise systems with binary right?

We need a human readable language , let's chose a compiled programming language like C to know the process of compilation .

After writing your program.c source code with human readable C , the source code is compiled & assembled to a relocatable object file program.o ( machine code that can be disassembled and turned into a geek-readable assembly ) ( we will talk about phases of compilation ( lexing , parsing , AST...) in future articles ) .

So we have now a object file , but we may called several functions like printf() which is also written by system programmers before to give us a layer of abstraction and increase our productivity , and compiled aka turned into an object file , it's part of GCC .

we need now to link all those object files and make them a single binary executable file that can be loaded into memory , this mechanism is done by what we call a " Linker " . (this is static linking)

the mechanism of loading the binary executable file into memory so it can be executed by one of the cpu's core is done by the loader , it's also responsible of addresses reallocation in memory..

You may heard about DLL ( dynamically linked libraries ) files in Windows right ? now we speak about dynamic linking . actually , most systems allows programs to dynamically link the libraries even if the program is already loaded and started execution ( that's good from memory optimization prospective, because the programmer may include a library and doesn't use it during runtime , i am sure we all did that before )

you know that modern systems allows multiple processes to share DLL ? yes they do!

And now you have a binary executable program into memory , also known as a process , with it's own memory addresses .

I will talk deeper next time about the compilation phases , and how the cpu fetch & decode & execute instructions & data from the bounded virtual memory of the process.
