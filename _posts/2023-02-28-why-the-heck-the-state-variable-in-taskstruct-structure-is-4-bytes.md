---
layout: post
title: "Why the heck the _state variable in task_struct structure is 4 bytes"
date: 2023-02-28 01:34:32 +0000
description: "According to Robert Love the Author of the book Linux Kernel developement &amp; Linux kernel source..."
canonical_devto: https://dev.to/wassim31/why-the-heck-the-state-variable-in-taskstruct-structure-is-4-bytes-3nkh
image: /images/posts/why-the-heck-the-state-variable-in-taskstruct-structure-is-4-bytes/cover.jpeg
---

According to Robert Love the Author of the book Linux Kernel developement & Linux kernel source :

In the Linux kernel , specifically in the PCB ( process control block ) which a structure that represents the process in the kernel .

The process state is represented by a combination of six state flags, which are:

**TASK_RUNNING**: The process is currently running or ready to run.

**TASK_INTERRUPTIBLE:** The process is waiting for a specific event to occur and can be interrupted by a signal.

**TASK_UNINTERRUPTIBLE**: The process is waiting for a specific event to occur and cannot be interrupted by a signal.

**__TASK_STOPPED**: The process has been stopped (e.g. by a SIGSTOP signal) and can be resumed later.

**__TASK_TRACED**: The process is being traced by another process (e.g. a debugger).

**TASK_DEAD**: The process has terminated and is waiting to be reaped by its parent process.

Each of these flags can be set or cleared to represent the different states that a process can be in. For example, a process that is both **TASK_INTERRUPTIBLE** and **TASK_UNINTERRUPTIBLE** might be waiting for a disk I/O operation to complete, and can be interrupted by a signal but cannot be killed until the I/O operation is finished.

In other words, a process can have multiple states at once, depending on which combination of flags is set.

This state is saved in an **UNSIGNED INTEGER **, that means 32 bits , that means 2³² possible values , but we need only 2⁶ , so 64 possible states as a maximum .

![Image description](/images/posts/why-the-heck-the-state-variable-in-taskstruct-structure-is-4-bytes/img1.png)

the reason is the following : 

There are some predefined functions that manages the process's state such as **READ_ONCE/WRITE_ONCE** that obliges this variable to be an unsigned INT , it's used to be a volatile Long ; here are the commits : 

https://github.com/torvalds/linux/commit/2f064a59a11ff9bc22e52e9678bc601404c7cb34#diff-f8d8a1568ae83bbff6f40f9c70559a4f7dbf426a397131ba9d4fbfb947ea5222R669

Thank you for reading <3 .
