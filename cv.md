---
layout: page
title: Wassim Boussebha Abdessamed's CV
permalink: /cv/
description: CV for Wassim Boussebha Abdessamed, backend software engineer and doctoral researcher in distributed systems at IMT Atlantique.
---

Backend software engineer and doctoral researcher with experience across backend systems, cloud infrastructure, and systems research. Currently researching data movement efficiency in large-scale distributed infrastructures at IMT Atlantique.

### Research

**[IMT Atlantique](https://www.imt-atlantique.fr/en)** (Jan 2026--present) -- Doctoral Researcher, Nantes, France. Research on distributed systems, storage architectures, and cloud infrastructures, focused on reducing data movement overhead and improving resource efficiency in large-scale systems. Designing operating-system and middleware mechanisms to optimize data transfer paths across distributed applications and storage tiers. Investigating zero-copy techniques, caching strategies, and kernel-level optimizations for data-intensive workloads. Prototyping on Linux-based platforms using the Grid'5000 research data center.

**[Inria](https://www.inria.fr/en)** (Apr 2025--Oct 2025) -- Research Intern, STACK team (remote). Implemented a zero-copy system for multi-tier architectures handling image/video workloads, overriding Linux system calls (`send()`/`recv()`) to minimize redundant data movement across intermediate nodes. Integrated Redis as a shared buffer so only lightweight descriptors were exchanged until the client fetched the actual response. Achieved a 30% reduction in network bandwidth usage and 20--50% fewer I/O operations.

### Employment

**Attraxia** (Aug 2024--Mar 2025) -- Backend Developer (part-time), Algeria. Built a secure auth flow (OTP, email verification, token-based login/logout) with Spring Boot, adopted by 100+ users. Designed a real-time assignment tracking system with the Java WebSockets API, cutting notification latency from minutes to milliseconds. Containerized and deployed the platform via Docker and GitLab CI/CD, reducing release cycles from 1 day to under 1 hour.

**SellSabeel** (Sep 2023--Mar 2024) -- Backend Developer (part-time), Algeria. Developed a Spring Boot web application that automates deployment of Docker-based applications: users provide a GitHub repository with a Dockerfile or docker-compose.yml, and the system provisions an AWS EC2 instance, installs dependencies, and runs docker-compose -- cutting manual setup time from 1--2 hours to under 10 minutes. Supported multiple concurrent deployments.

**LabLabee** (Apr 2022--Sep 2022) -- System Engineer (intern), Algeria. Validated lab environments simulating 4G/5G/IMS with Docker containers; ensured connectivity, QoS, and KPIs. Contributed to a migration from DigitalOcean to AWS and built interactive documentation for training labs.

**American Corner Oran** (Sep 2021--Mar 2022) -- Backend Developer (part-time), Oran, Algeria. Developed a library and HR system with Spring Boot, ReactJS, and MySQL. Reduced manual admin work by 20+ hours/week and sped up record lookups by 80%.

### Education

**National Higher Institute of Telecommunications**, Oran, Algeria (Dec 2020--Jun 2025) -- Master's in Computer Networking. Coursework: computer networks, distributed systems, network security, algorithms and data structures, system programming, telecommunications systems.

### Projects

**[OptRoute](https://outstanding-creativity-production-b5ce.up.railway.app/)** -- Route optimization web app for efficient path planning and delivery routing.

**[Orsh](https://github.com/wassim31/Orsh)** -- Open source shell written in C for Linux systems.

**[Laravel Passport](https://github.com/wassim31/Laravel_Passport)** -- OAuth2 server implementation for secure API authentication.

**[Django Assignment Tracker](https://github.com/wassim31/Django-Assignment-Tracker-Authentication-API)** -- RESTful API for assignment tracking and authentication.

More on the [projects page](/projects/).

### Writing

Technical articles on [dev.to](https://dev.to/wassim31) covering Linux internals, containers, virtualization, and systems programming.

### Technical skills

Java, Python, C, JavaScript, PostgreSQL, MySQL. Spring Boot, Django, NestJS. AWS, Docker, Kubernetes, Git, GitLab CI, Ansible. Linux kernel systems, eBPF, socket programming, zero-copy I/O.

### Contact

[wassim.boussebha@imt-atlantique.fr](mailto:wassim.boussebha@imt-atlantique.fr) · [GitHub](https://github.com/wassim31) · [LinkedIn](https://linkedin.com/in/wassimboussebha)
