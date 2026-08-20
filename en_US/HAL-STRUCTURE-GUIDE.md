# Simurgh OS - Hardware Abstraction Layer (HAL)
## Directory Structure & File Descriptions

> **Layer:** 1 - Hardware Abstraction Layer  
> **Status:** Initial Workspace Setup  
> **Reference:** [01-HAL-Layer.md](./01-HAL-Layer.md)

---

## 📁 Overview

This directory contains the complete HAL implementation for Simurgh OS, following the design principles outlined in the architecture document. The HAL is the **only layer** that directly interacts with hardware (registers, MMIO, CPU instructions) and provides a unified interface to the microkernel (Layer 2) and higher layers.

### Key Design Principles
- ✅ **No OS dependencies** - All crates use `#![no_std]`
- ✅ **Architecture agnostic** - No `#[cfg(target_arch)]` above Layer 1
- ✅ **Hardware discovery always** - Full hardware enumeration regardless of profile
- ✅ **Capability-gated direct access** - `hal-direct` requires valid tokens
- ✅ **Fixed-size manifest** - `#[repr(C)]` for early boot handoff

---

## 🗂️ Directory Tree
```
os-project/
├── Cargo.toml # Root workspace configuration
├── rust-toolchain.toml # Rust toolchain version (nightly)
├── .cargo/
│ └── config.toml # Cargo build configuration
├── targets/
│ ├── x86_64-hal.json # Target spec for x86_64 (no_std)
│ ├── aarch64-hal.json # Target spec for ARM64 (no_std)
│ └── riscv64gc-hal.json # Target spec for RISC-V (no_std)
├── hal/
│ ├── hal-manifest/ # Hardware Manifest data structures
│ │ ├── Cargo.toml
│ │ └── src/
│ │ ├── lib.rs # Public API for manifest
│ │ └── raw.rs # #[repr(C)] structures for boot handoff
│ ├── hal-core/ # Core HAL traits (mandatory, always active)
│ │ ├── Cargo.toml
│ │ └── src/
│ │ ├── lib.rs # Re-exports all traits
│ │ ├── error.rs # HalError type definitions
│ │ ├── cpu.rs # CpuAbstraction trait
│ │ ├── memory.rs # MemoryBootstrap trait
│ │ ├── timer.rs # TimerAbstraction trait
│ │ ├── interrupt.rs # InterruptController trait
│ │ ├── compute.rs # ComputeDeviceDiscovery trait
│ │ ├── power.rs # Power & Thermal query interface
│ │ └── boot.rs # Boot info structures
│ ├── hal-direct/ # Capability-gated direct hardware access
│ │ ├── Cargo.toml
│ │ └── src/
│ │ └── lib.rs # HalDirectAccess trait + implementation
│ ├── hal-x86_64/ # x86_64 architecture implementation
│ │ ├── Cargo.toml
│ │ ├── build.rs # Build script (linker config, etc.)
│ │ └── src/
│ │ ├── lib.rs # Entry point, exports
│ │ ├── boot.S # Early bootstrap (assembly)
│ │ ├── linker.ld # Linker script for kernel layout
│ │ ├── cpu.rs # CPU abstraction implementation
│ │ ├── memory.rs # Memory bootstrap implementation
│ │ ├── timer.rs # Timer implementation (TSC/HPET)
│ │ └── interrupt.rs # Interrupt controller (APIC/x2APIC)
│ ├── hal-arm64/ # ARM64 (AArch64) architecture implementation
│ │ ├── Cargo.toml
│ │ ├── build.rs
│ │ └── src/
│ │ ├── lib.rs
│ │ ├── boot.S # Early bootstrap
│ │ ├── linker.ld
│ │ ├── cpu.rs # CPU abstraction (EL0-EL3)
│ │ ├── memory.rs # Memory bootstrap (ACPI/DT)
│ │ ├── timer.rs # Generic Timer implementation
│ │ └── interrupt.rs # GIC v3/v4 implementation
│ └── hal-riscv64/ # RISC-V (RV64GC) architecture implementation
│ ├── Cargo.toml
│ ├── build.rs
│ └── src/
│ ├── lib.rs
│ ├── boot.S # Early bootstrap (SBI)
│ ├── linker.ld
│ ├── cpu.rs # CPU abstraction (M/S/U modes)
│ ├── memory.rs # Memory bootstrap (Device Tree)
│ ├── timer.rs # mtime/mtimecmp implementation
│ └── interrupt.rs # PLIC + CLIC implementation
└── kernel-stub/ # Temporary microkernel stub for testing
├── Cargo.toml
└── src/
└── main.rs # "Hello from kernel" - MVP validation
```


---

## 📄 File Descriptions

### Root Workspace Files

| File | Purpose |
|------|---------|
| `Cargo.toml` | Root workspace with members: `hal-*`, `kernel-stub` |
| `rust-toolchain.toml` | Specifies nightly Rust with required components |
| `.cargo/config.toml` | Cargo settings (target, linker, runner for QEMU) |

### Target Specifications (`targets/`)

| File | Purpose |
|------|---------|
| `x86_64-hal.json` | Custom target for `x86_64-unknown-none` with `#![no_std]` |
| `aarch64-hal.json` | Custom target for `aarch64-unknown-none` |
| `riscv64gc-hal.json` | Custom target for `riscv64gc-unknown-none-elf` |

These define:
- CPU features (e.g., `mmx`, `sse`, `avx` for x86_64)
- Data layout and alignment
- Panic strategy (`abort`)
- Code model (`kernel`)

---

### HAL Crates

#### 1. `hal-manifest` - Hardware Manifest Structures
**Purpose:** Defines the boot-time hardware handoff data.

**Key Files:**
- `raw.rs` - Contains `HardwareManifestRaw` with `#[repr(C)]` layout
  - Fixed-size arrays (64 memory regions, 32 compute devices, 16 power domains)
  - No heap allocation, no `serde`
  - Used only during boot handoff (HAL → Microkernel)

**Dependencies:** `core` only (no `alloc`)

---

#### 2. `hal-core` - Core HAL Traits
**Purpose:** Defines the unified hardware abstraction interface.

**Key Files:**
| File | Trait | Responsibilities |
|------|-------|------------------|
| `cpu.rs` | `CpuAbstraction` | Core count, current core ID, feature flags, context switch, privilege levels |
| `memory.rs` | `MemoryBootstrap` | Physical memory map, identity mapping, IOMMU detection |
| `timer.rs` | `TimerAbstraction` | Current time (ns), oneshot/tickless modes |
| `interrupt.rs` | `InterruptController` | `register_irq`, `mask_irq`, `unmask_irq`, `send_ipi` |
| `compute.rs` | `ComputeDeviceDiscovery` | Enumerate GPU/NPU/TPU/FPGA devices |
| `power.rs` | `PowerManagement` | DVFS query/set, thermal readings |
| `boot.rs` | `BootInfo` | Standardized boot handoff structure |
| `error.rs` | `HalError` | Unified error type |

**Dependencies:** `core` only

---

#### 3. `hal-direct` - Capability-Gated Direct Access
**Purpose:** Provides raw hardware access for professional users.

**Key Features:**
- All functions require a valid `CapabilityToken`
- Tokens issued by Security Broker (Layer 4)
- HAL only verifies token validity (signature/scope)
- Available operations:
  - `map_mmio_region` - Direct MMIO mapping
  - `read_performance_counter` - Read perf counters
  - `pin_thread_to_core` - CPU affinity
  - `set_numa_policy` - NUMA memory policy

**Dependencies:** `hal-core`, `hal-manifest`

---

#### 4. Architecture-Specific Crates (`hal-x86_64`, `hal-arm64`, `hal-riscv64`)

**Purpose:** Implements all HAL traits for each architecture.

**Common Files:**
| File | Purpose |
|------|---------|
| `lib.rs` | Entry point, exports trait implementations |
| `boot.S` | Early bootstrap assembly (before Rust is initialized) |
| `linker.ld` | Linker script defining memory layout (kernel base, stack, etc.) |
| `cpu.rs` | Architecture-specific CPU abstraction |
| `memory.rs` | Memory bootstrap (e820/UEFI for x86, ACPI/DT for ARM, SBI/DT for RISC-V) |
| `timer.rs` | Timer implementation (TSC/HPET, Generic Timer, mtime) |
| `interrupt.rs` | Interrupt controller (APIC, GIC, PLIC) |

**Key Implementation Details:**

| Architecture | Timer | Interrupt Controller | Boot Source |
|--------------|-------|---------------------|-------------|
| **x86_64** | TSC/HPET | APIC/x2APIC | UEFI Memory Map / e820 |
| **ARM64** | Generic Timer | GIC v3/v4 | ACPI (priority) or Device Tree |
| **RISC-V** | mtime/mtimecmp | PLIC + CLIC | SBI + Device Tree (mandatory) |

---

#### 5. `kernel-stub` - MVP Test Stub
**Purpose:** Temporary microkernel stub for validating HAL correctness.

**Responsibilities:**
- Receives `HardwareManifestRaw` from HAL
- Prints "hello from kernel" on serial
- Validates that hardware detection works
- Minimal implementation - will be replaced by actual microkernel (Layer 2)

**Current Status:** ✅ Structure created, awaiting implementation

---

## 🔧 Build Configuration

### Target Files Reference

**x86_64-hal.json (excerpt):**
```json
{
  "llvm-target": "x86_64-unknown-none",
  "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128",
  "arch": "x86_64",
  "features": "+mmx,+sse,+sse2,+sse3,+ssse3,+sse4.1,+sse4.2,+avx,+avx2",
  "panic-strategy": "abort"
}
```
**aarch64-hal.json (excerpt):**
```json
{
  "llvm-target": "aarch64-unknown-none",
  "data-layout": "e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128",
  "arch": "aarch64",
  "features": "+v8a,+sve",
  "panic-strategy": "abort"
}
```

**riscv64gc-hal.json (excerpt):**
```json
{
  "llvm-target": "riscv64gc-unknown-none-elf",
  "data-layout": "e-m:e-p:64:64-i64:64-i128:128-n64-S128",
  "arch": "riscv64",
  "features": "+m,+a,+f,+d,+c",
  "panic-strategy": "abort"
}
```

