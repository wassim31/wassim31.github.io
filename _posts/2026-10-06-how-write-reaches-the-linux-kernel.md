---
layout: post
title: "How write() Gets From Your C Code Into the Linux Kernel"
date: 2026-10-06 10:00:00 +0000
description: "Tracing a single write() call on x86-64 Linux from glibc through the syscall instruction, entry_SYSCALL_64, the task's kernel stack, and back."
tags: [c, linux, systemsprogramming, kernel]
---

Take the smallest C program that actually does something:

```c
#include <unistd.h>

int main(void)
{
    write(1, "hello", 5);
    return 0;
}
```

One function call. But between that call and "hello" actually reaching your terminal, the CPU changes privilege level, your thread swaps stacks, and execution jumps into code that every other process on the machine is also running at that exact moment. Let's trace the whole thing.

Every program you run, Bash, Firefox, this one, executes in **user mode**. It can touch its own memory, but it can't poke at kernel memory, page tables, or raw hardware. The Linux kernel runs in **kernel mode**, where none of those restrictions apply. Getting from one to the other isn't a function call, it's a CPU privilege transition, and it only happens at points the kernel explicitly allows.

<p class="diagram">
  <img class="diagram-light" src="/images/posts/how-write-reaches-the-linux-kernel/01-modes-light.svg" alt="User mode containing the application, glibc, and user virtual memory, with a controlled entry point down into kernel mode containing the Linux kernel, VFS, scheduler, drivers, and networking">
  <img class="diagram-dark" src="/images/posts/how-write-reaches-the-linux-kernel/01-modes-dark.svg" alt="User mode containing the application, glibc, and user virtual memory, with a controlled entry point down into kernel mode containing the Linux kernel, VFS, scheduler, drivers, and networking">
</p>

## POSIX, glibc, and the actual syscall

`<unistd.h>` declares `write()` because POSIX specifies it, as an interface, not an implementation. POSIX says what `write(fd, buf, count)` has to do, write up to `count` bytes from `buf` to `fd`, return the number written or an error, but it says nothing about registers, instructions, or privilege levels. On a normal dynamically linked Linux binary, the code that actually runs when you call `write()` comes from glibc, and glibc is the thing that has to turn that abstract contract into something a specific CPU understands.

Not every libc function needs the kernel at all. `strlen()` just walks memory you already own and never leaves user mode. `write()` is different: it has to ask the kernel to move bytes into a file descriptor the kernel manages, so sooner or later it has to cross that privilege boundary.

## What the compiler generates

Compile the program for x86-64 and `write(1, "hello", 5)` turns into something like this, following the ordinary x86-64 System V calling convention:

```asm
mov    $5, %edx
lea    message(%rip), %rsi
mov    $1, %edi
call   write@PLT
```

```text
RDI = 1                 first argument
RSI = address("hello")  second argument
RDX = 5                 third argument
```

`call write@PLT` is still a completely ordinary userspace function call. Nothing privileged has happened yet. The PLT entry is part of how dynamic linking resolves `write` to wherever glibc actually lives in memory, and once that resolves, execution lands inside glibc's own `write()`, which is where the real work starts.

## Crossing into the kernel

Here's the part that trips people up: the registers above are the *C calling convention*, how arguments get passed between user-space functions. The Linux x86-64 syscall ABI is a different, narrower contract, and glibc has to re-pack things into it before it can ask the kernel for anything.

```text
RAX = syscall number
RDI = fd
RSI = buffer
RDX = count
```

For `write`, `RAX` has to hold `__NR_write`, which is `1` on x86-64:

```text
RAX = 1
RDI = 1
RSI = address of "hello"
RDX = 5
```

With that set up, glibc executes one instruction:

```asm
syscall
```

This is not glibc calling a kernel function. There's no such thing, from user mode, as calling into the kernel directly, and that's deliberate: if user code could jump to any address it wanted and land in kernel mode, every syscall would also be an invitation to skip whatever checks the kernel meant to run first. `syscall` is a CPU instruction, and on x86-64 Linux configures the processor, through a model-specific register called `IA32_LSTAR`, so that executing it jumps to exactly one fixed kernel entry point and switches the CPU into kernel mode at the same time. One door, always the same door:

```text
glibc write()
     │ sets RAX / RDI / RSI / RDX
     ▼
syscall
     │ CPU privilege transition
     ▼
entry_SYSCALL_64
```

That entry point lives in `arch/x86/entry/entry_64.S`. The real file has years of Meltdown and Spectre mitigations layered on top, but the core of it looks roughly like this:

```asm
SYM_CODE_START(entry_SYSCALL_64)

    swapgs

    movq %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)

    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp

    movq PER_CPU_VAR(cpu_current_top_of_stack), %rsp

    /* ... construct struct pt_regs on the stack ... */

    call do_syscall_64
```

`swapgs` swaps in the kernel's per-CPU data segment. The important line for our purposes is the one that reloads `%rsp`: the `syscall` instruction itself does not switch stacks. It changes privilege level and jumps to `entry_SYSCALL_64`, but it leaves `RSP` pointing at whatever your user stack was using. The first thing the kernel does, by hand, in assembly, is save that user `RSP` somewhere and load the kernel stack for this task instead. Only after that does it start building `struct pt_regs`, the saved copy of your registers, and call into `do_syscall_64()`.

## Two stacks, one task

This is why your thread effectively has two stacks the moment it's inside a syscall. Not two threads, the same task, just executing in two different modes with two different `RSP` values at the same time.

<p class="diagram">
  <img class="diagram-light" src="/images/posts/how-write-reaches-the-linux-kernel/02-two-stacks-light.svg" alt="The same task shown with a user stack holding main, foo, and write frames, and a kernel stack holding entry_SYSCALL_64, do_syscall_64, and the write syscall handler">
  <img class="diagram-dark" src="/images/posts/how-write-reaches-the-linux-kernel/02-two-stacks-dark.svg" alt="The same task shown with a user stack holding main, foo, and write frames, and a kernel stack holding entry_SYSCALL_64, do_syscall_64, and the write syscall handler">
</p>

The kernel stack isn't conjured up for this one syscall. It's part of the task's kernel-side resources already, sitting there for the whole life of the task. `struct task_struct`, in `include/linux/sched.h`, has a field pointing straight at it, among its many other fields:

```c
struct task_struct {
    ...
    void *stack;
    ...
};
```

And `include/linux/sched/task_stack.h` treats it as exactly that, the task's kernel stack, in what's close to the real implementation:

```c
static __always_inline void *task_stack_page(
        const struct task_struct *task)
{
    return task->stack;
}
```

<p class="diagram">
  <img class="diagram-light" src="/images/posts/how-write-reaches-the-linux-kernel/03-task-stack-light.svg" alt="task_struct with fields pid, state, mm, files, and stack, where stack points to a separate kernel stack region holding saved state and call frames">
  <img class="diagram-dark" src="/images/posts/how-write-reaches-the-linux-kernel/03-task-stack-dark.svg" alt="task_struct with fields pid, state, mm, files, and stack, where stack points to a separate kernel stack region holding saved state and call frames">
</p>

`task_struct` and the kernel stack are two separate allocations in kernel memory, one pointing at the other. And worth saying out loud since it's easy to assume otherwise: the kernel doesn't keep a private copy of its own code per process. `entry_SYSCALL_64`, `do_syscall_64`, the VFS, the scheduler, it's all one shared body of kernel `.text` that every task on the machine runs through, each one just bringing its own `task_struct` and its own kernel stack along for the ride.

## Finding fd 1

Back inside `do_syscall_64()`, the kernel dispatches to the actual `write` syscall implementation, which eventually has to figure out what file descriptor `1` even means for this particular process. That's also resolved through the current task:

<p class="diagram">
  <img class="diagram-light" src="/images/posts/how-write-reaches-the-linux-kernel/04-fd-table-light.svg" alt="current points to task_struct, which through its files field points to files_struct, which holds the fd table containing fd 0, fd 1, and fd 2, where fd 1 points to a struct file representing stdout">
  <img class="diagram-dark" src="/images/posts/how-write-reaches-the-linux-kernel/04-fd-table-dark.svg" alt="current points to task_struct, which through its files field points to files_struct, which holds the fd table containing fd 0, fd 1, and fd 2, where fd 1 points to a struct file representing stdout">
</p>

`current` is how kernel code refers to whichever task is running on this CPU right now. Following it to its `files_struct` and indexing the fd table at `1` is exactly how the kernel knows your `1` means this process's stdout, and not some other process's open socket. It's also why `2>&1` in a shell works the way it does: redirection just swaps which `struct file` sits at a given slot in this same table, nothing upstream of it has to know or care. The exact chain of function names between the generic syscall dispatch and this lookup has shifted across kernel versions, the shape here, current task to fd table to the underlying file, is the part that's stayed stable.

Once the write actually happens, the kernel sets `RAX` to the return value, unwinds back up through `do_syscall_64()` and `entry_SYSCALL_64`, restores your saved user registers including the original `RSP`, and executes an instruction that drops the CPU back to user mode at the instruction right after `syscall`. Your program resumes with no idea any of this happened, just a return value sitting in what C sees as `write()`'s result.

## Watching it happen

You can see half of this from userspace. `cat /proc/<pid>/maps` will show the user stack mapping as `[stack]`. `sudo cat /proc/<pid>/stack` shows something different: a kernel stack *trace* for that task, the call chain it's currently sitting in, not the raw memory.

`write()` on 5 bytes returns too fast to catch mid-flight. A blocking syscall won't:

```c
#include <unistd.h>
#include <stdio.h>

int main(void)
{
    int fd[2];
    pipe(fd);

    printf("PID = %d\n", getpid());
    fflush(stdout);

    char c;
    read(fd[0], &c, 1);
}
```

The pipe is empty and still has a writer end open, so `read()` blocks. Run it, grab the PID, then:

```bash
sudo cat /proc/<pid>/stack
```

You'll see kernel functions from the syscall and wait path, exact names depending on your kernel version, proof that this specific task is parked inside the kernel, on its own kernel stack, right now.

<p class="diagram">
  <img class="diagram-light" src="/images/posts/how-write-reaches-the-linux-kernel/05-pipeline-light.svg" alt="The full path from the C source through the compiler, glibc, the syscall instruction, entry_SYSCALL_64, the task's kernel stack, do_syscall_64, the write implementation, VFS, and back to the C program">
  <img class="diagram-dark" src="/images/posts/how-write-reaches-the-linux-kernel/05-pipeline-dark.svg" alt="The full path from the C source through the compiler, glibc, the syscall instruction, entry_SYSCALL_64, the task's kernel stack, do_syscall_64, the write implementation, VFS, and back to the C program">
</p>
