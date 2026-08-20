# Simurgh — The Intelligent Operating System

> **A new operating system architecture built around security, isolation, hardware awareness, and intelligent computing.**

## Why Simurgh?

Every operating system begins with a set of assumptions about how software should interact with hardware.

For decades, many operating systems have followed a relatively similar path: a large privileged kernel provides a broad set of services, while applications and system components interact with that kernel through increasingly complex interfaces.

Linux is an extraordinary example of how successful this model can be. It powers everything from embedded devices and smartphones to cloud infrastructure and supercomputers.

But hardware is changing.

Modern systems are no longer simply:

```text
CPU + RAM + Storage + Network
```

A contemporary computer may contain:

```text
CPU
├── Multiple cores
├── Multiple NUMA nodes
├── GPU
├── NPU
├── TPU
├── FPGA
├── High-speed storage
├── High-speed networking
└── Multiple heterogeneous memory domains
```

At the same time, modern workloads are becoming increasingly heterogeneous:

```text
Interactive applications
Games
Cloud services
AI inference
AI training
Real-time workloads
Scientific computing
Edge computing
```

This led to a simple question:

> **If we were designing an operating system today, without being constrained by the historical evolution of existing operating systems, what would its foundations look like?**

That question is the starting point of **Simurgh**.

---

# The Idea

**Simurgh** is an experimental operating-system architecture designed from the ground up around several principles:

* strong isolation
* capability-based security
* minimal privileged code
* hardware abstraction
* user-space system services
* high-performance IPC
* heterogeneous computing
* NUMA awareness
* zero-copy data exchange
* memory safety
* and an architecture designed with AI workloads in mind

The name comes from the **Simurgh**, the legendary Persian bird associated with wisdom, protection, knowledge, and renewal.

The philosophy behind the name is intentional:

> **Simurgh is not intended to be another Linux distribution. It is an exploration of what an operating system could look like if its foundations were redesigned for modern hardware and workloads.**

---

# The First Principle: Keep the Privileged Core Small

One of the most important decisions in Simurgh is the use of a **microkernel architecture**.

Instead of putting drivers, filesystems, networking and other operating-system services inside a huge privileged kernel, Simurgh attempts to keep the privileged kernel responsible for a much smaller set of fundamental mechanisms.

Conceptually:

```text
┌───────────────────────────────────────────────┐
│                Applications                   │
├───────────────────────────────────────────────┤
│             System Services                  │
├───────────────────────────────────────────────┤
│      VFS / Network / Drivers / Services       │
│                  User Space                   │
├───────────────────────────────────────────────┤
│                 Microkernel                   │
│                                               │
│   Memory • IPC • Capability • Scheduling      │
├───────────────────────────────────────────────┤
│                    HAL                        │
├───────────────────────────────────────────────┤
│                   Hardware                    │
└───────────────────────────────────────────────┘
```

The first two layers of the architecture are therefore the foundation of the entire project:

**Layer 1 — Hardware Abstraction Layer**

**Layer 2 — Capability-Based Microkernel**

The complete architecture is intentionally developed from the bottom upward. 

---

# 01 — Hardware Abstraction Layer

The first question an operating system must answer is:

> **What hardware actually exists on this machine?**

Simurgh places a dedicated **Hardware Abstraction Layer (HAL)** directly above the hardware.

The HAL is the only layer that directly interacts with hardware-specific mechanisms such as:

* CPU registers
* MMIO
* interrupt controllers
* architecture-specific instructions
* firmware-provided memory maps
* timers
* architecture-specific boot mechanisms

Everything above the HAL should see an abstract hardware model rather than individual CPU architectures. 

The initial architecture targets:

```text
x86_64
ARM64
RISC-V RV64GC
```

The HAL exposes common Rust traits so that the Microkernel does not need architecture-specific conditionals throughout its implementation. 

---

# Hardware Discovery Instead of Hardware Assumptions

A particularly important idea in Simurgh is **Discovery + Policy**.

The HAL does not decide whether a machine is:

```text
Gaming
AI
Professional
General Purpose
```

It simply discovers what the machine actually contains.

For example:

```text
CPU
64 cores

RAM
256 GB

GPU
NVIDIA ...

NPU
Available

NUMA
4 nodes

IOMMU
Available
```

This information is converted into a standardized **Hardware Manifest**.

The upper layers therefore do not need to know whether the machine was booted through UEFI, Device Tree, or another architecture-specific mechanism. 

The current design also treats GPU/NPU/TPU/FPGA resources as first-class compute resources rather than merely generic peripheral devices. 

---

# 02 — The Microkernel

The second foundation is the Simurgh Microkernel.

The Microkernel deliberately contains a small set of fundamental mechanisms:

```text
Memory Management
Capability Management
IPC
Scheduling
Threads
Kernel Objects
```

The goal is not to build a smaller version of Linux inside the kernel.

The goal is to make the privileged part of the system **small, understandable, auditable and strongly isolated**.

The Microkernel communicates with user-space services through a controlled syscall boundary. 

---

# Capability-Based Security

This is perhaps the most fundamental architectural difference.

Traditional Unix/Linux security revolves heavily around concepts such as:

```text
UID
GID
File Permissions
ACL
Root
```

Simurgh instead uses **Capabilities as the fundamental authorization primitive**.

A process does not automatically have access to a resource simply because it is running as a particular user.

It needs a capability granting access to that resource.

Conceptually:

```text
Process
   │
   ├── Capability → Memory
   │
   ├── Capability → Device
   │
   ├── Capability → IPC Endpoint
   │
   └── Capability → Shared Memory
```

A capability contains a reference to a kernel object together with explicit rights.

Capabilities can also be transferred explicitly and revoked, including derived capabilities through the capability derivation model. 

This gives Simurgh a security model where:

> **Access is something a process possesses explicitly, rather than something it implicitly inherits from its identity.**

---

# Why This Matters

Consider a device driver.

In a traditional monolithic-kernel architecture, a driver normally executes in privileged kernel space.

If the driver contains a serious bug, the consequences can potentially reach the kernel itself.

Simurgh instead aims to run drivers as isolated user-space processes.

Conceptually:

```text
┌────────────────────────────┐
│       Device Manager       │
├────────────────────────────┤
│       Driver Process       │
│                            │
│  Capability:               │
│    MMIO → Device A         │
│    IRQ  → Device A         │
└────────────────────────────┘
```

A driver should receive only the capabilities required for its specific device.

The architectural goal is therefore:

```text
Driver crash
     ↓
Restart driver
     ↓
System continues running
```

rather than:

```text
Driver crash
     ↓
Kernel corruption
     ↓
System crash
```

This user-space isolation model is one of the central motivations for the Microkernel architecture.

---

# IPC Is the Performance-Critical Part

A microkernel architecture introduces an important challenge:

> **Communication between isolated components must be extremely efficient.**

In Simurgh, IPC is therefore treated as a first-class architectural component rather than an implementation detail.

Two primary communication paths are proposed:

### Small synchronous messages

For operations that transfer only a few words:

```text
Process A
   │
   │ IPC Call
   ↓
Process B
```

The message can be transferred without copying large buffers.

### Large data

For things such as:

```text
AI tensors
GPU buffers
Video frames
Network packets
```

Simurgh uses shared memory capabilities.

Instead of:

```text
Process A
 ↓ copy
Kernel
 ↓ copy
Process B
```

the architecture aims for:

```text
Process A
      │
      ▼
Shared Memory
      ▲
      │
Process B
```

with zero data copies.

This is explicitly part of the Microkernel design. 

The MVP consequently includes a target of benchmarking the IPC fast path rather than simply assuming that the architecture will be fast. 

---

# A Different Approach to Scheduling

Modern workloads are not identical.

A desktop UI wants:

```text
Low latency
```

An AI inference engine may want:

```text
Maximum throughput
```

A game may care about:

```text
Frame-time consistency
Low input latency
GPU locality
```

A server may care about:

```text
Throughput
NUMA locality
Predictability
```

Simurgh therefore separates scheduling behavior into modes such as:

```text
Interactive
Throughput
```

and makes NUMA and compute-device locality explicit inputs to scheduling decisions.

The proposed design also introduces the concept of an **IPC Chain Group**, allowing a chain such as:

```text
Application
    ↓
VFS
    ↓
Driver
```

to be treated as a connected scheduling workload rather than unrelated threads. 

Importantly, this is an area where Simurgh intends to be **benchmark-driven**. If the custom scheduler does not outperform a proven CFS-like reference workload, the design should be reconsidered rather than defended for ideological reasons. 

---

# Why Rust?

Choosing Rust was not primarily about fashion.

It follows directly from the architecture.

An operating system contains some of the most dangerous classes of software bugs:

```text
Buffer overflows
Use-after-free
Double-free
Data races
Invalid memory access
Concurrency bugs
```

These bugs are particularly dangerous in privileged software.

Rust gives Simurgh a combination that is attractive for this kind of project:

```text
Low-level control
+
Memory safety
+
Strong type system
+
Concurrency guarantees
+
No garbage collector
```

This is especially valuable because Simurgh intends to move as much functionality as possible out of the privileged kernel.

The architecture therefore uses Rust across the HAL and Microkernel, with `no_std` at the lowest levels and only minimal architecture-specific assembly where required during early bootstrap. 

The philosophy is simple:

> **Use unsafe code where the hardware requires it, but make every unsafe boundary explicit, minimal and documented.**

---

# Simurgh vs. Linux

The purpose of Simurgh is **not** to claim that Linux is obsolete.

Linux is one of the most successful and important operating systems ever created.

The question is different:

> **What trade-offs would we make if we were starting from a clean architectural foundation today?**

Here is the high-level distinction:

| Area                 | Linux approach                       | Simurgh direction                          |
| -------------------- | ------------------------------------ | ------------------------------------------ |
| Kernel architecture  | Large monolithic kernel              | Microkernel                                |
| Drivers              | Primarily kernel space               | User-space isolation                       |
| Security foundation  | UID/GID + permissions + capabilities | Capability-first                           |
| Hardware abstraction | Architecture-specific kernel code    | Dedicated HAL                              |
| IPC                  | Multiple mechanisms                  | First-class kernel primitive               |
| Large data transfer  | Multiple paths                       | Capability-based shared memory             |
| Fault isolation      | Limited for kernel components        | Process-level isolation                    |
| Memory safety        | C + other languages                  | Rust-first                                 |
| CPU architectures    | Broad support                        | Architecture-neutral upper layers          |
| NUMA                 | Supported                            | Designed as a first-class scheduling input |
| AI accelerators      | Evolving ecosystem                   | First-class discovery model                |
| Kernel size          | Very large                           | Intentionally minimal                      |
| System services      | Many kernel-integrated components    | User-space services                        |

The important point is that **these are architectural goals, not claims of proven superiority**.

Simurgh has to earn every performance and reliability claim through implementation and benchmarking.

---

# Where Could Simurgh Have an Advantage?

## 1. Security

A smaller privileged kernel means a smaller amount of code operating at the highest privilege level.

Combined with capabilities:

```text
Minimal Kernel
+
Capability Security
+
Process Isolation
+
Rust
```

the architecture aims to reduce the consequences of component failures.

---

## 2. Fault Isolation

A driver should not need to be part of the trusted computing base simply because it controls hardware.

Instead:

```text
Driver
 ↓
Capability
 ↓
Device
```

This creates the possibility of restarting individual components independently.

---

## 3. Hardware Evolution

The HAL deliberately separates hardware discovery from operating-system policy.

This means future hardware such as:

```text
GPU
NPU
TPU
FPGA
CXL memory
```

can be represented in the Hardware Manifest without requiring the entire upper architecture to understand each hardware vendor's implementation. 

---

## 4. AI Workloads

AI workloads are fundamentally different from traditional desktop workloads.

They depend heavily on:

```text
GPU/NPU
Memory bandwidth
NUMA locality
Large shared buffers
DMA
High-speed networking
```

Simurgh's architecture considers these resources from the beginning rather than treating AI as an application category added later.

The combination of:

```text
Heterogeneous compute discovery
+
NUMA-aware scheduling
+
Capability-based device access
+
Zero-copy shared memory
+
High-performance IPC
```

is intended to provide a foundation for AI-oriented system services.

---

# But Simurgh Has Real Costs

A serious open-source project should also be honest about its disadvantages.

A Microkernel architecture is not automatically faster.

It introduces:

```text
IPC overhead
Context switching
More complex service coordination
More complex debugging
More components
More interfaces
```

A new filesystem is difficult.

A new driver ecosystem is difficult.

POSIX compatibility is difficult.

Supporting multiple CPU architectures is difficult.

And building an ecosystem comparable to Linux is an enormous undertaking.

Therefore:

> **Simurgh is an architectural experiment first, and a production operating system only if the implementation proves the architecture in practice.**

That distinction is important.

---

# The First Two Layers

The current project deliberately starts at the bottom.

## Layer 1 — HAL

The HAL establishes a hardware-neutral interface for:

```text
CPU
Memory
Timers
Interrupts
Boot
IOMMU
Compute devices
Power / thermal information
```

and produces a standardized **Hardware Manifest** for the Microkernel. 

Read the complete design:

**[`01-HAL-Layer.md`](01-HAL-Layer.md)**

---

## Layer 2 — Microkernel

The Microkernel builds on that foundation and provides:

```text
Capability
Memory Management
IPC
Scheduling
Kernel Objects
Threads
```

Its syscall surface is intentionally kept small, and the MVP explicitly requires testing IPC performance, capability revocation, zero-copy memory sharing, and syscall fuzzing. 

Read the complete design:

**[`02-Microkernel-Layer.md`](02-Microkernel-Layer.md)**

---

# The Goal

The goal of Simurgh is not to recreate Linux with different source code.

It is to explore a different set of fundamental assumptions:

```text
Hardware is heterogeneous.
Security should be capability-based.
Drivers should not need kernel privilege.
The privileged core should be minimal.
Communication should be explicit and fast.
Large data should move without unnecessary copies.
Scheduling should understand modern hardware topology.
Memory safety should be a foundational property.
AI should be considered a first-class workload.
```

And most importantly:

> **The architecture should be judged by measurements, not ideology.**

If an idea is faster, safer or more reliable, we keep it.

If a benchmark proves that a traditional approach is better, we use the better approach.

That is the spirit of **Simurgh**.

---

## Simurgh

### *The Intelligent Operating System*

**An experimental operating-system architecture designed for modern hardware, secure isolation, and intelligent computing.**

---


