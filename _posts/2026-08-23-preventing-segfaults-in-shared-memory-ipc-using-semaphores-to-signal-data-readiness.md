---
layout: post
title: "Preventing Segfaults in Shared Memory IPC: Using Semaphores to Signal Data Readiness"
date: 2026-08-23 10:00:00 +0000
description: "mmap() succeeding doesn't mean the data behind it is valid. Why reading shared memory before the writer is done causes intermittent segfaults, and how a semaphore closes that race."
tags: [c, linux, ipc, systemsprogramming]
---

Here's a bug that's genuinely annoying to chase down: your reader process calls `shm_open()`, it succeeds. It calls `mmap()`, it succeeds. Every syscall you're checking tells you everything is fine. And then, sometimes, not always, the process segfaults anyway, or prints garbage instead of the message the writer sent.

If you've hit this, your instinct is probably to double-check the shared memory setup. The segment exists, the size is right, the permissions are right. That's not where the bug is. The bug is in *when* the reader looked at the memory, not *whether* it was allowed to.

## Two different problems that look identical from the outside

Before going further, it's worth separating two failure modes that get conflated constantly, because the fix for one does nothing for the other.

**Problem one: the segment doesn't exist yet.** The reader runs before the writer has created it. `shm_open()` returns `-1`, or a subsequent `mmap()` on a bad file descriptor fails. This is a lifecycle problem, and you fix it the boring way: check the return value of every syscall, and fail loudly (or retry the *open*, deliberately) instead of plowing ahead.

**Problem two: the segment exists, but the data inside it isn't ready yet.** This is the one this article is about. `shm_open()` succeeds. `mmap()` succeeds. The memory is right there, mapped, readable. It's just that the writer hasn't finished writing to it, or hasn't written to it at all, and the reader has no way to know that from the syscalls it just called successfully.

Semaphores solve problem two. Error-checking solves problem one. Neither solves the other, and if you only implement one of them, you'll fix one class of crash and be baffled by the other still happening intermittently.

## Why a successful mmap doesn't mean the data is valid

Here's the setup. The writer creates the shared memory object and sizes it:

```c
int fd = shm_open("/shared_memory", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

`ftruncate()` here does two things: it sets the segment's size to 4096 bytes, and on a freshly created segment, it zero-fills those bytes. That's the key detail. Zero-filled is not the same thing as "contains the data I'm about to send you." It's zeros. Meaningless zeros, that happen to be sitting there because the kernel had to put something in newly allocated pages.

And that's the *best* case. If the segment already existed from a previous run of your program (more on this later), those bytes aren't even guaranteed to be zero. They're whatever was left over from last time.

Now say the writer and reader agree on this layout:

```c
typedef struct payload {
    size_t length;
    char buffer[];
} payload;
```

The writer's job, eventually, is to set `length` and copy a string into `buffer`. The reader's job is to read `length`, then copy that many bytes out:

```c
size_t length = *(size_t *)addr;
char string[length];
memcpy(string, (char *)addr + sizeof(size_t), length);
```

Walk through what happens if the reader gets there first, before the writer has written anything.

`length` gets read from memory that's either all zeros or full of stale garbage from a previous run. If it's zero, `char string[length]` is a zero-length VLA, and the `memcpy()` copies zero bytes; you get an empty string. Confusing, but survivable.

If it's stale garbage, though, `length` could be anything a `size_t` can hold. Say it happens to be some enormous number. `char string[length]` tries to reserve that many bytes on the stack, on the spot, as a variable-length array. That's not a heap allocation that fails gracefully and returns `NULL`. It's a stack pointer bump. If the number is big enough, you blow past the end of the stack before you've written a single byte, and the kernel kills the process with a segfault right there, on the *declaration*, before `memcpy()` is even called.

And if the size happens to be merely large-but-not-catastrophic, `memcpy()` then reads that many bytes starting from `addr + sizeof(size_t)`. If that runs past the end of the 4096-byte mapping, you're reading unmapped memory, which is its own segfault, or you're reading whatever else happens to be mapped nearby, which is a quieter, worse bug: it doesn't crash, it just hands you garbage that looks like data.

None of this shows up as a failed syscall. `shm_open()` returned a valid fd. `mmap()` returned a valid pointer. Everything you'd normally check was fine. The crash comes from what the reader did *after* those calls succeeded, because it assumed success meant "the data is ready," when all it actually meant was "the memory is mapped."

## Why the obvious fixes don't work

Once you've diagnosed this as a timing problem, the first instinct is usually some flavor of "just wait a bit." None of the common versions of that actually work.

**Retry loops with `sleep()`.** Poll `length`, and if it looks unset, sleep for a bit and check again. This "works" on your machine, most of the time, because your writer probably finishes in a few milliseconds and your sleep is probably a full second. It's not a fix, it's a race with worse odds. Under load, or on a slower machine, or if the writer does anything before writing (allocates memory, reads a config file, anything), your sleep duration is just a guess about how long that takes. Guess wrong once and you're back to the original bug, just less often.

**Checking "is it still all zeros."** Aside from being another poll loop with the same timing problem, this has a correctness issue on top: zero is a legitimate value. If the writer's real message happens to produce a `length` of zero, or the first 8 bytes of a valid payload happen to be zero, your "is it ready" check is indistinguishable from "isn't ready yet." And per the stale-data point above, "non-zero" doesn't mean "written by this run" either. A previous run's leftover data can easily be non-zero, which means your check can report "ready" for a segment the current writer hasn't touched at all.

**Assuming the OS schedules the writer first.** There's no such guarantee, and there's no reason to expect one. Process creation order, `fork()`/`exec()` timing, and scheduler decisions are not something POSIX promises you control over. Even if the writer process technically starts first, "started" isn't "finished writing." If the writer does any work at all before it gets to writing the payload, that's a window where a reader that started a moment later can still get there first.

What all three of these share is that they're trying to infer readiness from the *content* of memory, or from timing, when what you actually need is an explicit, unambiguous signal from the writer that says "I am done." That's precisely what a semaphore gives you.

## Semaphore as a one-directional signal

If you've used semaphores before, it was probably as a mutex: a binary semaphore initialized to 1, where a thread calls `sem_wait()` to acquire it, does some work, and calls `sem_post()` to release it. Both sides call both functions. It's symmetric, and it protects a critical section.

That's not the pattern here, and mentally filing this under "semaphore = lock" will make the rest of this confusing. What we want is a **readiness signal**, and it looks different in two specific ways:

- **The initial value is 0, not 1.** Zero means "nothing to report yet." A lock starts at 1 because the resource is initially available; a readiness signal starts at 0 because the data is initially *not* ready.
- **The two sides do different things.** The writer only ever calls `sem_post()`. The reader only ever calls `sem_wait()`. Nobody waits and posts around the same operation the way you would with a mutex. It's one-directional: writer speaks, reader listens.

The mechanics that make this work are worth being precise about:

`sem_wait()` decrements the semaphore's internal counter. If the counter is already at 0, it doesn't decrement past that; it blocks, parking the calling thread until someone else increments the count. It genuinely suspends the process. There's no spinning, no polling, no `sleep()` involved.

`sem_post()` increments the counter, and if there's a thread blocked in `sem_wait()`, it wakes exactly one of them.

Trace through what this means for our reader and writer. The semaphore starts at 0. The reader calls `sem_wait()` before touching a single byte of the mapped memory. Since the count is 0, the reader blocks, immediately, at that line. It physically cannot execute the next line of code. Meanwhile the writer writes `length`, writes `buffer`, and only once both are written does it call `sem_post()`. That increments the count from 0 to 1, and wakes the reader. The reader's `sem_wait()` call returns, and only now does it proceed to read `length` and `buffer`.

There's no window in this sequence where the reader can observe a partially-written or not-yet-written payload, because the reader's code literally cannot reach the code that reads the payload until the writer has already called `sem_post()`, which the writer only does after finishing the write. That's the entire race from the earlier section, closed by construction.

## The full worked example

Here's a complete writer and reader, with error-checking on every syscall this time. The writer owns creation of both the shared memory object and the semaphore; the reader just opens what's already there.

**writer.c**

```c
#include <fcntl.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>

#define SHM_NAME "/shared_memory"
#define SEM_NAME "/semaphore"
#define SHM_SIZE 4096

typedef struct payload {
    size_t length;
    char buffer[];
} payload;

int main(void) {
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (fd == -1) {
        perror("shm_open");
        exit(EXIT_FAILURE);
    }

    if (ftruncate(fd, SHM_SIZE) == -1) {
        perror("ftruncate");
        exit(EXIT_FAILURE);
    }

    void *addr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }

    /* Initial value 0: nothing is ready until we say so. */
    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0666, 0);
    if (sem == SEM_FAILED) {
        perror("sem_open");
        exit(EXIT_FAILURE);
    }

    const char *message = "Hello world from Process A";
    size_t length = strlen(message) + 1;

    payload *data = (payload *)addr;
    data->length = length;
    memcpy(data->buffer, message, length);

    /* Only signal readiness after the payload is fully written. */
    if (sem_post(sem) == -1) {
        perror("sem_post");
        exit(EXIT_FAILURE);
    }

    printf("writer: wrote %zu bytes and signaled the reader\n", length);

    munmap(addr, SHM_SIZE);
    close(fd);
    sem_close(sem);
    return 0;
}
```

**reader.c**

```c
#include <fcntl.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>

#define SHM_NAME "/shared_memory"
#define SEM_NAME "/semaphore"
#define SHM_SIZE 4096

typedef struct payload {
    size_t length;
    char buffer[];
} payload;

int main(void) {
    int fd = shm_open(SHM_NAME, O_RDWR, 0666);
    if (fd == -1) {
        perror("shm_open");
        exit(EXIT_FAILURE);
    }

    void *addr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }

    sem_t *sem = sem_open(SEM_NAME, 0);
    if (sem == SEM_FAILED) {
        perror("sem_open");
        exit(EXIT_FAILURE);
    }

    printf("reader: waiting for data...\n");

    /* Blocks here until the writer calls sem_post(). No memory is
       touched before this line returns. */
    if (sem_wait(sem) == -1) {
        perror("sem_wait");
        exit(EXIT_FAILURE);
    }

    payload *data = (payload *)addr;
    size_t length = data->length;

    char *string = malloc(length);
    if (!string) {
        perror("malloc");
        exit(EXIT_FAILURE);
    }
    memcpy(string, data->buffer, length);

    printf("reader: got %zu bytes: %s\n", length, string);

    free(string);
    munmap(addr, SHM_SIZE);
    close(fd);
    sem_close(sem);
    return 0;
}
```

A couple of details worth pointing out. The writer calls `shm_open()` and `sem_open()` with `O_CREAT`; the reader calls both without it, so if it runs before the writer has created either one, it fails immediately with a clear `ENOENT` from `perror()` instead of silently creating something. And the reader copies the payload into a heap buffer sized with `malloc()` rather than a stack VLA. `length` here comes from a cooperating process on the same machine, not from an untrusted or adversarial source, so this isn't the same threat model as parsing attacker-controlled input, but it's still worth not trusting a size read out of shared memory for a stack allocation by default. If you want the deeper argument for why, and how to validate a length before using it, I wrote about that specifically in [Safe Length-Based Data Sharing in C](/2026/08/22/safe-length-based-data-sharing-in-c.html).

> **Two problems, two fixes, in this exact example.** The `shm_open()`/`mmap()` error checks above handle the case where the segment doesn't exist at all: run the reader before the writer has ever run, and `shm_open()` returns `-1`, which we catch and exit on. The semaphore handles the separate case where the segment exists but isn't populated: run the writer and reader in either order, and the reader's `sem_wait()` guarantees it only reads the payload after the writer has finished writing it. Neither mechanism substitutes for the other.

## Pitfalls that will bite you anyway

**Stale semaphore state between runs.** POSIX named semaphores don't go away when your process exits. They live in the kernel (backed by a file under `/dev/shm` on Linux) until something explicitly calls `sem_unlink()`, or the system reboots. This matters a lot here, because it can make the exact bug this article is about disappear from your testing while still being present in your code.

Say you run the writer and reader once, cleanly. The writer posts, the reader waits and consumes it. The semaphore ends the run at 0. Fine. Now say you run the writer *twice* in a row, maybe testing something, before running the reader at all. Each run calls `sem_post()` once, and since the semaphore already existed from the first run, `sem_open(SEM_NAME, O_CREAT, ...)` on the second run just returns the existing one, unchanged, ignoring the initial-value argument entirely, because `O_CREAT` on an already-existing semaphore doesn't reset it. So after two writer runs, the count sits at 2.

Now the reader runs. Its `sem_wait()` sees a non-zero count and returns immediately, without ever actually blocking. If you're mid-refactor and there's a bug that causes the *next* writer run to not post at all, or to crash before writing the payload, you won't see it: the leftover count from an earlier run masks it completely, and the reader sails through `sem_wait()` and reads whatever garbage happens to be sitting in the segment. The exact race this whole article is about, quietly reintroduced, by a semaphore that looks like it's doing its job.

The fix is `sem_unlink(SEM_NAME)` and `shm_unlink(SHM_NAME)` between test runs, so each run starts from a genuinely fresh semaphore at your intended initial value and a genuinely fresh (zeroed) shared memory segment, not whatever state the last run left behind.

**Deciding who owns `O_CREAT`.** In the example above, the writer creates both the shared memory object and the semaphore; the reader only opens them. This is a deliberate choice, not an arbitrary one. If both sides pass `O_CREAT`, you no longer have a clear answer to "who decided the initial value," because `O_CREAT`'s mode and initial-value arguments are only honored by whichever process's open call actually creates the object first, and that's a race in itself if both processes start around the same time. Pick one process to own creation, have it create both objects before doing anything else, and have the other process open them without `O_CREAT` so a missing object fails loudly instead of getting silently, ambiguously created by the wrong side.

## Summary

| | Segment doesn't exist yet | Segment exists but data isn't ready |
|---|---|---|
| What you'll see | `shm_open()` returns `-1`, or `mmap()` fails on a bad fd | `shm_open()` and `mmap()` both succeed; the crash or garbage happens later |
| What's actually wrong | Lifecycle ordering: the reader ran before the writer created the object | Timing: the reader read the memory before the writer finished writing it |
| How you catch it | Checking the return value of `shm_open()` and `mmap()` | You can't catch it by checking `mmap()`'s return value; it succeeds either way |
| The fix | Error-check the open calls and fail fast, or coordinate creation order at a level above shared memory | A semaphore (or equivalent readiness signal) the reader waits on before touching the mapping |
| What it looks like unfixed | An immediate, consistent crash on a missing segment | An intermittent segfault or garbage read, present only when timing goes wrong |
