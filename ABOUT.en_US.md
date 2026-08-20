# About Me — Amin Javaheri

> **Architect and Designer of Simurgh; Founder of the Project**

---

## Who is Amin Javaheri?

I am **Amin Javaheri**. A software developer, operating system architect, and researcher in the fields of distributed systems and security.

I have been working in the software industry for years. Over these years, I have worked with various languages, diverse architectures, and different operating systems — but above all, **Linux** has been my constant companion.

I love Linux. I learned from it. I grew up with it. But the deeper I explored it, the more I realized that some of its fundamental choices — though intelligent and necessary at the time — are no longer aligned with what modern hardware and the computing world demand from us.

---

## Why Simurgh?

For years, I worked with Linux — from the kernel to user space, from driver development to application development. Along this journey, I encountered numerous challenges:

- **The ever-increasing complexity of the kernel** — The Linux kernel has become so large and complex that full auditing and security verification is practically impossible.
- **User-based security model** — The UID/GID model, rooted in 1970s Unix, does not meet today's security needs.
- **Drivers in kernel space** — A single bug in a driver can compromise the entire system.
- **Insufficient attention to heterogeneous hardware** — GPUs, AI accelerators, CXL memory, and NUMA architectures were all added to Linux later, rather than being part of the initial architecture.
- **The C language** — For all its power, it does not guarantee memory safety, and many of Linux's critical vulnerabilities stem from this very issue.

> **I concluded that patching a 30-year-old system is not the right path. We must start from scratch.**

---

## The Birth of an Idea

The idea of Simurgh took shape in my mind when I asked myself this question:

> **If I were to design an operating system today, without any legacy constraints, what would it look like?**

The answer to that question turned into a personal project. And that project gradually grew into something much larger.

**Simurgh is the foundation of my PhD thesis.**

This project is not just "another operating system." It is a deep academic exploration of how an operating system can be designed with modern assumptions:

- Security from the ground up
- Fault isolation
- Maximum utilization of hardware
- And most importantly: **architecture for tomorrow's humans and machines**

---

## My Philosophy

I believe in a few things:

1. **Software must be understandable.** If we cannot fully comprehend a system, we cannot trust it.

2. **Security must be intrinsic, not bolted on.** You cannot patch security onto a 30-year-old system.

3. **Hardware is changing, and operating system architecture must change with it.** NUMA, AI accelerators, non-uniform memory — all of these must be part of the foundational design.

4. **Benchmarks don't lie.** Not ideology, not emotions. If something is better, it will be proven by data.

---

## Why Rust?

Choosing Rust was not a random or trendy decision.

Operating systems are the most dangerous software we write. Any bug in the kernel can be catastrophic.

Rust allows me to:

- Work at a low level (access to hardware)
- While guaranteeing memory safety
- Without needing a GC or hidden overhead

And most importantly: **Rust gives me the courage to be bolder.** Because I can build bigger things with more confidence.

---

## Vision

I am not building Simurgh to be a competitor to Linux.

I am building it to **answer a question**:

> **Can we build an operating system that is more secure, more resilient, and more intelligent — without sacrificing performance?**

If the answer is "yes," Simurgh will open a new path.

If "no," at least we tried and learned a great deal along the way.

---

## Join Me

Simurgh is an open-source project. Not because it's trendy, but because **I believe the best software is built through collective collaboration**.

If you are interested in security, operating system architecture, Rust, or distributed systems, I would be delighted to have you join this adventure.

---

### Simurgh

**Architecture for tomorrow, built with today's experience.**