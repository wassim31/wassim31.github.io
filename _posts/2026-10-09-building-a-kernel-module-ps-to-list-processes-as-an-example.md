---
layout: post
title: "Building a kernel module : ps to list processes as an example"
date: 2026-10-09 10:00:00 +0000
description: "Building pss.ko to walk Linux task_struct objects, print process IDs and names, and test the result through the kernel log."
tags: [c, linux, systemsprogramming, kernel]
---

## What we're building

Start with something familiar:

```bash
ps -eo pid,comm
```

Linux keeps process information in kernel memory. We'll list those processes by walking their descriptors directly. The normal userspace `ps` reads `/proc`; our `pss` module executes inside the kernel and writes to its log. Here, “processes” includes sleeping processes, not just tasks currently executing on a CPU.

## Creating our first kernel module

A loadable kernel module, a `.ko` file, adds code to the running kernel without rebuilding the whole operating system. Create a working directory:

```bash
mkdir -p ~/kernel-lab/k_module
cd ~/kernel-lab/k_module
```

Create `pss.c` with this skeleton:

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>

MODULE_AUTHOR("wassim");
MODULE_DESCRIPTION("first module");
MODULE_LICENSE("GPL");

static int __init ps_init(void)
{
    pr_info("pss: loaded\n");
    return 0;
}

static void __exit ps_exit(void)
{
    pr_info("good bye from wassim\n");
}

module_init(ps_init);
module_exit(ps_exit);
```

`module_init()` registers the entry function; `insmod` invokes it once during insertion. Returning zero signals success. `module_exit()` registers the removal callback. `__init` lets the kernel discard initialization code afterward; `__exit` marks cleanup code. `pr_info()` writes informational kernel messages. `MODULE_LICENSE("GPL")` declares licensing and permits access to GPL-only exports.

## Exploring real Linux kernel data structures

Inside your lab's full kernel source tree, run:

```bash
cd ~/kernel-lab/linux
rg -n 'struct task_struct \{' include/linux/sched.h
rg -n 'for_each_process|next_task' include/linux/sched/signal.h
rg -n 'struct task_struct init_task' init/init_task.c
```

These searches require full source; packaged build headers may omit `init/init_task.c`. Inspect these members of [`task_struct`](https://github.com/torvalds/linux/blob/v6.12/include/linux/sched.h):

```c
pid_t pid;
char comm[TASK_COMM_LEN];
struct list_head tasks;
```

`pid` stores the task ID. `comm` stores a short task name, rather than the complete command line. `tasks` embeds the links connecting process leaders in a circular doubly linked list.

The struct describes an object's layout. A `struct task_struct *` holds an existing object's address. An iteration macro advances that pointer through linked objects.

We don't allocate process descriptors ourselves. They already belong to the kernel; our module borrows pointers while reading them. Kernel threads also appear in this list, and unreaped zombies can still have descriptors.

[`for_each_process(task)`](https://github.com/torvalds/linux/blob/v6.12/include/linux/sched/signal.h) starts at `init_task`, advances before entering the loop, and stops when it returns there. It visits thread-group leaders, not every individual thread, and skips `init_task` itself.

## Implementing our simplified ps

Replace `pss.c` with this complete version, keeping the original module's function names and cleanup message:

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/sched/signal.h>
#include <linux/rcupdate.h>

MODULE_AUTHOR("wassim");
MODULE_DESCRIPTION("List process IDs and names");
MODULE_LICENSE("GPL");

static int __init ps_init(void)
{
    struct task_struct *task;
    char comm[TASK_COMM_LEN];

    rcu_read_lock();
    for_each_process(task) {
        get_task_comm(comm, task);
        pr_info("pss: PID=%d NAME=%s\n",
                task_pid_nr(task), comm);
    }
    rcu_read_unlock();

    return 0;
}

static void __exit ps_exit(void)
{
    pr_info("good bye from wassim\n");
}

module_init(ps_init);
module_exit(ps_exit);
```

The original loop accessed `task->pid` and `task->comm` directly. Here, `task_pid_nr()` returns the ID in the initial PID namespace. `get_task_comm()` copies the name under the task lock, handling concurrent renaming.

`struct list_head` contains next and previous pointers. `next_task()` follows `tasks.next` using an RCU-aware list helper and recovers the enclosing descriptor. That's why `for_each_process()` needs a pointer variable: it assigns object addresses, not numeric PIDs.

The [RCU read-side section](https://docs.kernel.org/RCU/listRCU.html) prevents traversed descriptors from being freed during our walk. It doesn't freeze the process list into an atomic snapshot. Don't retain `task` after unlocking or add sleeping operations inside this section. Each iteration copies a name, reads an ID, and logs one line.

## Compiling the module

On Ubuntu or Debian, install the build tools and matching headers:

```bash
sudo apt install build-essential "linux-headers-$(uname -r)"
cd ~/kernel-lab/k_module
```

Create `Makefile`, using actual tabs before recipe commands:

```makefile
obj-m += pss.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

Run:

```bash
make
file pss.ko
```

[`Kbuild`](https://docs.kernel.org/kbuild/modules.html) compiles `pss.c` and links `pss.ko`. Headers and build configuration must match the target kernel. The original lab Makefile uses `~/kernel-lab/linux`; use `make KDIR="$HOME/kernel-lab/linux"` only with a prepared, built tree matching your target kernel.

## Loading and testing the module

```bash
sudo insmod pss.ko
sudo dmesg | tail -30
lsmod | grep pss
sudo rmmod pss
```

Insertion runs `ps_init()`, producing lines like these; the PIDs are illustrative:

```text
pss: PID=1 NAME=systemd
pss: PID=842 NAME=bash
pss: PID=1204 NAME=sshd
```

`tail` shows only the last messages. Inspect the full listing with:

```bash
sudo dmesg | grep 'pss:'
```

Removal runs `ps_exit()`. Reinsert to execute initialization again. Compare with `ps -eo pid,comm`; processes can change between runs.

The module stays loaded after printing; it doesn't continuously refresh the listing.

Test inside a QEMU Linux guest, building against its kernel. Missing headers stop compilation; version mismatch can produce “Invalid module format.” Check `dmesg`. Unloading needs kernel support and no outstanding module references. Secure Boot may reject unsigned modules.

## What we learned

We accessed `task_struct`, followed embedded linked lists through kernel macros, and executed C in kernel space. Next, inspect parent-child relationships through `real_parent`, `children`, and `sibling`, checking their locking requirements first.
