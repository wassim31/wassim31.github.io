---
layout: post
title: "Safe Length-Based Data Sharing in C"
date: 2026-08-22 10:00:00 +0000
description: "Don't ask where the '\\0' is. Ask how many bytes are valid. A progressive look at why strcpy() fails on shared data, and how length-delimited buffers fix it."
tags: [c, systemsprogramming, memory, security]
---

Let's say two processes share a region of memory. One of them writes some data into it, the other reads it. Simple enough, until the reading side has to answer a question that sounds trivial but isn't: *how much of this memory is actually mine to read?*

## The problem

Here's the kind of code that shows up when nobody has asked that question yet:

```c
char buffer[256];
strcpy(buffer, source);
```

`source` is a pointer into the shared region. It looks like a string. It might even be a string, most of the time. So `strcpy()` gets called, and it works, in testing, on your machine, with the inputs you happened to try.

The problem is that `strcpy()` doesn't know three things:

- how large the memory region behind `source` actually is
- how large `buffer`, the destination, actually is
- whether there is a null terminator anywhere inside the valid region at all

It doesn't ask. It just starts reading byte by byte from `source`, copying each one into `buffer`, until it hits a `'\0'`. That's the entire contract of `strcpy()`: keep going until you see a zero byte.

```
source:
+----------------------------------+
| H | e | l | l | o | ... | ???  |
+----------------------------------+
                              ^
                       where is '\0'?
```

If the writing process filled the shared region with exactly `"Hello"` and nothing else, and the region happens to end right after those five bytes, there is no `'\0'` inside the valid data. `strcpy()` doesn't know that "valid data" ended, so it keeps reading whatever comes next: uninitialized memory, another process's leftover bytes, or an unmapped page that segfaults on touch.

This isn't a strcpy bug. `strcpy()` is doing exactly what it was designed to do. The bug is upstream of it: we handed it a pointer and implicitly assumed a length that was never actually communicated.

## The key idea: stop asking where, start asking how much

The fix isn't a smarter string function. It's a different question. Instead of:

> "Where is the `'\0'`?"

the receiving side should be asking:

> "How many bytes are valid, starting from this pointer?"

That second question has an answer that doesn't depend on scanning memory and hoping. It's a number, and if the writer tells you that number up front, you never have to guess.

This means representing the data explicitly as a length followed by a payload, instead of a bare, unmarked stream of bytes:

```
+------------+----------------------+
| length     | payload              |
+------------+----------------------+
             <---- length bytes --->
```

This is usually called **length-delimited** or **length-prefixed** data, and it's a genuinely old idea, older than C strings themselves. The boundary of the data isn't a special byte value hidden somewhere in the payload. It's a number, sitting right next to the data it describes.

## Why this works better than null termination

Length-prefixing isn't just a workaround for the shared-memory case. It's a strictly more general representation, because it works for arbitrary binary data, not just text.

Think about what a null terminator actually assumes: that the byte value `0x00` never legitimately appears inside your data. That's a reasonable assumption for ASCII text. It's not a safe assumption for, say, a serialized struct, an image, an encrypted blob, or a network packet.

```
+----+----+----+----+----+
| 41 | 00 | FF | 42 | 00 |
+----+----+----+----+----+
```

The second byte here is `0x00`. Treat this as a C string, and anything reading it with `strlen()` or `strcpy()` would stop after just one byte, `0x41`, silently discarding the rest of the payload. That `0x00` isn't a terminator. It's just a byte that happens to be zero, sitting in the middle of a binary blob with four more bytes after it.

So `'\0'` can't be a universal boundary marker for arbitrary data, because for binary data, every byte value including zero is a legitimate payload byte. Length, on the other hand, doesn't care what values the bytes hold. It just says how many of them belong to you.

## Introducing memcpy()

Once you have an explicit length, the right copying primitive changes too.

```c
memcpy(destination, source, length);
```

`memcpy()` copies exactly `length` bytes. It doesn't search for anything inside them, and it doesn't care whether byte 40 happens to be zero. It just moves the number of bytes you told it to move, and stops. That makes it the natural fit for `[length][payload]` data, since the whole point of that representation was to make "how many bytes" an explicit, known value instead of something discovered by scanning.

But here's the part that's easy to skip past: **`memcpy()` is not automatically safe just because you used it instead of `strcpy()`.**

```c
char buffer[256];
memcpy(buffer, source, length);
```

If `length` is 1000, this overflows `buffer` just as badly as an unterminated `strcpy()` would. `memcpy()` trusts you completely. It will happily walk off the end of `source`, or write off the end of `buffer`, or both, if you hand it a length that doesn't actually fit. Switching functions didn't buy you safety. What buys you safety is validating the length before you ever call `memcpy()` at all.

## The two boundaries you're actually protecting

This is the part worth slowing down for, because it's easy to only think about half of it.

Every copy has two separate regions of memory involved, and each one has its own size that the length has to respect.

**The source boundary**: how much valid data actually exists at the other end of that pointer.

```
+---------------------------+
| valid source region       |
+---------------------------+
```

**The destination boundary**: how much space you actually have to write into.

```
+----------------+
| destination    |
+----------------+
```

A copy is only safe when both of these hold at the same time:

```
length <= source_size
length <= destination_capacity
```

It's tempting to check only one of these and feel done. Check only the source side, and you can overflow a destination buffer that was smaller than you assumed. Check only the destination side, and you can read past the end of a source region that was shorter than the length claimed. Both failures look identical from the outside: memory corruption, a crash, or worse, neither. The length has to fit both ends, not just one.

## Validating the length against the region itself

Now bring this back to the shared-memory scenario we started with. Say the shared region is `SIZE` bytes total, and it's laid out as a length field followed by the payload:

```
+----------------------+----------------------+
| length               | payload              |
+----------------------+----------------------+
```

Before touching the payload at all, the receiver has to check that the claimed payload actually fits inside the region it was given. The condition you want is:

```c
if (length <= SIZE - sizeof(length)) {
    // safe to read `length` bytes of payload
}
```

It's tempting to write this the other way around, checking `sizeof(length) + length <= SIZE` instead. Don't. That version adds two numbers before comparing, and if `length` is attacker-controlled or corrupted, the addition can overflow and wrap around to a small number that sails right past the check while `length` itself is enormous. The subtraction form avoids this: `SIZE - sizeof(length)` is a fixed, known-safe value computed once, so you're only ever comparing an untrusted number against it, never adding an untrusted number into something that could wrap.

## Turning length-delimited data into a C string

Length-delimited data never needed a `'\0'` to begin with; the length told you exactly where it ends. But sometimes the code on the receiving end genuinely wants an ordinary C string, because it's about to hand the data to a function that expects one.

That's fine, as long as you understand it as a conversion step, not the native representation:

```c
char string[length + 1];
memcpy(string, payload, length);
string[length] = '\0';
```

The `+ 1` and the manual assignment matter. The length-delimited payload is exactly `length` bytes, no more:

```
Length-delimited:
+----------------------+
| H e l l o            |
+----------------------+
```

A C string needs one extra byte tacked on at the end, the terminator, which was never part of the original data:

```
C string:
+-------------------------+
| H e l l o | \0         |
+-------------------------+
```

The shared representation on the wire, or in the shared region, stays length-based the whole time. The null terminator only gets added at the very last moment, locally, when you're producing a C string for some function that specifically requires one. It's a local convenience, not the source of truth for where the data ends.

## Why `char string[length]` alone is still dangerous

Notice that the snippet above uses `length + 1` on the stack. If `length` comes from shared or external data, you cannot trust it just because it's sitting right there in a length field. A corrupted region, or one written by a process you don't fully trust, can set `length` to whatever it wants, and an enormous value can blow the stack before you've written a single byte, trigger allocation failures your code never checked for, or drive reads and writes to addresses that have nothing to do with your actual data.

The length has to be validated against a real, known bound, like the region size we checked above, *before* it's used to size an allocation or drive a copy. Once that check is done, it's often simpler and safer to copy into a fixed-size buffer you already control, and reject or truncate anything that doesn't fit, rather than sizing a new buffer dynamically off a number you just received.

## A quick word on strncpy()

At this point someone always says: just use `strncpy()` instead of `strcpy()`, problem solved. It's worth being clear about why that's not the whole fix.

`strncpy()` takes a maximum number of bytes to copy, which sounds like what we want. But it still thinks in terms of strings: if the source runs out of bytes before hitting `'\0'`, it pads the rest of the destination with zero bytes, and if the source doesn't contain a `'\0'` within the given max length, it won't add one for you either. You can walk away with an unterminated buffer, having done exactly what the function documented, and still be back at square one.

`strncpy()` is a safer string function. It is not a length-delimited-data function. It still frames the problem in terms of a maximum scan for a terminator, not an explicit number of valid bytes. The actual fix was never "pick a better string function." It's moving away from string-shaped thinking entirely, from an implicit length discovered by scanning to an explicit length that's known up front.

## The takeaway

In C, a pointer tells you where the bytes are. It does not tell you how many of them are valid.

That single sentence is the whole article, if you need to remember one thing from it. Everything else follows from taking it seriously:

- **Pointer** = where the data starts.
- **Length** = how much of it is actually yours to read.
- **Buffer capacity** = how much you can safely write, a separate number from the length.
- **`'\0'`** = a string terminator, useful for text, not a universal marker for where arbitrary data ends.
- **`memcpy()`** = copies exactly the number of bytes you tell it to, nothing more, nothing less, and trusts you completely to have gotten that number right.
- **Validation** = the actual step that makes any of this safe, checked against both the source region and the destination buffer, before a single byte moves.

None of this is exotic. It's the same idea underneath length-prefixed protocol messages, `struct`-based binary formats, and most serious binary parsers you'll come across. The pointer was never going to tell you how many bytes were valid. You have to bring that number with you.
