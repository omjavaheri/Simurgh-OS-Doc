# Layer 4 — System Services + Profile Policy Layer (Version 2)

> **Layer:** 4 - System Services + Profile Policy Layer (Version 2)
> **Status:** Architecture Specification — Revision 2, based on an analysis of general/home-user needs. This version replaces the related sections of `04-System-Services-Policy-Layer.md`, not the whole document (see "Status of This Document" below).
> **Implementation language:** Rust for the core services (Init, Simurgh Store, Security Broker, Policy Engine); C for the POSIX Compatibility Layer's ABI surface, backed by `#[no_mangle] extern "C"` Rust functions wherever possible (the "Rust-backed libc" pattern); TOML for static configuration, Rhai for dynamic Policy Engine logic — unchanged from the base document (`04-System-Services-Policy-Layer.md`), Section 9
> **Execution type:** User-space, no elevated Capability required (except the Security Broker) — unchanged from the base document

---

**Status of this document:** a redesign based on an analysis of "general/home user" needs. This version replaces the related sections in `04-System-Services-Policy-Layer.md`, not the whole document. The Init and POSIX Compatibility Layer sections remain valid, unchanged, from the previous version. Sections 6.1, 6.2, and 8 of the base document (Profile Policy Layer) are also updated in this version (Section 7 of this document).

---

## Changes from the Previous Version (Summary)

| Section | Previous Version | New Version (this document) |
|---|---|---|
| Package Ecosystem | Open Nixpkgs-like model, user-side compilation | Centralized store, only pre-compiled and tested packages |
| Installation outside the store | Unspecified | **Forbidden, no exceptions** |
| Logging/errors | Did not exist | New service: Diagnostics & Telemetry |
| Multi-user support | Unspecified | New service: Account & Session Manager with an Elevation model |
| Backups | Did not exist | New service: Backup Manager |
| Default system installation | Did not exist | New concept: Installation Manifest |
| Profile Policy — profile kinds | 4 profiles: General/Professional/Gaming/AI | 6 profiles: + Server + RealTime (soft real-time) |

---

## 1. Updated Components of This Layer

```
Layer 4 (Version 2)
 ├── Init / Service Manager                (unchanged from Version 1)
 ├── POSIX Compatibility Layer             (unchanged from Version 1)
 ├── Simurgh Store                         ← replaces the old Package/Build Ecosystem
 ├── Diagnostics & Telemetry Manager       ← new
 ├── Security / Permission Broker          (expanded with Elevation)
 ├── Account & Session Manager             ← new
 ├── Backup Manager                        ← new
 └── Profile Policy Layer                  (updated: RealTime profile added)
```

---

## 2. Diagnostics & Telemetry Manager

### 2.1 Purpose
Every error or abnormal behavior in any system component (from the microkernel up to an app) must be logged, categorized, and — with the user's consent — sent to the development team, so that a developer can fix the bug without needing extra data from the user.

### 2.2 Two-Part Architecture

- **Log Collector** (Layer 3): the shared infrastructure that collects raw logs from every system component; sits alongside VFS/Netstack/Compositor.
- **Diagnostics Manager** (this layer): the policy-making side — deciding what gets sent, when, and at what consent level.

### 2.3 Crash Report Structure

```rust
pub struct CrashReport {
    // Precise identification of where and what
    pub component_id: String,        // e.g. "driver-printer-hp-laserjet"
    pub component_version: String,
    pub layer: LayerId,

    // Full system state at the moment of the error
    pub os_version: String,
    pub hardware_manifest_snapshot: HardwareManifest,
    pub active_profile: ProfileKind,

    // Technical details of the error itself
    pub severity: Severity,
    pub error_code: Option<u32>,
    pub stack_trace: Vec<StackFrame>,
    pub capability_state: Option<CapabilitySnapshot>,
    pub last_ipc_chain: Vec<IpcCallRecord>,
    pub memory_state: Option<MemoryDiagnostic>,

    // For reproduction, without personal data
    pub preceding_actions: Vec<AnonymizedAction>,  // e.g. "file_open_attempt"
    pub timestamp: u64,
    pub anonymous_device_id: Uuid,
}
```

Critical design note: no file name, real path, or personal data content is ever included in the report. Instead, a ring buffer of the last 20-30 operations is kept in anonymized form, to be appended to the report if a crash occurs.

### 2.4 Finalized Decisions

- **Cannot be removed:** this service is part of the base install, like the File Manager.
- **Admin access:** has full access to every user's logs (locally).
- **Sending:** opportunistic — for any logged-in user with internet access, the whole system's new logs (not just their own) are sent periodically in the background; overall consent to send is a single, system-wide setting, not per-user.

---

## 3. Simurgh Store (replaces the old Package/Build Ecosystem)

### 3.1 General Principle
Unlike the open Nixpkgs-like model in the previous version, everything is **centralized, pre-compiled, and tested by the Simurgh team**. No compilation ever happens on the user's system.

### 3.2 Package Production Pipeline (Simurgh team side)

```
Software source (open-source or closed-source, Linux or Windows)
   ↓
Execution-path detection: Native | LinuxCompatTested | WindowsCompatTested
   ↓
Compile/package
   ↓
Automated (CI) testing: install, run, clean exit
   ↓
Digital signing
   ↓
Publish to the store
```

### 3.3 Package Structure

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

### 3.4 Finalized Decisions

- **No installation outside the store:** no exceptions. The Security Broker verifies a package's signature before every `install()`; a package without a valid Simurgh-team signature is never installed at all.
- **Drivers are in the store too:** the HAL/Device Manager only performs discovery; installing/updating a driver is always the user's choice from the store, never automatic.
- **Updates:** an "Updates" section in the store, covering the OS, apps, drivers, and the UI; installation only happens on the user's click.
- **Closed-source Windows software support:** yes, tested and offered the same way as Linux software (execution details are in the Layer 5 document, logged there as an open question).

### 3.5 Per-user Installation

Each user installs/removes independently (per Section 4). To avoid wasting disk space, the actual binary lives in shared content-addressed storage; only the install record, Capabilities, and data are separate per user:

```rust
pub struct AppInstallRecord {
    pub owner_uid: UserId,
    pub package_ref: ContentAddressedRef,
    pub granted_capabilities: Vec<Capability>,
    pub user_data_path: PathBuf,
}
```

---

## 4. Account & Session Manager (New)

### 4.1 Data Model

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

### 4.2 Isolation Model
Each user has a separate Capability space (not just UID/GID-based permissions). This means even a technically buggy app has no way to access another user's files.

### 4.3 Admin Model — No Standing Access (Privilege Elevation)

In normal operation, an Admin works just like a regular user. For every sensitive operation (installing a driver, viewing another user's logs/files, changing system settings), the system requires a fresh password/confirmation every single time — no exceptions, even for viewing another user's files a second time within the same session.

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

### 4.4 Switching the UI Between Users

```rust
pub trait SessionManager {
    fn switch_shell(&self, uid: UserId, new_shell: PackageRef) -> Result<(), SessionError>;
}
```
Can be done both from the Login Screen and from settings inside a running UI session.

---

## 5. Backup Manager (New)

### 5.1 Design Principle
Because installed apps are just references to a store package, a backup doesn't need to copy the full application binaries — it only records "which apps this user had," and on restore, they're downloaded again from the store.

### 5.2 Structure

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

### 5.3 Scheduling

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

### 5.4 Storage Destination — Security Requirement (3-2-1 Principle)

```rust
pub enum BackupDestination {
    ExternalDrive { device_id: Uuid },
    NetworkLocation { address: String },
    // Deliberately, there is no "main system disk" option
}
```

### 5.5 Restore

Can be performed on the same device or on any other Simurgh device: automatic app installation from the store (via the Installation Manifest mechanism, Section 6) + data restore + settings applied. An Admin can configure/run a separate backup for each user.

---

## 6. Installation Manifest (New Concept)

A file prepared before installation that specifies what the installer should automatically install from the store — separating the decision of "what should be the default" from the system code itself.

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

Flow: HAL discovery + the microkernel + critical subsystems are installed → Init/Installer reads the file → the Package Manager connects to the store and automatically downloads/installs each item listed in the file → the first boot is ready with the default UI and default apps.

It can have different variants (home installation, enterprise, or advanced-user custom), similar to Kickstart on Linux or Unattend on Windows.

---

## 7. Profile Policy Layer Update

This section replaces Sections 6.1, 6.2, and item 6 of Section 8 in the base document, `04-System-Services-Policy-Layer.md`.

### 7.1 Update to Section 6.1 — Data Model

```rust
pub enum ProfileKind {
    General,
    Professional,
    Server,
    RealTime,      // soft real-time / latency-priority — not hard-RT with a guaranteed WCET
    Gaming,
    AI,
}
```

### 7.2 Update to Section 6.2 — The Six-Profile Table

| Setting | General | Professional | Server | RealTime (Soft RT) | Gaming | AI |
|---|---|---|---|---|---|---|
| Scheduler default | Interactive | Interactive | Throughput | Interactive (no/low aging) | Interactive (input latency) | Throughput |
| Power Policy | Balanced | Balanced | Efficiency | Performance | Performance | Performance (finer-grained thermal control) |
| Default `hal-direct` access | No | Yes | No | Yes (timer/perf counter/IRQ pinning) | No (only specific GPU tuning) | Partial (device memory) |
| Compositor | Installed, active | Installed, active | Not loaded | Installed, active (if needed) | Installed, active | Installed, inactive |
| AI runtime service | Installed, inactive | Installed, inactive | Installed, inactive | Installed, inactive | Installed, inactive | Active, preloaded |
| Unified Memory by default | No | Optional | No | No | No | Yes |
| NUMA-aware scheduling | No | Yes | Yes | Yes | Yes (optional) | Yes |
| lazy-load threshold | Default (50MB) | Default | Default | Lower (fewer non-essential services loaded) | Default | Override per Section 9 |

### 7.3 Architectural Note — A Critical Point About the "RealTime" Profile

This profile **does not** create a new scheduling mode in the microkernel — per `02-Microkernel-Layer.md`, Section 4.4, the mode choice (Interactive/Throughput) is already made per-thread, not system-wide. The "RealTime" profile only adjusts **policy-level defaults** for more predictable latency:

- A smaller or zero `aging_cap_ms` (prevents priority fluctuation caused by aging)
- More open default access to `hal-direct` for high-precision timers and pinning a thread to a core
- Non-essential services lazy-loaded or disabled by default, to reduce jitter caused by competing IPC

**This profile is not equivalent to hard real-time (with a firm, guaranteed deadline, as in safety-critical industrial/avionics systems).** Per `02-Microkernel-Layer.md`, Section 1.1, full formal verification (the real prerequisite for hard-RT) is out of scope for this MVP; only the path toward it is kept open. If real hard-RT is needed in the future, it should be treated as a separate phase (similar to a seL4-style formal-verification phase), not simply a row added to this table.

### 7.4 Update to Section 8 (MVP Acceptance Criteria) — Added Item

6. Combined scenario: the "Server + RealTime" profile is active at the same time (e.g. a low-latency message-queue server) and runs without any conflicting settings.

---

## 8. Table of Finalized Decisions (This Version)

| Topic | Decision |
|---|---|
| Store model | Centralized, signed and tested packages only |
| Installation outside the store | Forbidden, no exceptions |
| Driver installation | Only by user choice, never automatic |
| Software installation | Per-user, with disk-level binary sharing |
| Admin model | No standing access; Elevation with confirmation every time |
| Log sending | Opportunistic, system-level (not per-user) |
| Backup destination | Must be off the main system disk |
| Default system installation | Via the Installation Manifest, not hardcoded |

---

## 9. Remaining Open Questions

- **Windows Compat Runtime:** the exact method for running closed-source Windows software (Wine-like API translation vs. a full VM requiring a Windows license) has not yet been decided. Details are logged in `05-Legacy-Compat-Applications-Layer-v2.md`'s open-questions section.
- **Real hard real-time:** if a firm deadline guarantee (not just latency-priority) is needed in the future, it should be examined as a separate phase with formal verification in the microkernel, not as an extension of the current `RealTime` profile.
