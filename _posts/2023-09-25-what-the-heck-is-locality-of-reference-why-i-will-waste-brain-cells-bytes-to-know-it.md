---
layout: post
title: "what the heck is locality of reference , why i will waste brain cells bytes to know it?"
date: 2023-09-25 23:51:05 +0000
description: "Locality of Reference is a critical Computer Science topic and helps improving the software's..."
tags: [softwareengineering, cpu, softwaredevelopment]
canonical_devto: https://dev.to/wassim31/what-the-heck-is-locality-of-reference-why-i-will-waste-brain-cells-bytes-to-know-it-1l48
---

Locality of Reference is a critical Computer Science topic and helps improving the software's performance if the programmer is aware of it .

Programs that tend to exhibit good locality are those who're constituted of components that tend to reference data items that are near other recently referenced data items or that were recently referenced themselves .

There is 2 types of locality : 

1- Temporal locality : if a main memory's word is referenced once , and there are potential references in the near future , it should be in the cache memory.

2- Spatial locality : if a memory location is referenced once, then the program is likely to reference a nearby memory location in the near future ; A good example is a programming block which is logically distinguishable that is : Traversing a contiguous array of data using a loop statement .

Okay thank you for giving me information that i can get from Wikipedia , how the heck that will help me improve my programming skills & performance ?

Answer : We have the following example that performs matrix values sum

![example](/images/posts/what-the-heck-is-locality-of-reference-why-i-will-waste-brain-cells-bytes-to-know-it/img1.png)

the sumarraycols() function in Figure 6.19(a) computes the same result as the sumarrayrows function in Figure 6.18(a). The only difference is that we have interchanged the i and j loops. 

But What impact does interchanging the loops have on its locality?

The sumarraycols() function suffers from poor spatial locality because it scans

the array column-wise instead of row-wise. Since C arrays are laid out in memory row-wise.

The examples retrieved from the book : Computer Systems - Programmer's perspective.
