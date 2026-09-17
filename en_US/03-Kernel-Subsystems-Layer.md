# Layer 3 — Kernel Subsystems (User-Space Services)

> **Layer:** 3 - Kernel Subsystems (User-Space Services)
> **Status:** Architecture Specification
> **Implementation language:** Rust for everything, except where an exception is explicitly stated
> **Execution type:** User-space, but at high scheduling priority (near-kernel priority); each service is a separate, isolated process with a restricted Capability set

---

## 0. Relationship to the Layer Below and Above

**Layer below (microkernel, Layer 2):** every service in this layer (Device Manager, VFS, Netstack, Compositor, mm-service) is an ordinary user-space process that talks to the microkernel only through **syscalls + Capabilities** (exactly the boundary described in the Layer 2 document, Section 0). These services have no special privilege over an ordinary layer-5 application; the only difference is the larger set of Capabilities the Root Task granted them at startup (e.g. a Capability for MMIO access to a specific device).

**Layer above (system services, Layer 4):** communication happens through a **high-level IPC protocol** defined with a shared IDL (Section 3 of this document) — not raw syscalls. Examples:
- When the POSIX Compat Layer (layer 4) has `open()` called, it internally sends an `FsRequest::Open` to the VFS Service
- The Package Manager (layer 4) performs IPC with the VFS Service and the Device Manager (for target disk/device space) to install a package
- The Security Broker (layer 4) ultimately validates and transfers the Capabilities it issues (such as access to a specific driver) through the microkernel (layer 2), but the policy decision of "who is allowed" is made in layer 4

An important architectural note: **layer 4 never issues syscalls directly to the microkernel (layer 2)** except for very limited administrative operations (such as issuing a Capability, which itself requires microkernel access); the main data path always passes through this layer's (3) protocols.

This layer contains everything that "everyone assumes is part of the kernel," but in reality none of it is privileged. Every component of this layer is a separate process at the microkernel level that only has access to the Capabilities it has been explicitly given.

**The first process the microkernel runs is called the "Root Task"**, and it is responsible for the initial bring-up of these subsystems and for distributing initial Capabilities (including each one's share of `UntypedMemory`).

## 2. Main Components

### 2.1 Device Manager (Driver Manager)

```
device-manager/
 ├── receives the Hardware Manifest (via IPC with the Root Task, which got it from the HAL)
 ├── launches a separate process for each driver (Driver Process Isolation)
 ├── issues a restricted Capability to each driver:
 │    - only the MMIO region belonging to that device (not all of memory)
 │    - only that device's IRQ
 └── Restart policy: if a driver crashes → the Device Manager brings it back up
     in a new process without the system going down (this is the single biggest
     win of this whole architecture over a monolithic Linux kernel)
```

**Driver model:**
```rust
#[async_trait]
pub trait DeviceDriver {
    fn probe(manifest_entry: &ComputeDevice) -> Result<Self, DriverError> where Self: Sized;
    async fn handle_irq(&mut self, irq: IrqId);
    async fn handle_request(&mut self, req: DriverRequest) -> DriverResponse;
}
```
Each driver is a separate Rust crate that implements this trait, compiled and run inside its own isolated process sandbox.

### 2.2 VFS Service (File System)

- Instead of the file system being part of the kernel (as in Linux), each file system (ext4-compat, btrfs-compat, a new native file system) is a **separate service** sitting behind a VFS Router
- Communication between an application and the VFS happens over IPC (layer 2) using a standard message protocol (`FsRequest`/`FsResponse`)
- For performance: the page cache is implemented as a Shared Memory Region between the VFS Service and the Kernel Memory Manager, to avoid extra copies

```rust
pub enum FsRequest {
    Open { path: PathBuf, flags: OpenFlags },
    Read { handle: FileHandle, offset: u64, len: usize },
    Write { handle: FileHandle, offset: u64, shared_buf: Capability },
    Stat { path: PathBuf },
    // ...
}
```

- **Proposed native file system for this project:** a new design with copy-on-write and checksumming (in the spirit of ZFS/btrfs), written entirely in Rust (it is possible to start by drawing inspiration from existing crates such as the Redox project's RedoxFS, but for the MVP the endianness and on-disk format must be defined from scratch and documented)

### 2.3 Network Stack

- A full TCP/IP implementation in user-space (modeled on Fuchsia's Netstack or the smoltcp project in the Rust ecosystem)
- Communication with the network card happens through the Device Manager + a Capability restricted to that device
- **Decision: DPDK-style kernel-bypass support from the MVP itself** (not phase 2) — because both the AI profile (low-latency distributed networking between inference nodes) and the gaming profile (low-latency online networking) depend on it, and adding it late usually means redesigning the API. Mechanism: an application with a special Capability maps the NIC buffer directly (via `hal-direct`, layer 1) and sends data without going through the usual IPC/Netstack path.

```rust
pub trait KernelBypassNetworking {
    fn request_direct_nic_access(&self, token: CapabilityToken, nic: DeviceId)
        -> Result<DirectNicHandle, NetstackError>;
    fn poll_rx_ring(&self, handle: DirectNicHandle) -> &[RawPacket];
    fn submit_tx_ring(&self, handle: DirectNicHandle, packets: &[RawPacket]);
}
```

This path is entirely parallel to and independent of the normal Netstack path (above); ordinary applications continue to use the standard path (with the usual IPC overhead, but full security and isolation), and only apps with an explicit Capability from the Security Broker (layer 4) get access to this path.

### 2.4 Compositor Service — Where UI Fits in the Architecture

This answers the key question of "where does UI live in the architecture?" Decision: **a Compositor service in this same layer (shared infrastructure), as opposed to Window Managers/DEs, which live in layer 5 as native, user-selectable applications.**

```
┌─────────────────────────────────────────┐
│ Layer 5: DEs/WMs (GNOME-like, KDE-like,   │  ← the user chooses freely
│          tiling WM, or none for a server)  │
├─────────────────────────────────────────┤
│ Layer 3: Compositor Service (this section) │  ← shared infrastructure, always the same
└─────────────────────────────────────────┘
```

**Protocol decision:** a fully native, new protocol (not Wayland-compatible), because the architecture's core goal is carrying the Capability model all the way up to the UI layer. If we built on top of Wayland, its older permission model would inevitably leak underneath the new protocol. If Linux GTK/Qt software needs to be supported later, that happens through the **Linux Compat Runtime (layer 5)**, rather than polluting this infrastructure service with an external protocol.

```rust
pub trait DisplayProtocol {
    fn create_surface(&self, client: ProcessId) -> Result<SurfaceHandle, CompositorError>;
    fn commit_buffer(&self, surface: SurfaceHandle, buf: Capability); // zero-copy, buf comes from GPU memory
    fn destroy_surface(&self, surface: SurfaceHandle);
    fn input_event_stream(&self, client: ProcessId) -> EventStream;   // async, via Notification (layer 2)
    fn output_topology(&self) -> Vec<OutputInfo>;                      // multi-monitor, resolution, refresh rate
}
```

**Relationship to Compute Device Discovery (layer 1):** the Compositor talks directly to the Device Manager (Section 2.1 of this document) for GPU access and uses zero-copy shared memory (layer 2, Section 5.2) to transfer frames between an app and the screen — no extra copy, unlike older models such as X11.

**Relationship to Profile Policy (layer 4):** this service is a member of `services_enabled_by_default`, but **is not loaded by default in the AI profile (headless server)** — exactly per the Discovery+Policy model: the HAL still discovers the GPU (for AI compute use), it's just that the display service doesn't need to be loaded.

### 2.5 Memory Manager Service (High Level)

Note: this is different from the "raw memory management" in the microkernel (layer 2). This service is the policy-maker:
- Swapping/paging decisions (if the system has swap)
- OOM policy (which process is sacrificed under memory pressure) — configurable by Profile Policy in layer 4
- Unified Memory management for compute devices (coordinating the CPU/GPU/NPU memory pool, based on CXL capability as reported in the Hardware Manifest)

## 3. IPC Protocol Definition — The Contract Between Services

To avoid versioning confusion, all inter-service messages in this layer are defined with a **shared IDL** (proposal: a format similar to Cap'n Proto, or a lightweight custom IDL, rather than protobuf, to avoid serialization overhead on high-traffic paths). The IDL output generates type-safe Rust code used inside `kernel-ipc` (layer 2).

## 4. Directory Structure

```
subsystems/
 ├── root-task/            # the first process, distributes initial Capabilities
 ├── device-manager/
 ├── drivers/
 │    ├── driver-framework/    # shared DeviceDriver trait + sandbox runtime
 │    ├── driver-nvme/
 │    ├── driver-gpu-generic/
 │    ├── driver-npu-generic/
 │    └── ...
 ├── vfs-service/
 │    ├── vfs-router/
 │    ├── fs-native/           # the new native file system
 │    └── fs-compat-ext4/      # read/write-compatible only, for migrating data from Linux
 ├── netstack/
 └── mm-service/
```

## 5. MVP Definition of Done
1. The Root Task successfully brings up the Device Manager, a minimal VFS Service, and a simple block driver (e.g. virtio-blk on QEMU)
2. Deliberately crashing a driver (an injected panic in a test) → the Device Manager restarts it without affecting the rest of the system (this must be an automated CI test, not just a claim)
3. The VFS Service can mount a simple file system and perform basic read/write via IPC
4. Netstack can receive/send an ICMP echo (ping) packet on QEMU with virtio-net
4.1. Kernel-bypass path: a test application with a special Capability can send and receive packets directly on a virtual NIC's rx/tx ring (virtio-net with bypass support) without going through Netstack; its latency benchmark must be noticeably (at least 30-40%) lower than the standard path
4.2. Compositor Service: a test client can create a surface, commit a buffer, and have that buffer displayed on screen with no extra copy (zero-copy, verified with a profiling tool) (even a headless/file output is sufficient for the MVP)
5. Benchmark: VFS read/write throughput compared to a reference system (currently Linux on the same QEMU setup) must be reported — it must not degrade by more than 20-30% in the MVP (due to IPC overhead)

## 6. Finalized Decisions
- **`fs-compat-ext4`:** a one-time data migration tool from Linux systems, not a permanent part of the architecture. Once the native file system is stable, this module can be moved out to a standalone migration CLI (outside the OS core) so the VFS Router doesn't carry this complexity permanently.
- **DPDK-style kernel-bypass:** from the MVP itself (per Section 2.3 above).
- **Compositor/UI placement:** resolved per Section 2.4 above.

## 7. Remaining Open Questions
Nothing remains open in this layer for now; if a new decision arises during MVP implementation, it will be added here.
