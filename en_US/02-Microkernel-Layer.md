# Layer 2 — Microkernel (Capability-Based)

> **Layer:** 2 - Microkernel (Capability-Based)
> **Status:** Architecture Specification
> **Implementation language:** Rust (`no_std` up to the IPC boundary; no part of this layer may depend on an unbounded heap allocator or ordinary panic-unwinding)
> **Execution type:** Kernel-space, privileged mode
> **Design inspiration:** seL4 (capability model and provability), Zircon (IPC model and object handles)

---

## 0. Relationship to the Layer Below and Above

**Layer below (HAL, Layer 1):** Per the Layer 1 document (Section 0), the microkernel talks to the HAL via a **direct function/trait call**, not IPC — both layers are linked into a single privileged binary. The microkernel is initialized with `HardwareManifestRaw` (Layer 1 document, Section 9) at boot time.

**Layer above (user-space subsystems, Layer 3):** Communication happens across the **syscall boundary** — the only point in the whole architecture where a real hardware context switch (from Ring 3/EL0/U-mode to Ring 0/EL1/M-mode) actually occurs:

```
Layer-3 process  --[syscall/svc/ecall instruction]-->  Microkernel (Layer 2)
Layer-3 process  <--[return with result in register]--  Microkernel (Layer 2)
```

The syscall surface (Section 6, `SyscallOp`) is the only API layer 3 and above has for talking to the microkernel. Real IPC between two layer-3 processes (e.g. an app and the VFS) also goes through this same path: both sides talk to the microkernel (`ipc_call`/`ipc_send_async`), and the microkernel is the intermediary for message/Capability transfer — the two processes never access each other directly. The `kernel-ipc` crate (Section 7) wraps this boundary as a higher-level API for Rust code in layer 3, so upstream code is never forced to write raw trap instructions.

The microkernel must be **the smallest amount of code possible running in privileged mode**. The golden rule:

> If a feature can be implemented in user-space, it does not belong in the kernel.

This layer has exactly four responsibilities, no more: **memory management, scheduling, IPC, and Capability**. The file system, drivers, networking, and even the traditional notion of a "process" all live in layer 3 (user-space).

## 1.1 Decision: Formal Verification Is Designed In From Day One

Full formal verification (seL4-style) is not in the MVP's scope, but **the code is written from the start under constraints that keep the path to provability open in phase 2**, rather than forcing a rewrite later:

- `unsafe` is forbidden without a documented justification comment (per Section 7 of the HAL document; the same rule is even more mandatory here since this layer is privileged)
- The syscall dispatcher must be written as an explicit **state machine**; every function must have a bounded, traceable effect, with no hidden side effects or mutation through scattered global state
- The Capability Derivation Tree structure (Section 2) follows seL4's proven pattern exactly, rather than a custom design — because this is the security heart of the system, and the risk of a from-scratch redesign has no justification
- Security-critical functions (granting/revoking a Capability, retyping memory) must document their own pre/post-conditions as structured comments, even before a formal verification tool (such as Kani or Prusti) enters the project — these comments later become the direct basis for proof annotations

## 2. The Capability Model — The Security Core of the Entire OS

Instead of Linux's traditional UID/GID/permission model, every resource (memory, a device, an IPC port, an HAL-Direct token) is represented by a **Capability**:

```rust
/// A Capability = an unforgeable reference to a Kernel Object + a set of rights
pub struct Capability {
    pub object_ref: KernelObjectRef,
    pub rights: CapabilityRights,   // bitflags: READ, WRITE, EXECUTE, GRANT, DUPLICATE, REVOKE
    pub badge: u64,                 // arbitrary identifier for distinguishing requests during IPC
}

bitflags::bitflags! {
    pub struct CapabilityRights: u32 {
        const READ     = 0b00001;
        const WRITE    = 0b00010;
        const EXECUTE  = 0b00100;
        const GRANT    = 0b01000; // permission to transfer this Capability to another process
        const DUPLICATE= 0b10000;
    }
}
```

**Key rules:**
- No process has access to any resource by default; only by holding a valid Capability
- Capabilities are transferred only through explicit IPC (`grant`), never implicitly
- Revocation must be possible: the kernel must be able to invalidate a Capability and all of its derivatives (children, in the case of duplication) — the CDT pattern (Capability Derivation Tree, same as seL4)

## 3. Memory Management

The microkernel itself does not allocate pages in the sense of a general-purpose `malloc`; instead, **physical memory is itself a Capability** (`UntypedMemory` — exactly the seL4 model):

- At boot, all physical memory reported in the Hardware Manifest (from layer 1) is handed to the first process (the Root Task, in layer 3) as a set of `UntypedMemory` objects
- The Root Task is responsible for dividing this memory among services, not the kernel itself — meaning **memory allocation policy does not live in the kernel**, only the mechanism does
- The kernel's core memory operations are: `retype` (converting `UntypedMemory` into a concrete type such as `PageTable`, `ThreadControlBlock`, or `Endpoint`), `map`, `unmap`

```rust
pub enum KernelObjectType {
    UntypedMemory,
    PageTable,
    ThreadControlBlock,
    Endpoint,        // for IPC
    Notification,    // for async signals
    CapabilitySpace, // a process's table of Capabilities
}
```

## 4. Scheduling (Scheduler)

Two modes, both backed by the HAL's Timer support, matching the needs of the layer-4 profile:

| Mode | Base algorithm | Use case |
|---|---|---|
| Interactive | Priority-based with aging + short quantum (~1-4ms) | General-purpose, gaming |
| Throughput/Batch | Custom algorithm (Section 4.1) | AI batch, professional |

### 4.1 Decision: A Custom Algorithm for Throughput Mode

Rather than directly implementing CFS or EEVDF, a dedicated algorithm is designed from the ground up to be optimal for our dual-mode model (not a generic algorithm patched later). Initial design notes:

- **The unit of scheduling is not a single thread, but an "IPC group"**: because in this architecture an operation is typically an IPC chain across several processes (e.g. app → VFS → driver), the algorithm must account for the cost of the whole chain, not just one isolated thread — something neither CFS nor EEVDF was natively designed for (because in a monolithic Linux kernel this chain doesn't exist at all).
- **NUMA and Compute Device Affinity awareness** as a first-class input to the algorithm, not a bolted-on side layer (something that was added late and with difficulty to Linux's EEVDF as well)
- **Weight selection criterion**: a combination of static priority (from Profile Policy) + time spent waiting in the queue (aging-like) + an "IPC chain cost" factor as mentioned above
- The risk of this path (a from-scratch design) must be managed with heavy benchmarking against a CFS-like reference implementation on the same workload — if the custom algorithm is not, in practice, better than plain CFS, we must switch back to a CFS-like algorithm without hesitation; this decision must remain data-driven, not ideological

### 4.3 Final Decision: The Weight Formula in Throughput Mode

**Base formula (first version; its numeric parameters are tuned by real benchmarking, but the algorithm's structure itself is fixed):**

```
vruntime_next(thread) = vruntime_current(thread) + (actual_runtime_ns / effective_weight)

effective_weight = base_priority_weight
                  × (1 + aging_factor × min(wait_time_ms, aging_cap_ms))
                  × numa_locality_bonus
```

- **`base_priority_weight`**: comes from Profile Policy (layer 4); for example, the AI profile gives a higher base value to inference threads
- **`aging_factor` and `aging_cap_ms`**: prevent starvation; the longer a thread waits in the queue, the higher its effective weight rises, but it is capped so the primary prioritization isn't broken
- **`numa_locality_bonus`**: a thread scheduled on a core close to its memory/compute device gets a bonus (a reduction in incremental vruntime), to encourage the scheduler to preserve NUMA locality

**IPC chain accounting model:** rather than accounting for each thread in an IPC chain (e.g. app → VFS → driver) separately, all threads participating in one synchronous IPC chain share a common **Chain Group ID**, and `vruntime` accrues at the group level rather than per individual thread:

```rust
pub struct ChainGroup {
    pub id: ChainGroupId,
    pub member_threads: Vec<ThreadId>,
    pub group_vruntime: u64,
}
```

This means a long IPC chain splits its scheduling cost fairly among its members, rather than getting two or three times the CPU share just because it involves multiple threads (a problem that exists in simple models like plain CFS, because IPC was never accounted for in its original design).

**Resolving the numeric parameters:** the proposed starting values for benchmarking are `aging_factor = 0.02`, `aging_cap_ms = 50`, `numa_locality_bonus = 0.9` (a 10% reduction in incremental vruntime for local access). These are starting points, not final numbers — precise tuning is part of the MVP's performance test phase, not an open architectural question.

### 4.4 General Scheduler Notes

- Mode selection happens at the **per-thread** level, not system-wide; an AI-inference thread can be in throughput mode while the UI on the same machine is in interactive mode
- The scheduler must be aware of NUMA topology information (from the HAL) to reduce the cost of cross-node memory access
- Priority inheritance is mandatory (to prevent priority inversion across IPC chains)

## 5. IPC — The Heart of the Microkernel Architecture

Because everything (drivers, file system, networking) runs as a separate process in layer 3, **IPC performance directly determines the performance of the whole system**. This is the single most critical area to optimize.

### 5.1 Two Types of IPC

```rust
// 1. Synchronous, small message — for fast calls like syscall-like RPC
pub struct Endpoint;
pub fn ipc_call(endpoint: Capability, msg: SmallMessage) -> SmallMessage;
// SmallMessage: at most a few words, transferred directly in registers, no memory copy

// 2. Asynchronous, bulk data — for transferring large data (e.g. an AI/GPU buffer)
pub struct Notification;
pub fn ipc_send_async(notif: Capability, shared_buffer: Capability);
```

### 5.2 Zero-Copy for Bulk Data
For large buffers (e.g. a video frame, an AI tensor), instead of copying, a **Capability to a shared memory region** is transferred; the receiver simply maps it into its own address space. The copy count is zero.

```rust
pub fn create_shared_region(size: usize, rights: CapabilityRights) -> Capability;
pub fn map_shared_region(cap: Capability, at: Option<VirtAddr>) -> VirtAddr;
```

### 5.3 IPC Fast Path
For high-traffic Endpoints (e.g. between the scheduler and a real-time driver), a "fast path" must exist that minimizes the context switch — this is exactly the well-known L4 optimization that brought IPC cost down from ~microseconds to ~hundreds of nanoseconds. This must be benchmarked and is part of the acceptance criteria (Section 8).

## 6. Kernel Object Model

```rust
pub enum SyscallOp {
    Send { endpoint: CapId, msg: SmallMessage },
    Recv { endpoint: CapId },
    Call { endpoint: CapId, msg: SmallMessage },   // atomic Send + Recv
    Yield,
    CapGrant { target_thread: CapId, cap: CapId, rights: CapabilityRights },
    CapRevoke { cap: CapId },
    Retype { untyped: CapId, target_type: KernelObjectType, count: usize },
    Map { page_table: CapId, frame: CapId, vaddr: VirtAddr, perms: MapPermissions },
}
```

The syscall surface must stay **very small** (seL4 has roughly 10-15 syscalls; our target is similar, not anywhere near Linux's 350-400 syscalls).

## 7. Directory Structure

```
kernel/
 ├── kernel-core/        # syscall dispatcher, object model, no_std, architecture-independent
 ├── kernel-mm/          # UntypedMemory management, PageTable ops
 ├── kernel-sched/       # scheduler, Interactive + Throughput mode
 ├── kernel-ipc/         # Endpoint, Notification, Shared Region, fast path
 ├── kernel-cap/         # Capability model, CDT, revocation
 └── kernel-arch-glue/   # bridge between kernel-core and hal-core traits (layer 1)
```

## 8. MVP Definition of Done
1. Boot on all three architectures with control handed off from the HAL, and creation of the first `UntypedMemory` objects
2. Successful execution of a minimal Root Task that creates a second thread and establishes synchronous IPC with it
3. Benchmark: `ipc_call` cost on the fast path under 500 nanoseconds on reference hardware (initial target; adjustable after real benchmarking)
4. Zero-copy shared memory between two processes, with a test proving no data copy occurred
5. Revocation test: invalidating a Capability and proving its derivatives are no longer valid either
6. Fuzz testing of the syscall surface (since this layer is the system's primary attack surface)

## 9. Remaining Open Questions
No open architectural questions remain in this layer. Precise tuning of the weight formula's numeric constants (Section 4.3) is naturally part of the implementation's benchmark phase, not a pending design decision.
