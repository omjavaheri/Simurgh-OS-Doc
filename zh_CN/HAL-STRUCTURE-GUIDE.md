# Simurgh OS - 硬件抽象层 (HAL)
## 目录结构与文件说明

> **层级：** 第 1 层 - 硬件抽象层  
> **状态：** 初始工作区设置  
> **参考：** [01-HAL-Layer.md](./01-HAL-Layer.md)

---

## 📁 概述

该目录包含 Simurgh OS 的完整 HAL 实现，遵循架构文档中概述的设计原则。HAL 是**唯一**直接与硬件（寄存器、MMIO、CPU 指令）交互的层，并为微内核（第 2 层）及更高层提供统一接口。

### 关键设计原则
- ✅ **无操作系统依赖** - 所有 crate 使用 `#![no_std]`
- ✅ **架构无关** - 第 1 层以上无 `#[cfg(target_arch)]`
- ✅ **始终进行硬件发现** - 无论配置文件如何，都进行完整硬件枚举
- ✅ **能力门控直接访问** - `hal-direct` 需要有效令牌
- ✅ **固定大小清单** - `#[repr(C)]` 用于早期启动移交

---

## 🗂️ 目录树

```
os-project/
├── Cargo.toml                    # 根工作区配置
├── rust-toolchain.toml           # Rust 工具链版本（nightly）
├── .cargo/
│   └── config.toml               # Cargo 构建配置
├── targets/
│   ├── x86_64-hal.json           # x86_64 目标规格（no_std）
│   ├── aarch64-hal.json          # ARM64 目标规格（no_std）
│   └── riscv64gc-hal.json        # RISC-V 目标规格（no_std）
├── hal/
│   ├── hal-manifest/             # 硬件清单数据结构
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs            # 清单公共 API
│   │       └── raw.rs            # 启动移交的 #[repr(C)] 结构
│   ├── hal-core/                 # 核心 HAL 特征（必需，始终激活）
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs            # 重新导出所有特征
│   │       ├── error.rs          # HalError 类型定义
│   │       ├── cpu.rs            # CpuAbstraction 特征
│   │       ├── memory.rs         # MemoryBootstrap 特征
│   │       ├── timer.rs          # TimerAbstraction 特征
│   │       ├── interrupt.rs      # InterruptController 特征
│   │       ├── compute.rs        # ComputeDeviceDiscovery 特征
│   │       ├── power.rs          # 电源与热量查询接口
│   │       └── boot.rs           # 启动信息结构
│   ├── hal-direct/               # 能力门控直接硬件访问
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs            # HalDirectAccess 特征 + 实现
│   ├── hal-x86_64/               # x86_64 架构实现
│   │   ├── Cargo.toml
│   │   ├── build.rs              # 构建脚本（链接器配置等）
│   │   └── src/
│   │       ├── lib.rs            # 入口点，导出
│   │       ├── boot.S            # 早期引导（汇编）
│   │       ├── linker.ld         # 内核布局链接脚本
│   │       ├── cpu.rs            # CPU 抽象实现
│   │       ├── memory.rs         # 内存引导实现
│   │       ├── timer.rs          # 定时器实现（TSC/HPET）
│   │       └── interrupt.rs      # 中断控制器（APIC/x2APIC）
│   ├── hal-arm64/                # ARM64（AArch64）架构实现
│   │   ├── Cargo.toml
│   │   ├── build.rs
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── boot.S            # 早期引导
│   │       ├── linker.ld
│   │       ├── cpu.rs            # CPU 抽象（EL0-EL3）
│   │       ├── memory.rs         # 内存引导（ACPI/DT）
│   │       ├── timer.rs          # 通用定时器实现
│   │       └── interrupt.rs      # GIC v3/v4 实现
│   └── hal-riscv64/              # RISC-V（RV64GC）架构实现
│       ├── Cargo.toml
│       ├── build.rs
│       └── src/
│           ├── lib.rs
│           ├── boot.S            # 早期引导（SBI）
│           ├── linker.ld
│           ├── cpu.rs            # CPU 抽象（M/S/U 模式）
│           ├── memory.rs         # 内存引导（设备树）
│           ├── timer.rs          # mtime/mtimecmp 实现
│           └── interrupt.rs      # PLIC + CLIC 实现
└── kernel-stub/                  # 用于测试的临时微内核存根
    ├── Cargo.toml
    └── src/
        └── main.rs               # “来自内核的问候” - MVP 验证
```

---

## 📄 文件说明

### 根工作区文件

| 文件 | 用途 |
|------|---------|
| `Cargo.toml` | 包含成员 `hal-*`、`kernel-stub` 的根工作区 |
| `rust-toolchain.toml` | 指定带有必需组件的 nightly Rust |
| `.cargo/config.toml` | Cargo 设置（目标、链接器、QEMU 运行器） |

### 目标规格 (`targets/`)

| 文件 | 用途 |
|------|---------|
| `x86_64-hal.json` | `x86_64-unknown-none` 的自定义目标，带 `#![no_std]` |
| `aarch64-hal.json` | `aarch64-unknown-none` 的自定义目标 |
| `riscv64gc-hal.json` | `riscv64gc-unknown-none-elf` 的自定义目标 |

这些定义：
- CPU 特性（例如 x86_64 的 `mmx`、`sse`、`avx`）
- 数据布局和对齐
- 恐慌策略（`abort`）
- 代码模型（`kernel`）

---

### HAL Crates

#### 1. `hal-manifest` - 硬件清单结构
**用途：** 定义启动时硬件移交数据。

**关键文件：**
- `raw.rs` - 包含带有 `#[repr(C)]` 布局的 `HardwareManifestRaw`
  - 固定大小数组（64 个内存区域，32 个计算设备，16 个电源域）
  - 无堆分配，无 `serde`
  - 仅在启动移交期间使用（HAL → 微内核）

**依赖：** 仅 `core`（无 `alloc`）

---

#### 2. `hal-core` - 核心 HAL 特征
**用途：** 定义统一的硬件抽象接口。

**关键文件：**

| 文件 | 特征 | 职责 |
|------|-------|------------------|
| `cpu.rs` | `CpuAbstraction` | 核心数、当前核心 ID、特性标志、上下文切换、特权级别 |
| `memory.rs` | `MemoryBootstrap` | 物理内存映射、恒等映射、IOMMU 检测 |
| `timer.rs` | `TimerAbstraction` | 当前时间（纳秒）、单次/无滴答模式 |
| `interrupt.rs` | `InterruptController` | `register_irq`、`mask_irq`、`unmask_irq`、`send_ipi` |
| `compute.rs` | `ComputeDeviceDiscovery` | 枚举 GPU/NPU/TPU/FPGA 设备 |
| `power.rs` | `PowerManagement` | DVFS 查询/设置、温度读取 |
| `boot.rs` | `BootInfo` | 标准化启动移交结构 |
| `error.rs` | `HalError` | 统一错误类型 |

**依赖：** 仅 `core`

---

#### 3. `hal-direct` - 能力门控直接访问
**用途：** 为专业用户提供原始硬件访问。

**主要特性：**
- 所有函数都需要有效的 `CapabilityToken`
- 令牌由安全代理（第 4 层）签发
- HAL 仅验证令牌有效性（签名/范围）
- 可用操作：
  - `map_mmio_region` - 直接 MMIO 映射
  - `read_performance_counter` - 读取性能计数器
  - `pin_thread_to_core` - CPU 亲和性
  - `set_numa_policy` - NUMA 内存策略

**依赖：** `hal-core`、`hal-manifest`

---

#### 4. 架构特定 Crates（`hal-x86_64`、`hal-arm64`、`hal-riscv64`）

**用途：** 为每个架构实现所有 HAL 特征。

**公共文件：**

| 文件 | 用途 |
|------|---------|
| `lib.rs` | 入口点，导出特征实现 |
| `boot.S` | 早期引导汇编（在 Rust 初始化之前） |
| `linker.ld` | 定义内存布局的链接器脚本（内核基址、栈等） |
| `cpu.rs` | 架构特定的 CPU 抽象 |
| `memory.rs` | 内存引导（x86 的 e820/UEFI，ARM 的 ACPI/DT，RISC-V 的 SBI/DT） |
| `timer.rs` | 定时器实现（TSC/HPET、通用定时器、mtime） |
| `interrupt.rs` | 中断控制器（APIC、GIC、PLIC） |

**关键实现细节：**

| 架构 | 定时器 | 中断控制器 | 引导源 |
|--------------|-------|---------------------|-------------|
| **x86_64** | TSC/HPET | APIC/x2APIC | UEFI 内存映射 / e820 |
| **ARM64** | 通用定时器 | GIC v3/v4 | ACPI（优先）或设备树 |
| **RISC-V** | mtime/mtimecmp | PLIC + CLIC | SBI + 设备树（必需） |

---

#### 5. `kernel-stub` - MVP 测试存根
**用途：** 用于验证 HAL 正确性的临时微内核存根。

**职责：**
- 从 HAL 接收 `HardwareManifestRaw`
- 在串行端口上打印“来自内核的问候”
- 验证硬件检测是否正常工作
- 最小实现 - 将被实际微内核（第 2 层）替换

**当前状态：** ✅ 结构已创建，等待实现

---

## 🔧 构建配置

### 目标文件参考

**x86_64-hal.json（摘录）：**
```json
{
  "llvm-target": "x86_64-unknown-none",
  "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128",
  "arch": "x86_64",
  "features": "+mmx,+sse,+sse2,+sse3,+ssse3,+sse4.1,+sse4.2,+avx,+avx2",
  "panic-strategy": "abort"
}
```

**aarch64-hal.json（摘录）：**
```json
{
  "llvm-target": "aarch64-unknown-none",
  "data-layout": "e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128",
  "arch": "aarch64",
  "features": "+v8a,+sve",
  "panic-strategy": "abort"
}
```

**riscv64gc-hal.json（摘录）：**
```json
{
  "llvm-target": "riscv64gc-unknown-none-elf",
  "data-layout": "e-m:e-p:64:64-i64:64-i128:128-n64-S128",
  "arch": "riscv64",
  "features": "+m,+a,+f,+d,+c",
  "panic-strategy": "abort"
}
```
