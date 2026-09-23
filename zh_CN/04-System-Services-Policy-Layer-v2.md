# 第 4 层 — 系统服务 + Profile Policy Layer（第 2 版）

> **层级：** 第 4 层 - 系统服务 + Profile Policy Layer（第 2 版）
> **状态：** 架构规范 —— 第 2 版修订，基于对「普通/家庭用户」需求的分析。本版本取代 `04-System-Services-Policy-Layer.md` 中的相关章节，而非整份文档（见下文「本文档的状态」）。
> **实现语言：** 核心服务（Init、Simurgh Store、Security Broker、Policy Engine）使用 Rust；POSIX Compatibility Layer 的 ABI 层面使用 C（尽可能在其内部用 `#[no_mangle] extern "C"` 的 Rust 函数实现，即「Rust-backed libc」模式）；静态配置使用 TOML，Policy Engine 的动态逻辑使用 Rhai —— 与基础文档（`04-System-Services-Policy-Layer.md`）第 9 节相比没有变化
> **运行类型：** 用户空间，无需高级 Capability（Security Broker 除外）—— 与基础文档相比没有变化

---

**本文档的状态：** 基于对「普通/家庭用户」需求分析的一次重新设计。本版本取代 `04-System-Services-Policy-Layer.md` 中的相关章节，而非整份文档。Init 与 POSIX Compatibility Layer 部分从上一版本起保持不变、依然有效。基础文档第 6.1、6.2 节以及第 8 节中的一项（Profile Policy Layer 相关）也在本版本中被更新（见本文档第 7 节）。

---

## 与上一版本相比的变化（摘要）

| 章节 | 上一版本 | 新版本（本文档） |
|---|---|---|
| Package Ecosystem | 类似 Nixpkgs 的开放模式，用户端编译 | 集中式商店，只提供预编译并经过测试的软件包 |
| 商店以外的安装 | 未明确规定 | **禁止，没有例外** |
| 日志/错误 | 不存在 | 新增服务：Diagnostics & Telemetry |
| 多用户支持 | 未明确规定 | 新增服务：Account & Session Manager，带有 Elevation 模型 |
| 备份 | 不存在 | 新增服务：Backup Manager |
| 系统默认安装 | 不存在 | 新概念：Installation Manifest |
| Profile Policy —— 配置文件种类 | 4 种配置文件：General/Professional/Gaming/AI | 6 种配置文件：新增 Server 与 RealTime（软实时） |

---

## 1. 本层已更新的组成部分

```
第 4 层（第 2 版）
 ├── Init / Service Manager                （相对第 1 版没有变化）
 ├── POSIX Compatibility Layer             （相对第 1 版没有变化）
 ├── Simurgh Store                         ← 取代旧的 Package/Build Ecosystem
 ├── Diagnostics & Telemetry Manager       ← 新增
 ├── Security / Permission Broker          （扩展了 Elevation）
 ├── Account & Session Manager             ← 新增
 ├── Backup Manager                        ← 新增
 └── Profile Policy Layer                  （更新：新增 RealTime 配置文件）
```

---

## 2. Diagnostics & Telemetry Manager

### 2.1 目的
系统任何组件中（从微内核到某个 App）出现的每一个错误或异常行为，都必须被记录、分类，并（在征得用户同意后）发送给开发团队 —— 这样开发者无需向用户索取额外数据，就能修复 bug。

### 2.2 两部分架构

- **Log Collector**（第 3 层）：从系统所有组件收集原始日志的共享基础设施；与 VFS/Netstack/Compositor 并列。
- **Diagnostics Manager**（本层）：策略制定方 —— 决定发送什么内容、何时发送、以何种同意级别发送。

### 2.3 崩溃报告结构

```rust
pub struct CrashReport {
    // 精确标识发生在哪里、发生了什么
    pub component_id: String,        // 例如 "driver-printer-hp-laserjet"
    pub component_version: String,
    pub layer: LayerId,

    // 错误发生那一刻系统的完整状态
    pub os_version: String,
    pub hardware_manifest_snapshot: HardwareManifest,
    pub active_profile: ProfileKind,

    // 错误本身的技术细节
    pub severity: Severity,
    pub error_code: Option<u32>,
    pub stack_trace: Vec<StackFrame>,
    pub capability_state: Option<CapabilitySnapshot>,
    pub last_ipc_chain: Vec<IpcCallRecord>,
    pub memory_state: Option<MemoryDiagnostic>,

    // 用于复现问题，但不含个人数据
    pub preceding_actions: Vec<AnonymizedAction>,  // 例如 "file_open_attempt"
    pub timestamp: u64,
    pub anonymous_device_id: Uuid,
}
```

关键设计要点：报告中绝不包含任何文件名、真实路径或个人数据内容。取而代之的是，用匿名化形式保留最近 20-30 次操作的环形缓冲区（ring buffer），一旦发生崩溃就把它附加到报告中。

### 2.4 已敲定的决策

- **不可移除：** 本服务是基础安装的一部分，就像 File Manager 一样。
- **Admin 访问权限：** 可以（在本地）完整访问所有用户的日志。
- **发送方式：** 机会式发送 —— 任何已登录且能访问互联网的用户，都会在后台周期性地发送整个系统的新日志（而不仅仅是自己的日志）；是否整体发送是一个系统级的全局设置，而不是按用户设置的。

---

## 3. Simurgh Store（取代旧的 Package/Build Ecosystem）

### 3.1 总体原则
与上一版本中类似 Nixpkgs 的开放模式不同，所有内容都**集中化、预编译，并由 Simurgh 团队测试**。用户系统上从不进行任何编译。

### 3.2 软件包生产流水线（Simurgh 团队一侧）

```
软件源码（开源或闭源，Linux 版或 Windows 版）
   ↓
执行路径检测：Native | LinuxCompatTested | WindowsCompatTested
   ↓
编译/打包
   ↓
自动化（CI）测试：安装、运行、干净退出
   ↓
数字签名
   ↓
发布到商店
```

### 3.3 软件包结构

```rust
pub struct StorePackage {
    pub name: String,
    pub version: String,
    pub binary_hash: String,
    pub signature: DigitalSignature,
    pub execution_path: ExecutionPath,
    pub required_capabilities: Vec<CapabilityRequest>,
    pub category: PackageCategory,   // App | Driver | UI-Shell | System-Update
    pub tested_on_profiles: Vec<ProfileKind>,
    pub changelog: String,
}

pub enum ExecutionPath {
    Native,
    LinuxCompatTested,
    WindowsCompatTested,
}
```

### 3.4 已敲定的决策

- **不允许在商店以外安装：** 没有任何例外。Security Broker 会在每次 `install()` 之前验证软件包签名；没有 Simurgh 团队有效签名的软件包根本不会被安装。
- **驱动也在商店里：** HAL/Device Manager 只负责发现；驱动的安装/更新始终由用户在商店中主动选择，绝不自动进行。
- **更新：** 商店中有一个「更新」板块，涵盖操作系统、App、驱动、UI；只有用户点击后才会安装。
- **对闭源 Windows 软件的支持：** 支持，测试和提供方式与 Linux 软件相同（具体执行方式记录在第 5 层文档中，作为一个开放问题）。

### 3.5 按用户安装

每个用户独立安装/卸载（依照第 4 节）。为了避免浪费磁盘空间，实际的二进制文件保存在共享的 content-addressed storage 中；只有安装记录、Capability 和数据是按用户分开的：

```rust
pub struct AppInstallRecord {
    pub owner_uid: UserId,
    pub package_ref: ContentAddressedRef,
    pub granted_capabilities: Vec<Capability>,
    pub user_data_path: PathBuf,
}
```

---

## 4. Account & Session Manager（新增）

### 4.1 数据模型

```rust
pub struct UserAccount {
    pub uid: UserId,
    pub username: String,
    pub account_type: AccountType,       // Standard | Admin
    pub home_capability_root: Capability,
    pub installed_apps: Vec<AppInstallRef>,
}

pub enum AccountType {
    Standard,
    Admin,
}
```

### 4.2 隔离模型
每个用户都有一个独立的 Capability 空间（而不仅仅是基于 UID/GID 的 permission）。也就是说，即便一个 App 存在技术上的 bug，也没有办法访问到另一个用户的文件。

### 4.3 Admin 模型 —— 没有常驻权限（Privilege Elevation）

Admin 在正常情况下的行为和普通用户一样。对于每一次敏感操作（安装驱动、查看另一个用户的日志/文件、更改系统设置），系统每次都会要求重新输入密码/重新确认 —— 没有例外，即便是在同一次会话内第二次查看另一个用户的文件也是如此。

```rust
pub trait ElevationBroker {
    fn request_elevation(
        &self,
        admin_uid: UserId,
        reason: ElevationReason,
        target: CapabilityRequest,
    ) -> Result<TemporaryCapability, PermissionDenied>;
}
```

### 4.4 在用户之间切换 UI

```rust
pub trait SessionManager {
    fn switch_shell(&self, uid: UserId, new_shell: PackageRef) -> Result<(), SessionError>;
}
```
既可以从 Login Screen 完成，也可以从正在运行的 UI 内部的设置中完成。

---

## 5. Backup Manager（新增）

### 5.1 设计原则
因为已安装的 App 只是指向某个商店软件包的引用，所以备份不需要复制应用程序的完整二进制文件 —— 只需要记录「这个用户曾经装过哪些 App」，在恢复时会重新从商店下载。

### 5.2 结构

```rust
pub struct BackupManifest {
    pub owner_uid: UserId,
    pub created_at: u64,
    pub source_device_id: Uuid,
    pub included_categories: BackupSelection,
    pub installed_apps: Vec<PackageRef>,
    pub user_settings: SettingsSnapshot,
    pub user_data_archive: ContentAddressedRef,
}

pub struct BackupSelection {
    pub include_documents: bool,
    pub include_pictures_videos: bool,
    pub include_app_settings: bool,
    pub include_desktop_customization: bool,
    pub excluded_paths: Vec<PathBuf>,
}
```

### 5.3 调度

```rust
pub struct BackupSchedule {
    pub enabled: bool,
    pub frequency: BackupFrequency,   // Daily | Weekly | Monthly
    pub preferred_time: TimeOfDay,
}

pub trait BackupManager {
    fn schedule_backup(&mut self, uid: UserId, schedule: BackupSchedule);
    fn backup_now(&self, uid: UserId, destination: BackupDestination) -> Result<BackupHandle, BackupError>;
}
```

### 5.4 存储目的地 —— 安全要求（3-2-1 原则）

```rust
pub enum BackupDestination {
    ExternalDrive { device_id: Uuid },
    NetworkLocation { address: String },
    // 故意没有「系统主磁盘」这个选项
}
```

### 5.5 恢复

可以在同一台设备上执行，也可以在任何其他 Simurgh 设备上执行：从商店自动安装 App（通过 Installation Manifest 机制，见第 6 节）+ 恢复数据 + 应用设置。Admin 可以为每个用户单独设置/执行备份。

---

## 6. Installation Manifest（新概念）

一个在安装前准备好的文件，用来指定安装程序应该从商店自动安装哪些内容 —— 把「默认应该是什么」这一决定从系统代码中分离出来。

```toml
# install-manifest.toml
[system]
default_ui = "simurgh-shell-minimal"
default_apps = [
    "file-manager",
    "browser-default",
    "text-editor",
    "settings-app"
]

[profile]
kind = "General"
```

流程：HAL discovery + 微内核 + 关键子系统被安装 → Init/Installer 读取该文件 → Package Manager 连接到商店，并自动下载/安装文件中列出的每一项 → 首次启动时，默认 UI 与默认 App 已经就绪。

它可以有不同的变体（家庭安装、企业安装，或高级用户的自定义安装），类似于 Linux 上的 Kickstart 或 Windows 上的 Unattend。

---

## 7. Profile Policy Layer 更新

本节取代基础文档 `04-System-Services-Policy-Layer.md` 中的第 6.1、6.2 节，以及第 8 节中的第 6 项。

### 7.1 对第 6.1 节的更新 —— 数据模型

```rust
pub enum ProfileKind {
    General,
    Professional,
    Server,
    RealTime,      // 软实时 / 延迟优先 —— 不是具有确定性 WCET 保证的硬实时（hard-RT）
    Gaming,
    AI,
}
```

### 7.2 对第 6.2 节的更新 —— 六种配置文件对照表

| 设置项 | 通用 | 专业 | 服务器 | 实时（软实时） | 游戏 | AI |
|---|---|---|---|---|---|---|
| 调度器默认模式 | Interactive | Interactive | Throughput | Interactive（无/低 aging） | Interactive（输入延迟优先） | Throughput |
| Power Policy | Balanced | Balanced | Efficiency | Performance | Performance | Performance（更精细的温控） |
| `hal-direct` 默认访问权限 | 否 | 是 | 否 | 是（定时器/perf counter/IRQ pinning） | 否（仅特定 GPU 调优） | 部分（device memory） |
| Compositor | 已安装，激活 | 已安装，激活 | 不加载 | 已安装，激活（如有需要） | 已安装，激活 | 已安装，未激活 |
| AI runtime 服务 | 已安装，未激活 | 已安装，未激活 | 已安装，未激活 | 已安装，未激活 | 已安装，未激活 | 激活，预加载 |
| 默认 Unified Memory | 否 | 可选 | 否 | 否 | 否 | 是 |
| NUMA-aware 调度 | 否 | 是 | 是 | 是 | 是（可选） | 是 |
| lazy-load 阈值 | 默认（50MB） | 默认 | 默认 | 更低（更少非必要服务被加载） | 默认 | 按第 9 节覆盖 |

### 7.3 架构说明 —— 关于「实时（RealTime）」配置文件的一个关键点

这个配置文件**不会**在微内核中创建新的调度模式 —— 依照 `02-Microkernel-Layer.md` 第 4.4 节，模式选择（Interactive/Throughput）本来就是按线程（per-thread）设定的，而不是全系统统一设定的。「RealTime」配置文件只是为了获得更可预测的延迟，调整了**策略层面的默认值**：

- 更小或为零的 `aging_cap_ms`（防止 aging 引起的优先级波动）
- 对 `hal-direct` 更开放的默认访问权限，用于高精度定时器以及把线程 pin 到某个核心上
- 默认将非必要服务设为 lazy 加载或直接关闭，以减少来自相互竞争的 IPC 所导致的抖动（jitter）

**这个配置文件并不等同于硬实时（hard real-time，即像工业/航空电子安全关键系统那样具有确定性截止时限保证的实时）。** 依照 `02-Microkernel-Layer.md` 第 1.1 节，完整的形式化验证（硬实时真正的前提条件）不在本次 MVP 的范围之内；这里只是为将来留出了一条可行路径。如果未来真的需要硬实时，应当把它当作一个独立阶段（类似 seL4 式的形式化验证阶段）来对待，而不是简单地在这张表里加一行。

### 7.4 对第 8 节（MVP 验收标准）的更新 —— 新增条目

6. 组合场景：「Server + RealTime」配置文件同时激活（例如一个低延迟消息队列服务器），并且在没有任何配置冲突的情况下正常运行。

---

## 8. 已敲定的决策表（本版本）

| 主题 | 决策 |
|---|---|
| 商店模式 | 集中式，仅提供已签名且经过测试的软件包 |
| 商店以外的安装 | 禁止，没有例外 |
| 驱动安装 | 只能由用户主动选择，绝不自动进行 |
| 软件安装 | 按用户，二进制文件在磁盘层面共享 |
| Admin 模型 | 没有常驻权限；每次都需要通过 Elevation 确认 |
| 日志发送 | 机会式，系统级别（而不是按用户） |
| 备份目的地 | 必须在系统主磁盘之外 |
| 系统默认安装 | 通过 Installation Manifest 实现，而不是硬编码 |

---

## 9. 剩余的开放问题

- **Windows Compat Runtime：** 运行闭源 Windows 软件的具体方式（类似 Wine 的 API 转译，还是需要 Windows 许可证的完整虚拟机）尚未决定。细节记录在 `05-Legacy-Compat-Applications-Layer-v2.md` 的开放问题部分。
- **真正的硬实时：** 如果将来确实需要确定性的截止时限保证（而不仅仅是延迟优先），应当把它作为一个独立阶段、在微内核中进行形式化验证来考察，而不是对现有 `RealTime` 配置文件的简单扩展。
