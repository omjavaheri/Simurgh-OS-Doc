# 第 2 层 — 微内核（基于 Capability）

> **层级：** 第 2 层 - 微内核（基于 Capability）
> **状态：** 架构规范
> **实现语言：** Rust（直到 IPC 边界为止均为 `no_std`；本层任何部分都不得依赖无边界的堆分配器或普通的 panic-unwinding）
> **运行类型：** 内核空间，特权模式
> **设计灵感来源：** seL4（capability 模型与可证明性）、Zircon（IPC 模型与对象句柄）

---

## 0. 与上下层的关系

**下层（HAL，第 1 层）：** 根据第 1 层文档（第 0 节），从微内核一侧看，这个连接是**直接的函数/trait 调用**，而不是 IPC —— 这两层被链接进同一个特权二进制文件中。微内核在启动时通过 `HardwareManifestRaw`（第 1 层文档，第 9 节）完成初始化。

**上层（用户空间子系统，第 3 层）：** 通信通过**系统调用边界**进行 —— 这是整个架构中唯一真正发生硬件上下文切换（从 Ring 3/EL0/U-mode 切换到 Ring 0/EL1/M-mode）的地方：

```
第 3 层进程  --[syscall/svc/ecall instruction]-->  微内核（第 2 层）
第 3 层进程  <--[返回结果在寄存器中]-------------  微内核（第 2 层）
```

系统调用层（第 6 节，`SyscallOp`）是第 3 层及以上与微内核对话的唯一 API。两个第 3 层进程之间（例如一个应用与 VFS）的真实 IPC 也要经过同一条路径：双方都与微内核对话（`ipc_call`/`ipc_send_async`），微内核是消息/Capability 传递的中介，而不是两个进程直接互相访问。`kernel-ipc` crate（第 7 节）把这个边界封装成一个面向第 3 层 Rust 代码的更高层 API，使上层代码无需编写裸的 trap 指令。

微内核必须是**运行在特权模式下尽可能少的代码**。黄金法则：

> 如果一项功能能在用户空间实现，它就不属于内核。

本层只有四项职责，不多也不少：**内存管理、调度、IPC 和 Capability**。文件系统、驱动、网络，乃至传统意义上的「进程」本身，全都属于第 3 层（用户空间）。

## 1.1 决策：从设计第一天起就考虑可证明性（Formal Verification）

完整的形式化证明（seL4 风格）不在 MVP 的范围内，但**代码从一开始就在若干约束下编写，以便在第二阶段保持通向可证明性的道路是打开的**，而不是以后被迫重写：

- 禁止在没有文档化说明注释的情况下使用 `unsafe`（依照 HAL 文档第 7 节的规则；由于本层是特权层，这条规则在这里更加强制）
- 系统调用分发器必须写成一个明确的**状态机**；每个函数都必须有受限且可追踪的效果，不能有隐藏的副作用或通过分散的全局状态进行的 mutation
- Capability Derivation Tree 的结构（第 2 节）严格遵循 seL4 已被证明的模式，而不是自定义设计 —— 因为这一部分是整个系统的安全核心，从零重新设计的风险没有正当理由
- 安全关键函数（授予/撤销 Capability、retype 内存）必须以结构化注释的形式记录自己的前置/后置条件，即使在形式化验证工具（如 Kani 或 Prusti）引入项目之前也是如此 —— 这些注释以后会直接成为证明注解的基础

## 2. Capability 模型 —— 整个操作系统的安全核心

不同于 Linux 传统的 UID/GID/permission 模型，每一种资源（内存、设备、IPC 端口、HAL-Direct 令牌）都由一个 **Capability** 来表示：

```rust
/// 一个 Capability = 对某个 Kernel Object 的不可伪造引用 + 一组权限（rights）
pub struct Capability {
    pub object_ref: KernelObjectRef,
    pub rights: CapabilityRights,   // bitflags: READ, WRITE, EXECUTE, GRANT, DUPLICATE, REVOKE
    pub badge: u64,                 // 用于在 IPC 时区分请求的自定义标识符
}

bitflags::bitflags! {
    pub struct CapabilityRights: u32 {
        const READ     = 0b00001;
        const WRITE    = 0b00010;
        const EXECUTE  = 0b00100;
        const GRANT    = 0b01000; // 允许将这个 Capability 转交给另一个进程
        const DUPLICATE= 0b10000;
    }
}
```

**关键规则：**
- 默认情况下任何进程都不能访问任何资源；只有持有有效的 Capability 才能访问
- Capability 只能通过显式的 IPC（`grant`）传递，绝不隐式传递
- 撤销（Revocation）必须是可行的：内核必须能够使一个 Capability 及其全部衍生物（如通过 duplicate 产生的 children）失效（CDT 模式 —— Capability Derivation Tree，与 seL4 相同）

## 3. 内存管理（Memory Management）

微内核本身不会像通用 `malloc` 那样分配页面；相反，**物理内存本身就是一个 Capability**（`UntypedMemory` —— 与 seL4 的模型完全一致）：

- 启动时，Hardware Manifest（来自第 1 层）中报告的全部物理内存会以若干个 `UntypedMemory` 对象的形式交给第一个进程（第 3 层的 Root Task）
- 在服务之间划分这些内存的责任在于 Root Task，而不是内核本身 —— 这意味着**内存分配策略不在内核里**，内核只提供机制
- 内核在内存上的核心操作有：`retype`（把 `UntypedMemory` 转换为某个具体类型，如 `PageTable`、`ThreadControlBlock`、`Endpoint`）、`map`、`unmap`

```rust
pub enum KernelObjectType {
    UntypedMemory,
    PageTable,
    ThreadControlBlock,
    Endpoint,        // 用于 IPC
    Notification,    // 用于异步信号
    CapabilitySpace, // 一个进程的 Capability 表
}
```

## 4. 调度（Scheduler）

有两种模式，都由 HAL 的 Timer 支持提供支撑，与第 4 层配置文件的需求相对应：

| 模式 | 基础算法 | 使用场景 |
|---|---|---|
| Interactive | 基于优先级并带有 aging + 短时间片（约 1-4ms） | 通用场景、游戏 |
| Throughput/Batch | 自定义算法（第 4.1 节） | AI 批处理、专业场景 |

### 4.1 决策：为 Throughput 模式设计自定义算法

不直接照搬 CFS 或 EEVDF，而是从零设计一个专门针对我们双模型（dual-mode）优化的专用算法（而不是一个通用算法以后再打补丁）。初步设计要点：

- **调度单位不只是单个 thread，而是「IPC 群组」**：因为在这个架构中，一次操作通常是跨多个进程的一条 IPC 链（例如 应用 → VFS → 驱动），算法必须考虑整条链的成本，而不只是单个孤立的线程 —— 这正是 CFS 和 EEVDF 在原生设计上都没有考虑到的（因为在单体式 Linux 内核中，这样的链条根本不存在）。
- 把 **NUMA 与计算设备亲和性（Compute Device Affinity）感知**作为算法的一等输入，而不是事后叠加的附属层（这一点在 Linux 的 EEVDF 中也是后来才艰难加入的）
- **权重选择标准**：静态优先级（来自 Profile Policy）+ 在队列中的等待时长（类似 aging）+ 上面提到的「IPC 链成本」系数的组合
- 这条路径（从零设计）的风险必须通过在相同 workload 上对照 CFS-like 参考实现的重度 benchmark 来管理 —— 如果自定义算法在实践中并不比简单的 CFS 更好，就必须毫不犹豫地退回 CFS-like 方案；这个决策必须始终由数据驱动，而不是出于理念坚持

### 4.3 最终决策：Throughput 模式下的 Weight 公式

**基础公式（第一版；其数值参数会通过真实 benchmark 调整，但算法本身的结构是固定的）：**

```
vruntime_next(thread) = vruntime_current(thread) + (actual_runtime_ns / effective_weight)

effective_weight = base_priority_weight
                  × (1 + aging_factor × min(wait_time_ms, aging_cap_ms))
                  × numa_locality_bonus
```

- **`base_priority_weight`**：来自 Profile Policy（第 4 层）；例如 AI 配置文件会给推理线程更高的基础值
- **`aging_factor` 与 `aging_cap_ms`**：用于防止饥饿（starvation）；线程在队列中等待越久，其有效权重就越高，但存在上限，以免破坏主要的优先级排序
- **`numa_locality_bonus`**：被调度到靠近其内存/计算设备的核心上的线程会获得奖励（即增量 vruntime 的降低），用以鼓励调度器维持 NUMA 局部性

**IPC 链核算模型：** 不是把一条 IPC 链（例如 应用 → VFS → 驱动）中的每个线程分别核算，而是让参与同一条同步 IPC 链的所有线程共享一个 **Chain Group ID**，`vruntime` 在群组层面累积，而不是逐个线程累积：

```rust
pub struct ChainGroup {
    pub id: ChainGroupId,
    pub member_threads: Vec<ThreadId>,
    pub group_vruntime: u64,
}
```

这意味着一条较长的 IPC 链会把它的调度成本在成员之间公平分摊，而不是因为涉及多个线程就拿到两三倍的 CPU 份额（在像原始 CFS 这样的简单模型中确实存在这个问题，因为 IPC 在其最初设计中根本没有被考虑进去）。

**数值参数的确定：** 用于开始 benchmark 的建议初始值为 `aging_factor = 0.02`、`aging_cap_ms = 50`、`numa_locality_bonus = 0.9`（对本地访问的增量 vruntime 降低 10%）。这些只是起点，不是最终数字 —— 精确调优是 MVP 性能测试阶段的一部分工作，而不是一个悬而未决的架构问题。

### 4.4 调度器一般说明
- 模式选择是在 **per-thread** 级别进行的，而不是全系统级别；同一台设备上，一个 AI 推理线程可以处于 throughput 模式，而其 UI 处于 interactive 模式
- 调度器必须能感知（来自 HAL 的）NUMA 拓扑信息，以降低跨节点内存访问的成本
- 优先级继承（Priority Inheritance）是强制性的（用于防止 IPC 链中的优先级反转）

## 5. IPC —— 微内核架构的核心

由于所有东西（驱动、文件系统、网络）都在第 3 层作为独立进程运行，**IPC 的性能直接决定了整个系统的性能**。这是最需要优化的关键部分。

### 5.1 两种 IPC 类型

```rust
// 1. 同步、小消息 —— 用于类似 syscall 的快速 RPC 调用
pub struct Endpoint;
pub fn ipc_call(endpoint: Capability, msg: SmallMessage) -> SmallMessage;
// SmallMessage：最多几个 word，直接在寄存器中传递，无内存拷贝

// 2. 异步、大数据量 —— 用于传输大量数据（例如 AI/GPU 缓冲区）
pub struct Notification;
pub fn ipc_send_async(notif: Capability, shared_buffer: Capability);
```

### 5.2 面向大数据量的 Zero-Copy
对于大缓冲区（例如视频帧、AI 张量），不进行拷贝，而是传递一个**指向共享内存区域（Shared Memory Region）的 Capability**；接收方只需将其映射到自己的地址空间即可。拷贝次数为零。

```rust
pub fn create_shared_region(size: usize, rights: CapabilityRights) -> Capability;
pub fn map_shared_region(cap: Capability, at: Option<VirtAddr>) -> VirtAddr;
```

### 5.3 IPC 快速路径（Fast Path）
对于高流量的 Endpoint（例如调度器与某个实时驱动之间），必须存在一条能把上下文切换降到最低的「fast path」—— 这正是 L4 那个著名的优化，它把 IPC 成本从约几微秒降低到约几百纳秒。这一点必须经过 benchmark 验证，并且是验收标准的一部分（第 8 节）。

## 6. Kernel Object 模型

```rust
pub enum SyscallOp {
    Send { endpoint: CapId, msg: SmallMessage },
    Recv { endpoint: CapId },
    Call { endpoint: CapId, msg: SmallMessage },   // 原子性的 Send + Recv
    Yield,
    CapGrant { target_thread: CapId, cap: CapId, rights: CapabilityRights },
    CapRevoke { cap: CapId },
    Retype { untyped: CapId, target_type: KernelObjectType, count: usize },
    Map { page_table: CapId, frame: CapId, vaddr: VirtAddr, perms: MapPermissions },
}
```

系统调用层必须保持**非常小**（seL4 大约有 10-15 个系统调用；我们的目标与之类似，而不是接近 Linux 的 350-400 个系统调用）。

## 7. 目录结构

```
kernel/
 ├── kernel-core/        # 系统调用分发器、object model、no_std、架构无关
 ├── kernel-mm/          # UntypedMemory 管理、PageTable 操作
 ├── kernel-sched/       # 调度器，Interactive + Throughput 模式
 ├── kernel-ipc/         # Endpoint、Notification、Shared Region、fast path
 ├── kernel-cap/         # Capability 模型、CDT、撤销
 └── kernel-arch-glue/   # kernel-core 与 hal-core（第 1 层）trait 之间的桥接
```

## 8. MVP 验收标准
1. 在三种架构上均能启动，由 HAL 移交控制权，并创建出第一批 `UntypedMemory` 对象
2. 一个最小化的 Root Task 成功运行，能创建第二个线程并与之建立同步 IPC
3. Benchmark：在参考硬件上，fast path 上的 `ipc_call` 成本低于 500 纳秒（初始目标；可在真实 benchmark 之后调整）
4. 两个进程之间的 zero-copy 共享内存，并有测试证明没有发生任何数据拷贝
5. 撤销（Revocation）测试：使一个 Capability 失效，并证明其衍生物也不再有效
6. 对系统调用层进行模糊测试（Fuzz testing）（因为这一层是整个系统主要的攻击面）

## 9. 剩余的开放问题
本层没有遗留任何开放的架构问题。Weight 公式（第 4.3 节）数值常量的精确调优，自然是实现阶段 benchmark 工作的一部分，而不是一个悬而未决的设计决策。
