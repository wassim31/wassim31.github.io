---
layout: post
title: "CAFE BABE, if you know, you know. JVM enthusiast."
date: 2023-12-30 22:04:20 +0000
description: "Each program has a magic word in its machine code that defines its identity. For Java class files, it's CAFE BABE, and the story behind it is great."
tags: [java, linux, programming, reverse]
canonical_devto: https://dev.to/wassim31/cafe-babe-if-you-know-you-know-jvm-enthusiast-nda
image: /images/posts/cafe-babe-if-you-know-you-know-jvm-enthusiast/cover.png
---

Each program has a magic word in its machine code representation that defines its identity among multiple other files.

## Go see it yourself

If you open a compiled Java file, aka `file.class`, with a hex editor like the `xxd` tool in Linux, you'll find the first hex words are:

```
CAFE BABE
```

## But why CAFE BABE?

Why is it so magical that it has to be added to each class file? That's probably the most important question to be asked.

Well, James Gosling explained why:

> We used to go to lunch at a place called St Michael's Alley. According to local legend, in the deep dark past, the Grateful Dead used to perform there before they made it big. It was a pretty funky place that was definitely a Grateful Dead Kinda Place. When Jerry died, they even put up a little Buddhist-esque shrine. When we used to go there, we referred to the place as Cafe Dead. Somewhere along the line, it was noticed that this was a HEX number. I was re-vamping some file format code and needed a couple of magic numbers: one for the persistent object file, and one for classes. I used CAFEDEAD for the object file format, and in grepping for 4 character hex words that fit after "CAFE" (it seemed to be a good theme) I hit on BABE and decided to use it. At that time, it didn't seem terribly important or destined to go anywhere but the trash can of history. So CAFEBABE became the class file format, and CAFEDEAD was the persistent object format. But the persistent object facility went away, and along with it went the use of CAFEDEAD. It was eventually replaced by RMI.

So every Java class file in the world starts with the name of a cafe where the Grateful Dead used to play. If you know, you know.
