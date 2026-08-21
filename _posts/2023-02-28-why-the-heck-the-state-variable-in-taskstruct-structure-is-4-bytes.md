---
layout: post
title: "Why the heck the _state variable in task_struct structure is 4 bytes"
date: 2023-02-28 01:34:32 +0000
description: "The Linux process state needs six flags, so why is it stored in an unsigned integer with 2^32 possible values? The answer is in READ_ONCE/WRITE_ONCE."
canonical_devto: https://dev.to/wassim31/why-the-heck-the-state-variable-in-taskstruct-structure-is-4-bytes-3nkh
image: /images/posts/why-the-heck-the-state-variable-in-taskstruct-structure-is-4-bytes/cover.jpeg
---

According to Robert Love, the author of the book Linux Kernel Development, and the Linux kernel source:

In the Linux kernel, specifically in the PCB (process control block), there is a structure that represents the process in the kernel: `task_struct`.

## The six state flags

The process state is represented by a combination of six state flags:

- **TASK_RUNNING**: the process is currently running or ready to run.
- **TASK_INTERRUPTIBLE**: the process is waiting for a specific event to occur and can be interrupted by a signal.
- **TASK_UNINTERRUPTIBLE**: the process is waiting for a specific event to occur and cannot be interrupted by a signal.
- **__TASK_STOPPED**: the process has been stopped (e.g. by a SIGSTOP signal) and can be resumed later.
- **__TASK_TRACED**: the process is being traced by another process (e.g. a debugger).
- **TASK_DEAD**: the process has terminated and is waiting to be reaped by its parent process.

## Combining flags

Each of these flags can be set or cleared to represent the different states a process can be in. For example, a process that is both **TASK_INTERRUPTIBLE** and **TASK_UNINTERRUPTIBLE** might be waiting for a disk I/O operation to complete, and can be interrupted by a signal but cannot be killed until the I/O operation is finished.

In other words, a process can have multiple states at once, depending on which combination of flags is set.

## So why 4 bytes?

This state is saved in an **unsigned integer**. That means 32 bits, which means 2³² possible values. But we only need 2⁶, so 64 possible states as a maximum.

![The _state field in task_struct](/images/posts/why-the-heck-the-state-variable-in-taskstruct-structure-is-4-bytes/img1.png)

The reason is the following:

There are some predefined functions that manage the process's state, such as **READ_ONCE/WRITE_ONCE**, that oblige this variable to be an unsigned int. It used to be a volatile long. Here is the commit:

<https://github.com/torvalds/linux/commit/2f064a59a11ff9bc22e52e9678bc601404c7cb34#diff-f8d8a1568ae83bbff6f40f9c70559a4f7dbf426a397131ba9d4fbfb947ea5222R669>

Thank you for reading <3
