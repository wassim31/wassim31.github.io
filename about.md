---
layout: page
title: About
permalink: /about/
description: Wassim Boussebha Abdessamed is a backend software engineer and PhD researcher at IMT Atlantique working on distributed systems, storage systems, and operating systems.
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
    "jobTitle": "Backend Software Engineer and Doctoral Researcher",
    "description": "{{ page.description }}",
    "knowsAbout": ["Distributed Systems", "Storage Systems", "Cloud Computing", "Operating Systems", "Backend Development"],
    "sameAs": [
      "https://github.com/wassim31",
      "https://linkedin.com/in/wassimboussebha",
      "https://dev.to/wassim31",
      "https://twitter.com/oran_wassim"
    ]
  }
}
</script>

I'm Wassim Boussebha Abdessamed, a backend software engineer and doctoral researcher working at the intersection of distributed systems, storage systems, cloud computing, and operating systems.

### Current work

I'm a PhD researcher at [IMT Atlantique](https://www.imt-atlantique.fr/en) in Nantes, France. My research focuses on improving data movement efficiency in large-scale distributed infrastructures through novel storage and networking abstractions. I design operating-system and middleware mechanisms that optimize data transfer paths across distributed applications and storage tiers, investigating zero-copy techniques, caching strategies, and kernel-level optimizations for data-intensive workloads. I develop and evaluate research prototypes on Linux-based platforms using the [Grid'5000](https://www.grid5000.fr) research data center.

Before starting my PhD, I was a research intern at [Inria](https://www.inria.fr/en) in the STACK team, where I built a zero-copy system for multi-tier architectures handling image and video workloads. By overriding Linux system calls (`send()`/`recv()`) and using Redis as a shared buffer, the system exchanged only lightweight descriptors until the client fetched the actual response — achieving a 30% reduction in network bandwidth usage and 20–50% fewer I/O operations.

### Engineering background

Alongside research, I've spent several years building backend systems:

- At **Attraxia**, I built a secure authentication flow (OTP, email verification, token-based login) with Spring Boot adopted by 100+ users, and a real-time assignment tracking system using Java WebSockets that cut notification latency from minutes to milliseconds.
- At **SellSabeel**, I developed a Spring Boot platform that automates deployment of Docker-based applications to AWS EC2, cutting manual setup time from 1–2 hours to under 10 minutes.
- At **LabLabee**, I validated lab environments simulating 4G/5G/IMS networks with Docker containers and contributed to a migration from DigitalOcean to AWS.
- At **American Corner Oran**, I built a library and HR system with Spring Boot, ReactJS, and MySQL that reduced manual admin work by 20+ hours per week.

My broader interests include Linux kernel systems, cloud infrastructure management, and cloud-native applications (Kubernetes & eBPF). Some of my open source projects are on the [projects page](/projects/), and I write about Linux internals, containers, and systems programming on [dev.to](https://dev.to/wassim31).

### Elsewhere

You can find me on [GitHub](https://github.com/wassim31), [LinkedIn](https://linkedin.com/in/wassimboussebha), [dev.to](https://dev.to/wassim31), and [Twitter](https://twitter.com/oran_wassim). There's also my [CV](/cv/).

If you want to chat about distributed systems, storage, Linux internals, or backend engineering, email me at [wassim.boussebha@imt-atlantique.fr](mailto:wassim.boussebha@imt-atlantique.fr).
