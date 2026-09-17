# 第 3 层 — 内核子系统（用户空间服务）

> **层级：** 第 3 层 - 内核子系统（用户空间服务）
> **状态：** 架构规范
> **实现语言：** 除非另有明确说明，否则一律使用 Rust
> **运行类型：** 用户空间，但调度优先级很高（接近内核优先级），每个服务都是独立、隔离的进程，拥有受限的 Capability

---

## 0. 与上下层的关系

**下层（微内核，第 2 层）：** 本层的每个服务（Device Manager、VFS、Netstack、Compositor、mm-service）都是一个普通的用户空间进程，只通过 **系统调用 + Capability** 与微内核对话（正是第 2 层文档第 0 节所描述的那条边界）。这些服务相对于一个普通的第 5 层应用没有任何特权；唯一的区别在于 Root Task 在启动时授予了它们更多的 Capability（例如访问某个特定设备 MMIO 的 Capability）。

**上层（系统服务，第 4 层）：** 通信通过由共享 IDL（本文档第 3 节）定义的**高层 IPC 协议**进行 —— 而不是裸系统调用。举例：
- POSIX Compat Layer（第 4 层）在 `open()` 被调用时，内部会向 VFS Service 发送一个 `FsRequest::Open`
- Package Manager（第 4 层）在安装一个软件包时，会与 VFS Service 和 Device Manager（针对目标磁盘/设备空间）进行 IPC
- Security Broker（第 4 层）自己签发的 Capability（例如访问某个特定驱动的权限）最终还是要通过微内核（第 2 层）进行校验和传递，但「谁被允许」这一策略性决策是在第 4 层做出的

一个重要的架构要点：**第 4 层从不直接向微内核（第 2 层）发起系统调用**，除了非常有限的管理性操作（例如签发 Capability，这本身就需要微内核层面的访问）；主要的数据路径始终经过本层（第 3 层）的协议。

本层包含了所有「大家都以为是内核一部分」的东西，但实际上它们没有一个是特权的。本层的每一个组件都是微内核层面上的一个独立进程，只能访问明确授予它的那些 Capability。

**微内核运行的第一个进程被称为「Root Task」**，负责这些子系统的初始启动，以及初始 Capability 的分发（包括每个服务各自分到的那一份 UntypedMemory）。

## 2. 主要组成部分

### 2.1 Device Manager（驱动管理器）

```
device-manager/
 ├── 接收 Hardware Manifest（通过与 Root Task 的 IPC，Root Task 从 HAL 那里获得它）
 ├── 为每个驱动启动一个独立进程（Driver Process Isolation）
 ├── 向每个驱动签发受限的 Capability：
 │    - 仅限于该设备对应的 MMIO 区域（而非全部内存）
 │    - 仅限于该设备的 IRQ
 └── 重启策略：某个驱动崩溃 → Device Manager 会在一个新进程中
     将它重新拉起，而不会让整个系统随之宕掉（这是整个架构相对于单体式 Linux 内核最重要的成果）
```

**驱动编写模型：**
```rust
#[async_trait]
pub trait DeviceDriver {
    fn probe(manifest_entry: &ComputeDevice) -> Result<Self, DriverError> where Self: Sized;
    async fn handle_irq(&mut self, irq: IrqId);
    async fn handle_request(&mut self, req: DriverRequest) -> DriverResponse;
}
```
每个驱动都是一个独立的 Rust crate，实现这个 trait，并在一个独立的进程沙箱中编译和运行。

### 2.2 VFS Service（文件系统）

- 文件系统不再是内核的一部分（不同于 Linux），每种文件系统（ext4-compat、btrfs-compat、新的原生文件系统）都是位于 VFS Router 背后的一个**独立服务**
- 应用与 VFS 之间通过 IPC（第 2 层）、使用标准消息协议（`FsRequest`/`FsResponse`）通信
- 为了性能：页缓存（page cache）以 VFS Service 与 Kernel Memory Manager 之间的 Shared Memory Region 形式实现，以避免额外拷贝

```rust
pub enum FsRequest {
    Open { path: PathBuf, flags: OpenFlags },
    Read { handle: FileHandle, offset: u64, len: usize },
    Write { handle: FileHandle, offset: u64, shared_buf: Capability },
    Stat { path: PathBuf },
    // ...
}
```

- **为本项目提议的原生文件系统：** 一个带有 copy-on-write 和 checksumming 的全新设计（精神上类似 ZFS/btrfs），完全使用 Rust 编写（可以从现有 crate 中获取灵感开始，例如借鉴 Redox 项目的 RedoxFS，但对 MVP 而言，字节序（endianness）和磁盘上格式必须从零开始定义并写好文档）

### 2.3 Network Stack（网络栈）

- 在用户空间完整实现 TCP/IP（参照 Fuchsia 的 Netstack 或 Rust 生态中的 smoltcp 项目）
- 与网卡的通信通过 Device Manager + 限定于该设备的 Capability 进行
- **决策：从 MVP 本身开始就支持类似 DPDK 的 kernel-bypass**（而不是第二阶段）—— 因为 AI 配置文件（推理节点之间的低延迟分布式网络）和游戏配置文件（低延迟在线网络）都依赖于此，而后期再加入通常意味着要重新设计 API。机制：拥有特殊 Capability 的应用直接映射 NIC 缓冲区（通过 `hal-direct`，第 1 层），不经过常规的 IPC/Netstack 路径直接发送数据。

```rust
pub trait KernelBypassNetworking {
    fn request_direct_nic_access(&self, token: CapabilityToken, nic: DeviceId)
        -> Result<DirectNicHandle, NetstackError>;
    fn poll_rx_ring(&self, handle: DirectNicHandle) -> &[RawPacket];
    fn submit_tx_ring(&self, handle: DirectNicHandle, packets: &[RawPacket]);
}
```

这条路径与常规的 Netstack 路径（上文）完全并行且相互独立；普通应用依然使用标准路径（具有常规的 IPC 开销，但拥有完整的安全性和隔离性），只有那些从 Security Broker（第 4 层）获得明确 Capability 的应用才能访问这条路径。

### 2.4 Compositor Service —— UI 在架构中的位置

这回答了一个关键问题：「UI 在架构中处于什么位置？」决策是：**在本层（共享基础设施）设置一个 Compositor 服务，而 Window Manager/桌面环境（DE）则位于第 5 层，作为用户可自由选择的原生应用。**

```
┌─────────────────────────────────────────┐
│ 第 5 层：DE/WM（类 GNOME、类 KDE、      │  ← 用户自由选择
│          平铺式 WM，或服务器场景下无任何 WM） │
├─────────────────────────────────────────┤
│ 第 3 层：Compositor Service（本节）       │  ← 共享基础设施，始终一致
└─────────────────────────────────────────┘
```

**协议决策：** 采用完全原生、全新的协议（不兼容 Wayland），因为该架构的核心目标是把 Capability 模型一路带到 UI 最上层。如果我们建立在 Wayland 之上，它那套旧的 permission 模型将不可避免地渗透到新协议之下。如果以后需要支持 Linux 上的 GTK/Qt 软件，将通过 **Linux Compat Runtime（第 5 层）** 来实现，而不是让这个基础设施服务被一个外部协议污染。

```rust
pub trait DisplayProtocol {
    fn create_surface(&self, client: ProcessId) -> Result<SurfaceHandle, CompositorError>;
    fn commit_buffer(&self, surface: SurfaceHandle, buf: Capability); // zero-copy，buf 来自 GPU 内存
    fn destroy_surface(&self, surface: SurfaceHandle);
    fn input_event_stream(&self, client: ProcessId) -> EventStream;   // 异步，通过 Notification（第 2 层）
    fn output_topology(&self) -> Vec<OutputInfo>;                      // 多显示器、分辨率、刷新率
}
```

**与 Compute Device Discovery（第 1 层）的关系：** Compositor 直接与 Device Manager（本文档第 2.1 节）对话以获得 GPU 访问，并使用 zero-copy 共享内存（第 2 层，第 5.2 节）在应用与屏幕之间传输帧 —— 没有额外拷贝，不同于 X11 那样的旧模型。

**与 Profile Policy（第 4 层）的关系：** 这个服务是 `services_enabled_by_default` 的成员之一，但**在 AI 配置文件（无头服务器）下默认不会被加载** —— 这恰好符合 Discovery+Policy 模型：HAL 依然会发现 GPU（供 AI 计算使用），只是显示服务不需要被加载。

### 2.5 Memory Manager Service（高层）

注意：这与微内核（第 2 层）中的「原始内存管理」不同。这个服务是策略制定者：
- 决定 swapping/paging（如果系统启用了 swap）
- OOM 策略（内存不足时牺牲哪个进程）—— 可由第 4 层的 Profile Policy 配置
- 为计算设备管理 Unified Memory（协调 CPU/GPU/NPU 内存池，基于 Hardware Manifest 中报告的 CXL 能力）

## 3. IPC 协议定义 —— 服务之间的契约

为避免版本混乱，本层所有服务间消息都用**共享 IDL** 定义（提议：采用类似 Cap'n Proto 的格式，或一个轻量级的自定义 IDL，而不是 protobuf，以避免在高流量路径上产生序列化开销）。IDL 会生成类型安全的 Rust 代码，供 `kernel-ipc`（第 2 层）使用。

## 4. 目录结构

```
subsystems/
 ├── root-task/            # 第一个进程，分发初始 Capability
 ├── device-manager/
 ├── drivers/
 │    ├── driver-framework/    # 共享的 DeviceDriver trait + sandbox runtime
 │    ├── driver-nvme/
 │    ├── driver-gpu-generic/
 │    ├── driver-npu-generic/
 │    └── ...
 ├── vfs-service/
 │    ├── vfs-router/
 │    ├── fs-native/           # 新的原生文件系统
 │    └── fs-compat-ext4/      # 仅用于兼容读写，用于从 Linux 迁移数据
 ├── netstack/
 └── mm-service/
```

## 5. MVP 验收标准
1. Root Task 成功拉起 Device Manager、一个最小化的 VFS Service，以及一个简单的块设备驱动（例如 QEMU 上的 virtio-blk）
2. 蓄意让某个驱动崩溃（测试中注入的 panic）→ Device Manager 会将其重启，而不影响系统其余部分（这必须是 CI 中的自动化测试，而不仅仅是一句声明）
3. VFS Service 能够挂载一个简单的文件系统，并通过 IPC 完成基本的读写
4. Netstack 能在带 virtio-net 的 QEMU 上收发一个 ICMP echo（ping）包
4.1. kernel-bypass 路径：一个拥有特殊 Capability 的测试应用能够直接在某个虚拟 NIC（支持 bypass 的 virtio-net）的 rx/tx ring 上收发数据包，而不经过 Netstack；其延迟 benchmark 必须明显（至少 30-40%）低于标准路径
4.2. Compositor Service：一个测试客户端能够创建一个 surface、提交一个缓冲区，并且该缓冲区能在没有额外拷贝的情况下（zero-copy，经 profiling 工具验证）显示在屏幕上（对于 MVP 而言，即便是 headless/文件输出也足够）
5. Benchmark：必须报告 VFS 读写吞吐量相对于参考系统（目前是同一台 QEMU 上的 Linux）的对比结果 —— 在 MVP 中下降幅度不应超过 20-30%（由于 IPC 开销）

## 6. 已敲定的决策
- **`fs-compat-ext4`：** 这是一个从 Linux 系统一次性迁移数据的工具，不是架构的永久组成部分。在原生文件系统稳定之后，这个模块可以被迁移为一个独立的迁移用 CLI（在操作系统核心之外），从而不让 VFS Router 永久承担这份复杂度。
- **DPDK 风格的 kernel-bypass：** 从 MVP 本身就开始（依照上文第 2.3 节）。
- **Compositor/UI 的定位：** 依照上文第 2.4 节已经解决。

## 7. 剩余的开放问题
目前本层没有遗留任何开放事项；如果在 MVP 实现过程中出现新的决策，会在此处补充。
