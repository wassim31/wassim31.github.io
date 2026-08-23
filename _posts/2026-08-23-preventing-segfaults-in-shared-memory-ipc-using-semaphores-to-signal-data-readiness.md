---
layout: post
title: "Preventing Segfaults in Shared Memory IPC: Using Semaphores to Signal Data Readiness"
date: 2026-08-23 10:00:00 +0000
description: "mmap() succeeding doesn't mean the data behind it is valid. A close look at the race where a reader touches shared memory before the writer is done, and how a semaphore closes it."
tags: [c, linux, ipc, systemsprogramming]
---

Here's a bug that'll make you want to throw your laptop across the room: the reader process calls `shm_open()`, it succeeds. It calls `mmap()`, it also succeeds. Every syscall you bothered to check is green. And then, sometimes, not always, the process segfaults anyway, or prints garbage where the writer's message was supposed to be.

If this is happening to you right now, your first move is probably to go re-check the shared memory setup. Right name, right size, right permissions. That's not where the bug is. It's not that the reader wasn't allowed to look at the memory. It's that it looked too early.

(There's a different bug that looks similar from far away: the reader runs before the writer has even created the segment, so `shm_open()` itself returns `-1`. That one's boring, you fix it by checking your return values and refusing to continue on a bad fd. It's not what's happening here, and I'm not going to spend more time on it, because the interesting case, and the one that actually costs people hours, is the one where every single syscall succeeds and the crash happens anyway.)

## What's actually sitting in that memory

Here's the setup. The writer creates the shared memory object and sizes it:

```c
int fd = shm_open("/shared_memory", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

`ftruncate()` does two things here: it sets the segment to 4096 bytes, and since this is a freshly created object, it zero-fills those bytes on the way there. That's not a courtesy, it's the kernel refusing to hand your process leftover contents of some physical page it doesn't know the history of. Fine. But "zero-filled" is not "contains the message the writer is about to send." It's zeros. Placeholder bytes that happen to be sitting there because something had to be.

And that's the *good* case. If the segment already existed from an earlier run of your program, and you didn't clean it up, `ftruncate()` to the same size it already has does nothing at all. No zeroing, no reset. Whatever was in there from last time is still in there, and it is very much not guaranteed to look like zeros.

Say the writer and reader agree on this layout:

```c
typedef struct payload {
    size_t length;
    char buffer[];
} payload;
```

The writer's job is eventually to set `length` and copy a string into `buffer`. The reader's job is to read `length`, then copy that many bytes back out:

```c
size_t length = *(size_t *)addr;
char string[length];
memcpy(string, (char *)addr + sizeof(size_t), length);
```

Now picture the two processes starting at roughly the same time, which is the normal case, not some pathological edge case you have to work hard to trigger:

```
writer                              reader
------                              ------
shm_open(O_CREAT)
ftruncate()
mmap()
                                     shm_open()   <- segment exists, succeeds
                                     mmap()       <- mapping is valid, succeeds
                                     length = *(size_t *)addr   <- reads zeros or stale bytes
data->length = length
memcpy(data->buffer, ...)
```

Nothing in that reader column failed. There's no line in there you could wrap in an `if` and catch. The reader just happened to read `length` a few instructions before the writer got around to setting it, and there was nothing stopping it from doing so.

So what does the reader actually get? If the segment was freshly zeroed, `length` comes out as `0`. `char string[length]` is a zero-length VLA, `memcpy()` copies nothing, and you get an empty string. Weird output, but the process survives.

If the segment had stale bytes from a previous run instead, `length` can be anything a `size_t` holds. Say it comes out enormous. `char string[length]` isn't a heap allocation that can fail gracefully and hand you `NULL`. It's a stack pointer bump, done on the spot, at the point of declaration. If that number is big enough, you blow through the rest of the stack before you've written a single byte of the string, and the process dies right there, on a line that doesn't even mention `memcpy()`.

And if the garbage length is large but not stack-shattering large, `memcpy()` goes ahead and reads that many bytes starting at `addr + sizeof(size_t)`. Run past the end of the 4096-byte mapping and you get a segfault from touching unmapped memory. Or you don't run quite that far, and instead you silently read whatever else happens to be mapped nearby, and hand your caller a string that isn't garbage-looking at all, it's just wrong. That second version is worse, because nothing crashes and nothing looks broken.

The thing worth sitting with here is that `shm_open()` returned a valid fd and `mmap()` returned a valid pointer both times, in both the empty-string case and the stack-overflow case. The syscalls did their job. The bug is entirely downstream of them, in the assumption that a successful mapping means there's something meaningful behind it.

## sleep() is not a fix

Once you've figured out it's a timing problem, the first instinct is almost always some version of "just wait a little before reading." None of the usual ways of doing that actually close the race.

The most common one is a retry loop with `sleep()` in it: poll `length`, and if it still looks unset, sleep and check again. This appears to work, most of the time, because your writer probably finishes in a few milliseconds and your sleep is probably a full second, so in practice you never catch it in the act. That's not the same thing as it being fixed. It's a race where you've made the reader's side slower, which changes the odds without touching the actual problem. Put the machine under load, or add anything at all to the writer's startup, like reading a config file or allocating a buffer, and your sleep duration stops being generous enough. You're back to the original bug, just less often, which honestly might be worse, because now it's the kind of bug that only shows up in production.

A slightly smarter-sounding version checks whether the memory is still all zeros before reading it. This has the same timing problem as the sleep loop, plus a correctness problem of its own: zero is a perfectly legitimate value. If the writer's real payload happens to produce a `length` of `0`, or the first eight bytes of a valid message happen to be zero, your "is it ready yet" check can't tell that apart from "not ready yet." And as covered above, if the segment wasn't freshly zeroed, "non-zero" doesn't mean "written by this run" either. Leftover bytes from an old run can easily be non-zero, so this check can happily report "looks ready" on a segment the current writer hasn't touched at all.

And then there's just assuming the writer runs first because you start it first. There's no guarantee anywhere that backs this up. Process creation order and scheduler decisions aren't something POSIX promises you control over, and even setting that aside, "started first" isn't "finished writing." If the writer does anything at all before it gets to the actual write, that's a window where a reader that technically started later can still get there first.

All three of these are trying to guess readiness from either the contents of the memory or the passage of time. What you actually want is for the writer to tell you, explicitly, "I'm done," and for the reader to be physically unable to proceed until it hears that. That's what a semaphore gives you, and it's the only one of these that isn't a guess.

## A semaphore that only goes one way

If you've used semaphores before, it was probably as a mutex: a binary semaphore that starts at 1, where a thread calls `sem_wait()` to grab it, does some work, and calls `sem_post()` to hand it back. Both sides call both functions. It's symmetric, and it's protecting a critical section.

That's not what we're building here, and thinking of this as "a lock" will make the rest of it confusing. What we want is a readiness signal, and it differs in two specific ways:

- It starts at 0, not 1. Zero means "nothing to report yet." A lock starts at 1 because the resource is free from the beginning; a readiness signal starts at 0 because the data isn't ready from the beginning, and shouldn't be treated as ready until someone says otherwise.
- The two sides don't do the same thing. The writer only ever calls `sem_post()`. The reader only ever calls `sem_wait()`. Nobody waits and posts around the same operation the way a mutex would have both sides do. It only goes one direction: the writer speaks once, the reader listens once.

The mechanics: `sem_wait()` decrements the semaphore's counter. If the counter's already at 0, it doesn't go negative, it just blocks, parking the calling process until something else bumps the count back up. This is a genuine suspension, handled by the kernel, not a spin loop and not a `sleep()` in disguise. `sem_post()` increments the counter, and if there's a process sitting in `sem_wait()`, it wakes exactly one of them.

Put that back into our reader and writer. The semaphore starts at 0. The reader hits `sem_wait()` before it has touched a single byte of the mapped memory, sees a count of 0, and blocks right there. It cannot execute the next line. It's not choosing to wait, it's stuck. Meanwhile the writer sets `length`, copies the string into `buffer`, and only after both of those are done does it call `sem_post()`. That's what bumps the count and wakes the reader up. The reader's `sem_wait()` returns, and only now, after the writer is provably finished, does the reader go read `length` and `buffer`.

Go back to the diagram from earlier. There's no version of it anymore where the reader's read of `length` can land before the writer's write of `length`, because the line that does the reading is now behind a wall the writer built, and the writer only takes that wall down once it's actually done.

## Here's the fix, in full

Complete writer and reader, error-checked this time. The writer owns creating both the shared memory object and the semaphore; the reader just opens what's already there.

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

The writer passes `O_CREAT` to both `shm_open()` and `sem_open()`; the reader passes neither, so if it happens to run before the writer has created either one, it fails immediately with a plain `ENOENT` instead of quietly creating something itself. The `shm_open()`/`mmap()` checks are what catch that case specifically; `sem_wait()` is what catches the one this whole article is about, where the segment exists but isn't populated yet. The two don't overlap and neither one covers for the other.

One more thing about the reader: it copies the payload into a heap buffer sized with `malloc()`, not a stack VLA like the earlier broken version. `length` here is coming from a cooperating process on the same machine rather than something adversarial, so this isn't quite the same threat model as parsing untrusted input, but I still wouldn't size a stack array off a number I read out of shared memory without thinking about it first. If you want the longer version of that argument, I wrote a whole piece on it: [Safe Length-Based Data Sharing in C](/2026/08/22/safe-length-based-data-sharing-in-c.html).

## The pitfalls that get you anyway

POSIX named semaphores don't go away when your process exits. They live in the kernel, backed by a file under `/dev/shm` on Linux, until something explicitly calls `sem_unlink()`, or the machine reboots. This bites people in a specific and annoying way: it can make the exact race this article is about disappear from your testing while still being sitting in your code.

Say you run the writer and reader once, cleanly. Writer posts, reader waits and consumes it, semaphore ends the run at 0. Now say you run the writer *twice* in a row, before ever running the reader, maybe because you're testing something else entirely. Each run calls `sem_post()`. Since the semaphore already existed after the first run, the second run's `sem_open(SEM_NAME, O_CREAT, ...)` just hands back the existing one, unchanged, because `O_CREAT` on an object that already exists ignores the initial-value argument you gave it. So after two writer runs, the count sits at 2.

Now the reader runs. Its `sem_wait()` sees a non-zero count and returns immediately, without ever actually blocking on anything. If you happen to be mid-refactor and there's a bug that makes the *next* writer run crash before it writes anything, or skip the write entirely, you will not see it. The leftover count from an earlier run covers for it completely, and the reader sails through `sem_wait()` and reads whatever's sitting in the segment from before. It's the exact race from the top of this article, quietly back, except this time it's hiding behind a semaphore that looks, from the outside, like it's doing its job.

The fix is `sem_unlink(SEM_NAME)` and `shm_unlink(SHM_NAME)` between runs, so each run starts from a genuinely fresh semaphore at the value you meant, and a genuinely fresh, zeroed segment, instead of whatever the last run happened to leave behind.

The other thing worth deciding on purpose, not by accident: who owns `O_CREAT`. In the example above, the writer creates both the shared memory object and the semaphore, and the reader only ever opens what's already there. That's not arbitrary. If both sides pass `O_CREAT`, there's no longer a clean answer to "who set the initial value," because those creation arguments only take effect for whichever process's open call happens to run first, and if both processes start around the same moment, that's a race of its own. Pick one side to own creation, have it create both objects before doing anything else, and have the other side open them without `O_CREAT`, so a missing object fails loudly instead of getting created ambiguously by whichever process got there first.

## So, back to that segfault

If you're staring at a reader that segfaults sometimes, or hands back garbage sometimes, and every syscall you're checking is returning success, it's very likely this. Not a missing segment, not a bad permission bit, not a corrupted mapping. Just a reader that got to the memory before the writer was done with it, because nothing was stopping it from trying.

A semaphore initialized to 0, posted once by the writer after it's actually finished, waited on once by the reader before it touches anything, closes that gap completely. Not by making the race less likely. By making it structurally impossible for the reader's read to land before the writer's write.
