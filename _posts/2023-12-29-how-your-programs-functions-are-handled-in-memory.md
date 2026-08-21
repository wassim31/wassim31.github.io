---
layout: post
title: "How your program's functions are handled in memory ?"
date: 2023-12-29 15:40:54 +0000
description: "Because i always forget how the kernel is handling the program's functions in the call stack , i'll..."
tags: [linux, c, x86, memory]
canonical_devto: https://dev.to/wassim31/how-your-programs-functions-are-handled-in-memory--afl
image: /images/posts/how-your-programs-functions-are-handled-in-memory/cover.png
---

Because i always forget how the kernel is handling the program's functions in the call stack , i'll write about it so i come back after months or someone else will :) .

So hello , let's do it in C in a x86-64bits machine, why ? because it's my favorite language .

first , when you compile your C code ( gcc -c main main.c ), you get an object file main.o that will be linked later ( manually or dynamically ) with other already compiled standard code ( like your famous printf() ) , you get an executable file , ELF64 in my case.
okay , when you disassemble either with objdump or gdb , you can see the assembly code of your program :
gdb ./main
disassemble function

![Disassembly of main and function showing the function prologue](/images/posts/how-your-programs-functions-are-handled-in-memory/functions.png)

you'll find some sections like .text , .bss or .data , heap and the stack these are the sections of the virtual address space of your program .

what concern us is the .text section , it contains the code ( instructions to be executed [ for example int x = 3 will have the equivalent of movl  $0x3,-0x4(%rbp)]) 

okay , good , what concerns us is how the functions main() and function() are handled in the stack (the section where the functions resides) .

before , the CPU unit has some special registers called %rbp , %rsp ( the base pointer and the stack pointer ) to manipulate the scope of the functions.

%rbp : is used to access the memory words by adding values to it like the example we had : movl  $0x3,-0x4(%rbp)
%rsp : will always points to the top of the stack.

so the first thing we need to do is to push the value of the old %rbp ( because it's used by other functions to do the same thing , in this case the main() function which called the function() ) .

then , we push the return address to the stack so the main() function resumes it's execution where it stopped ( in this case it will return to line 13). , this information is retrieved from register r14 in ARM processors for example.

then we'll put the new value of the base pointer %rbp ( the one we said used for accessing memory words in the scope ) to the value of %rsp , so the top of the stack .

this process of handling the %rbp and %rsp is called function prologue and it's done in each function start ( as you see in the assembly code ) .

this is how it's done with the initialization of the function scope in multiple architectures and systems similarly.
 
now we'll use the function call stack by putting the local variables , the function parameters etc , these specific details follows calling conventions:
https://lnkd.in/eUNNAiXH

one of the conventions to follow in x64 architectures for example is SYSTEM V
the registers used for function parameters handling 
The first is placed in rdi, the second in rsi, the third in rdx, and then rcx, r8 and r9. Only the 7th argument and onwards are passed on the stack , the left most parameter is passed first on the stack , and then the old value of the base pointer , then it's followed by the local variables of the function.
