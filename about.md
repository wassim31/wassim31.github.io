---
layout: page
title: About
permalink: /about/
description: Wassim Boussebha Abdessamed is a PhD researcher at IMT Atlantique working on AI infrastructure, distributed storage, and operating systems.
---

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "ProfilePage",
  "mainEntity": {
    "@type": "Person",
    "name": "Wassim Boussebha Abdessamed",
    "url": "{{ site.url }}",
    "image": "{{ site.url }}/face.jpg",
    "jobTitle": "AI Infrastructure & Distributed Storage Researcher",
    "description": "{{ page.description }}",
    "knowsAbout": ["AI Infrastructure", "Distributed Systems", "Storage Systems", "Operating Systems", "Linux", "eBPF", "Kubernetes"],
    "sameAs": [
      "https://github.com/wassim31",
      "https://linkedin.com/in/wassimboussebha",
      "https://dev.to/wassim31",
      "https://twitter.com/oran_wassim"
    ]
  }
}
</script>

I'm a PhD researcher at [IMT Atlantique](https://www.imt-atlantique.fr/en) working at the intersection of distributed systems, storage systems, cloud computing, and operating systems.

My main areas of interest include systems programming, distributed systems, Linux internals, networking, cloud-native infrastructure, and AI infrastructure. I enjoy working close to the system and understanding how software interacts with the kernel, network, hardware, and distributed infrastructure. Most performance stories — and most outages — are written at that layer, and I find it endlessly interesting.

I primarily work with C, Go, Linux, eBPF, Kubernetes, Docker, and networking technologies. I'm particularly interested in high-performance systems, distributed infrastructure, and the infrastructure required to build and run large-scale AI workloads.

### What I'm working on

My PhD research focuses on improving data movement efficiency in large-scale distributed infrastructures through novel storage and networking abstractions. In practice that means designing operating-system and middleware mechanisms that optimize how data travels between distributed applications and storage tiers — zero-copy techniques, caching strategies, kernel-level optimizations — and evaluating prototypes on Linux platforms in the [Grid'5000](https://www.grid5000.fr) research data center.

I got hooked on this problem during a research internship at [Inria](https://www.inria.fr/en)'s STACK team. I built a zero-copy system for multi-tier architectures handling image and video workloads: by overriding Linux's `send()`/`recv()` system calls and using Redis as a shared buffer, intermediate nodes exchanged only lightweight descriptors until the client actually needed the data. The result was a 30% reduction in network bandwidth and 20–50% fewer I/O operations. Watching redundant data movement disappear from a live system convinced me this was the layer I wanted to spend my career in.

### Before research

I came to research from engineering. Over several years I built backend systems for companies in Algeria: a secure authentication flow and a real-time WebSocket tracking system at Attraxia, a Spring Boot platform at SellSabeel that turned "here's a repo with a Dockerfile" into a running AWS deployment in under ten minutes, 4G/5G/IMS lab environments at LabLabee, and a library and HR system for American Corner Oran that saved the staff twenty hours of manual work a week. That background shapes how I do research — I care about systems that run, not just systems that publish.

Some of my open source projects are on the [projects page](/projects/), and I write about Linux internals, containers, and systems programming [here on this blog](/posts/) (originally published on [dev.to](https://dev.to/wassim31)).

### Elsewhere

You can find me on [GitHub](https://github.com/wassim31), [LinkedIn](https://linkedin.com/in/wassimboussebha), [dev.to](https://dev.to/wassim31), and [Twitter](https://twitter.com/oran_wassim). There's also my [CV](/cv/).

If you want to chat about AI infrastructure, distributed storage, Linux internals, or systems engineering, email me at [wassim.boussebha@imt-atlantique.fr](mailto:wassim.boussebha@imt-atlantique.fr).
