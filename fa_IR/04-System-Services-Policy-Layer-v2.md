# لایه ۴ — سرویس‌های سیستمی + Profile Policy Layer (نسخه ۲)

> **لایه:** ۴ - سرویس‌های سیستمی + Profile Policy Layer (نسخه ۲)
> **وضعیت:** سند معماری — بازنگری نسخه ۲، بر اساس تحلیل نیازهای کاربر عمومی/خانگی. این نسخه جایگزین بخش‌های مرتبط سند `04-System-Services-Policy-Layer.md` می‌شود، نه کل آن سند (به «وضعیت این سند» در پایین نگاه کنید).
> **زبان پیاده‌سازی:** Rust برای سرویس‌های اصلی (Init، Simurgh Store، Security Broker، Policy Engine)؛ C برای سطح ABI لایه‌ی POSIX Compatibility، با پشتیبانی از توابع Rust با `#[no_mangle] extern "C"` هرجا ممکن باشد (الگوی «Rust-backed libc»)؛ TOML برای تنظیمات استاتیک و Rhai برای منطق پویای Policy Engine — بدون تغییر نسبت به سند اصلی (`04-System-Services-Policy-Layer.md`)، بخش ۹
> **نوع اجرا:** User-space، بدون نیاز به Capability بالا (به‌جز Security Broker) — بدون تغییر نسبت به سند اصلی

---

**وضعیت این سند:** بازطراحی بر اساس تحلیل نیازهای «کاربر عمومی/خانگی». این نسخه جایگزین بخش‌های مرتبط در `04-System-Services-Policy-Layer.md` می‌شود، نه کل آن سند. بخش‌های Init و POSIX Compatibility Layer بدون تغییر از نسخه قبلی معتبر می‌مانند. بخش ۶.۱، ۶.۲، و ۸ سند اصلی (Profile Policy Layer) هم در این نسخه به‌روزرسانی شده‌اند (بخش ۷ همین سند).

---

## تغییرات نسبت به نسخه قبلی (خلاصه)

| بخش | نسخه قبلی | نسخه جدید (این سند) |
|---|---|---|
| Package Ecosystem | مدل باز شبیه Nixpkgs، کامپایل سمت کاربر | فروشگاه متمرکز، فقط بسته‌های از‌پیش‌کامپایل و تست‌شده |
| نصب خارج از فروشگاه | مشخص نبود | **ممنوع بدون استثنا** |
| لاگ/خطا | وجود نداشت | سرویس جدید: Diagnostics & Telemetry |
| چندکاربری | مشخص نبود | سرویس جدید: Account & Session Manager با مدل Elevation |
| پشتیبان‌گیری | وجود نداشت | سرویس جدید: Backup Manager |
| نصب پیش‌فرض سیستم | وجود نداشت | مفهوم جدید: Installation Manifest |
| Profile Policy — انواع پروفایل | ۴ پروفایل: General/Professional/Gaming/AI | ۶ پروفایل: + Server + RealTime (soft real-time) |

---

## 1. اجزای به‌روزشده‌ی این لایه

```
لایه ۴ (نسخه ۲)
 ├── Init / Service Manager                (بدون تغییر از نسخه ۱)
 ├── POSIX Compatibility Layer             (بدون تغییر از نسخه ۱)
 ├── Simurgh Store                         ← جایگزین Package/Build Ecosystem قدیمی
 ├── Diagnostics & Telemetry Manager       ← جدید
 ├── Security / Permission Broker          (گسترش‌یافته با Elevation)
 ├── Account & Session Manager             ← جدید
 ├── Backup Manager                        ← جدید
 └── Profile Policy Layer                  (به‌روزرسانی: افزوده‌شدن پروفایل RealTime)
```

---

## 2. Diagnostics & Telemetry Manager

### 2.1 هدف
هر خطا یا رفتار غیرعادی در هر جزء سیستم (از میکروکرنل تا یک اپ) باید ثبت، دسته‌بندی، و (با رضایت کاربر) برای تیم توسعه ارسال شود — طوری که برنامه‌نویس بدون نیاز به داده‌ی اضافی از کاربر بتواند باگ را رفع کند.

### 2.2 معماری دو بخشی

- **Log Collector** (لایه ۳): زیرساخت مشترک جمع‌آوری لاگ خام از همه‌ی اجزای سیستم؛ کنار VFS/Netstack/Compositor.
- **Diagnostics Manager** (همین لایه): تصمیم‌گیری سیاستی — چه‌چیزی، چه زمانی، با چه سطح رضایتی ارسال شود.

### 2.3 ساختار گزارش خطا

```rust
pub struct CrashReport {
    // شناسایی دقیق کجا و چه چیزی
    pub component_id: String,        // مثلا "driver-printer-hp-laserjet"
    pub component_version: String,
    pub layer: LayerId,

    // وضعیت کامل سیستم لحظه خطا
    pub os_version: String,
    pub hardware_manifest_snapshot: HardwareManifest,
    pub active_profile: ProfileKind,

    // جزئیات فنی خود خطا
    pub severity: Severity,
    pub error_code: Option<u32>,
    pub stack_trace: Vec<StackFrame>,
    pub capability_state: Option<CapabilitySnapshot>,
    pub last_ipc_chain: Vec<IpcCallRecord>,
    pub memory_state: Option<MemoryDiagnostic>,

    // بازتولید، بدون داده شخصی
    pub preceding_actions: Vec<AnonymizedAction>,  // مثلا "file_open_attempt"
    pub timestamp: u64,
    pub anonymous_device_id: Uuid,
}
```

نکته طراحی حیاتی: هیچ نام فایل، مسیر واقعی، یا محتوای داده‌ی شخصی در گزارش قرار نمی‌گیرد. به‌جای آن، یک بافر حلقه‌ای (ring buffer) از آخرین ۲۰-۳۰ عملیات به‌صورت anonymized نگه‌داشته می‌شود تا در صورت کرش به گزارش اضافه شود.

### 2.4 تصمیمات نهایی‌شده

- **غیرقابل حذف:** این سرویس جزو نصب پایه است، مثل File Manager.
- **دسترسی Admin:** به لاگ همه‌ی کاربران دسترسی کامل دارد (محلی).
- **ارسال:** فرصت‌طلبانه — هر کاربر لاگین‌شده‌ای که به اینترنت دسترسی داشت، لاگ‌های جدید کل سیستم (نه فقط خودش) در پس‌زمینه دوره‌ای ارسال می‌شود؛ رضایت کلی ارسال یک تنظیم سراسری سیستم است، نه per-user.

---

## 3. Simurgh Store (جایگزین Package/Build Ecosystem قدیمی)

### 3.1 اصل کلی
برخلاف مدل باز Nixpkgs در نسخه قبلی، همه‌چیز **متمرکز، از‌پیش‌کامپایل، و تست‌شده توسط تیم سیمرغ** است. هیچ کامپایلی روی سیستم کاربر انجام نمی‌شود.

### 3.2 خط تولید بسته (سمت تیم سیمرغ)

```
سورس نرم‌افزار (متن‌باز یا closed-source، لینوکسی یا ویندوزی)
   ↓
تشخیص مسیر اجرا: Native | LinuxCompatTested | WindowsCompatTested
   ↓
کامپایل/بسته‌بندی
   ↓
تست خودکار (CI): نصب، اجرا، خروج تمیز
   ↓
امضای دیجیتال
   ↓
انتشار در فروشگاه
```

### 3.3 ساختار بسته

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

### 3.4 تصمیمات نهایی‌شده

- **بدون نصب خارج از فروشگاه:** بدون استثنا. Security Broker پیش از هر `install()` امضای بسته را تایید می‌کند؛ بسته‌ی بدون امضای معتبر تیم سیمرغ اصلاً نصب نمی‌شود.
- **درایورها هم در فروشگاه‌اند:** HAL/Device Manager فقط شناسایی می‌کند؛ نصب/آپدیت درایور همیشه با انتخاب کاربر از فروشگاه است، نه خودکار.
- **آپدیت:** یک بخش «آپدیت‌ها» در فروشگاه، شامل سیستم‌عامل، اپ‌ها، درایورها، UI؛ نصب فقط با کلیک کاربر.
- **پشتیبانی از closed-source ویندوزی:** بله، مثل لینوکس تست و ارائه می‌شود (جزئیات اجرا در سند لایه ۵، به‌عنوان سوال باز ثبت شده).

### 3.5 نصب per-user

هر کاربر مستقل نصب/حذف می‌کند (طبق بخش ۴). برای جلوگیری از اتلاف فضای دیسک، باینری واقعی در content-addressed storage مشترک است؛ فقط رکورد نصب، Capability، و دیتا برای هر کاربر جداست:

```rust
pub struct AppInstallRecord {
    pub owner_uid: UserId,
    pub package_ref: ContentAddressedRef,
    pub granted_capabilities: Vec<Capability>,
    pub user_data_path: PathBuf,
}
```

---

## 4. Account & Session Manager (جدید)

### 4.1 مدل داده

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

### 4.2 مدل ایزوله‌سازی
هر کاربر یک فضای Capability جداگانه دارد (نه فقط permission بر پایه UID/GID). یعنی حتی یک اپ باگ‌دار از نظر فنی امکان دسترسی به فایل‌های کاربر دیگر را ندارد.

### 4.3 مدل Admin — بدون دسترسی دائم (Privilege Elevation)

Admin در حالت عادی مثل کاربر معمولی کار می‌کند. برای هر عملیات حساس (نصب درایور، دیدن لاگ/فایل کاربر دیگر، تغییر تنظیمات سیستمی)، سیستم هر بار رمز/تایید تازه می‌خواهد — بدون استثنا، حتی برای دیدن فایل‌های کاربر دیگر بار دوم در همان نشست.

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

### 4.4 تعویض UI بین کاربران

```rust
pub trait SessionManager {
    fn switch_shell(&self, uid: UserId, new_shell: PackageRef) -> Result<(), SessionError>;
}
```
هم از Login Screen، هم از تنظیمات داخل UI در حال اجرا قابل انجام است.

---

## 5. Backup Manager (جدید)

### 5.1 اصل طراحی
چون اپ‌های نصب‌شده فقط رفرنس به بسته‌ی فروشگاه‌اند، بک‌آپ نیازی به کپی باینری کامل برنامه‌ها ندارد — فقط لیست می‌کند «این کاربر چه اپ‌هایی داشت» و در بازیابی، از فروشگاه دوباره دانلود می‌شوند.

### 5.2 ساختار

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

### 5.3 زمان‌بندی

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

### 5.4 مقصد ذخیره — الزام امنیتی (اصل ۳-۲-۱)

```rust
pub enum BackupDestination {
    ExternalDrive { device_id: Uuid },
    NetworkLocation { address: String },
    // عمداً هیچ گزینه‌ی "دیسک اصلی سیستم" وجود ندارد
}
```

### 5.5 بازیابی

قابل انجام روی همان دستگاه یا هر دستگاه دیگر سیمرغ: نصب خودکار اپ‌ها از فروشگاه (طبق مکانیزم Installation Manifest، بخش ۶) + بازگردانی دیتا + اعمال تنظیمات. Admin می‌تواند برای هر کاربر جدا بک‌آپ تنظیم/بگیرد.

---

## 6. Installation Manifest (مفهوم جدید)

فایلی که پیش از نصب آماده می‌شود و مشخص می‌کند نصب‌کننده باید چه چیزی را خودکار از فروشگاه نصب کند — تصمیم «چه چیزی پیش‌فرض باشد» را از کد سیستم جدا می‌کند.

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

جریان: HAL discovery + میکروکرنل + زیرسیستم‌های حیاتی نصب می‌شود → Init/Installer فایل را می‌خواند → Package Manager به فروشگاه وصل می‌شود و هرکدام از موارد فایل را خودکار دانلود/نصب می‌کند → بوت اول با UI و اپ‌های پیش‌فرض آماده.

می‌تواند نسخه‌های مختلف داشته باشد (نصب خانگی، سازمانی، یا سفارشی کاربر پیشرفته)، شبیه Kickstart در لینوکس یا Unattend در ویندوز.

---

## 7. به‌روزرسانی Profile Policy Layer

این بخش جایگزین بخش‌های ۶.۱، ۶.۲، و بند ۶ در بخش ۸ سند اصلی `04-System-Services-Policy-Layer.md` می‌شود.

### 7.1 به‌روزرسانی بخش ۶.۱ — مدل داده

```rust
pub enum ProfileKind {
    General,
    Professional,
    Server,
    RealTime,      // soft real-time / latency-priority — نه hard-RT با WCET تضمین‌شده
    Gaming,
    AI,
}
```

### 7.2 به‌روزرسانی بخش ۶.۲ — جدول شش پروفایل

| تنظیم | عمومی | تخصصی | سرور | بلادرنگ (Soft RT) | گیمینگ | AI |
|---|---|---|---|---|---|---|
| Scheduler default | Interactive | Interactive | Throughput | Interactive (بدون/کم aging) | Interactive (latency ورودی) | Throughput |
| Power Policy | Balanced | Balanced | Efficiency | Performance | Performance | Performance (کنترل حرارت دقیق‌تر) |
| hal-direct دسترسی پیش‌فرض | خیر | بله | خیر | بله (تایمر/perf counter/IRQ pinning) | خیر (فقط GPU-tuning خاص) | بخشی (device memory) |
| Compositor | نصب، فعال | نصب، فعال | لود نمی‌شود | نصب، فعال (اگر لازم) | نصب، فعال | نصب، غیرفعال |
| سرویس AI runtime | نصب، غیرفعال | نصب، غیرفعال | نصب، غیرفعال | نصب، غیرفعال | نصب، غیرفعال | فعال، پیش‌بارگذاری‌شده |
| Unified Memory پیش‌فرض | خیر | اختیاری | خیر | خیر | خیر | بله |
| NUMA-aware scheduling | خیر | بله | بله | بله | بله (اختیاری) | بله |
| lazy-load threshold | پیش‌فرض (۵۰MB) | پیش‌فرض | پیش‌فرض | پایین‌تر (سرویس غیرضروری کمتر لود شود) | پیش‌فرض | override طبق بخش ۹ |

### 7.3 یادداشت معماری — نکته‌ی حیاتی درباره‌ی پروفایل «بلادرنگ»

این پروفایل یک مود زمان‌بندی جدید در میکروکرنل **نمی‌سازد** — طبق `02-Microkernel-Layer.md` بخش ۴.۴، انتخاب مود (Interactive/Throughput) از قبل per-thread است، نه سراسری. پروفایل «بلادرنگ» فقط **پیش‌فرض‌های سطح سیاست** را برای latency قابل‌پیش‌بینی‌تر تنظیم می‌کند:

- `aging_cap_ms` کوچک‌تر یا صفر (جلوگیری از نوسان اولویت ناشی از aging)
- دسترسی پیش‌فرض بازتر به `hal-direct` برای تایمر با دقت بالا و pin کردن ترد به core
- سرویس‌های غیرضروری به‌صورت پیش‌فرض lazy یا خاموش، برای کاهش jitter ناشی از IPC رقابتی

**این پروفایل معادل hard real-time (با تضمین قطعی deadline، مثل سیستم‌های ایمنی-critical صنعتی/avionics) نیست.** طبق `02-Microkernel-Layer.md` بخش ۱.۱، اثبات صوری کامل (پیش‌نیاز واقعی hard-RT) در scope این MVP نیست؛ فقط مسیر برای آن باز نگه داشته شده. اگر در آینده نیاز به hard-RT واقعی بود، باید به‌عنوان یک فاز جدا (شبیه فاز اثبات صوری seL4-style) در نظر گرفته شود، نه صرفاً یک ردیف در این جدول.

### 7.4 به‌روزرسانی بخش ۸ (معیار پذیرش MVP) — آیتم اضافه

6. سناریوی ترکیبی: پروفایل هم‌زمان «سرور + بلادرنگ» فعال شود (مثلاً یک سرور صف‌پیام کم‌تاخیر) و بدون تضاد تنظیمات اجرا شود.

---

## 8. جدول تصمیمات نهایی‌شده (این نسخه)

| موضوع | تصمیم |
|---|---|
| مدل فروشگاه | متمرکز، فقط بسته‌ی امضاشده و تست‌شده |
| نصب خارج از فروشگاه | ممنوع بدون استثنا |
| نصب درایور | فقط با انتخاب کاربر، هرگز خودکار |
| نصب نرم‌افزار | per-user، با اشتراک باینری در سطح دیسک |
| مدل Admin | بدون دسترسی دائم؛ Elevation با تایید هر بار |
| ارسال لاگ | فرصت‌طلبانه، سطح سیستم (نه per-user) |
| مقصد بک‌آپ | اجباراً خارج از دیسک اصلی |
| نصب پیش‌فرض سیستم | از طریق Installation Manifest، نه سخت‌کد شده |

---

## 9. سوالات باز باقیمانده

- **Windows Compat Runtime:** روش دقیق اجرای closed-source ویندوزی (ترجمه‌ی API شبیه Wine در برابر ماشین مجازی کامل با نیاز به لایسنس ویندوز) هنوز تصمیم‌گیری نشده. جزئیات در `05-Legacy-Compat-Applications-Layer-v2.md` بخش سوالات باز ثبت شده است.
- **hard real-time واقعی:** اگر در آینده نیاز به تضمین قطعی deadline (نه فقط latency-priority) پیش بیاید، باید به‌عنوان یک فاز جدا با اثبات صوری در میکروکرنل بررسی شود، نه گسترش پروفایل `RealTime` فعلی.
