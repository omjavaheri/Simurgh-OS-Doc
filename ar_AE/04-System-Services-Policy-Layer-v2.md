# الطبقة ٤ — خدمات النظام + Profile Policy Layer (الإصدار ٢)

> **الطبقة:** ٤ - خدمات النظام + Profile Policy Layer (الإصدار ٢)
> **الحالة:** وثيقة معمارية — مراجعة الإصدار ٢، مبنية على تحليل احتياجات «المستخدم العام/المنزلي». يحل هذا الإصدار محل الأقسام المرتبطة في وثيقة `04-System-Services-Policy-Layer.md`، لا الوثيقة بأكملها (انظر «حالة هذه الوثيقة» أدناه).
> **لغة التنفيذ:** Rust للخدمات الأساسية (Init، Simurgh Store، Security Broker، Policy Engine)؛ لغة C لسطح الـ ABI الخاص بـ POSIX Compatibility Layer، مدعومة قدر الإمكان بدوال Rust بصيغة `#[no_mangle] extern "C"` (نمط «Rust-backed libc»)؛ TOML للإعدادات الثابتة، وRhai للمنطق الديناميكي في Policy Engine — دون تغيير عن الوثيقة الأساسية (`04-System-Services-Policy-Layer.md`)، القسم ٩
> **نوع التشغيل:** فضاء المستخدم، دون حاجة إلى Capability مرتفع (باستثناء Security Broker) — دون تغيير عن الوثيقة الأساسية

---

**حالة هذه الوثيقة:** إعادة تصميم مبنية على تحليل احتياجات «المستخدم العام/المنزلي». يحل هذا الإصدار محل الأقسام المرتبطة في `04-System-Services-Policy-Layer.md`، لا الوثيقة بأكملها. تبقى أقسام Init وPOSIX Compatibility Layer سارية دون تغيير منذ الإصدار السابق. كما تم تحديث القسم ٦.١، والقسم ٦.٢، وبند من القسم ٨ في الوثيقة الأساسية (الخاصة بـ Profile Policy Layer) في هذا الإصدار (القسم ٧ من هذه الوثيقة).

---

## التغييرات مقارنة بالإصدار السابق (ملخص)

| القسم | الإصدار السابق | الإصدار الجديد (هذه الوثيقة) |
|---|---|---|
| Package Ecosystem | نموذج مفتوح شبيه بـ Nixpkgs، مع الترجمة (compile) من جانب المستخدم | متجر مركزي، حزم مُجمَّعة مسبقاً ومُختبرة فقط |
| التثبيت من خارج المتجر | غير محدد | **ممنوع دون استثناء** |
| السجلات/الأخطاء | لم تكن موجودة | خدمة جديدة: Diagnostics & Telemetry |
| دعم تعدد المستخدمين | غير محدد | خدمة جديدة: Account & Session Manager بنموذج Elevation |
| النسخ الاحتياطي | لم يكن موجوداً | خدمة جديدة: Backup Manager |
| التثبيت الافتراضي للنظام | لم يكن موجوداً | مفهوم جديد: Installation Manifest |
| Profile Policy — أنواع ملفات التعريف | ٤ ملفات تعريف: General/Professional/Gaming/AI | ٦ ملفات تعريف: + Server وRealTime (وقت فعلي لين) |

---

## ١. مكوّنات هذه الطبقة بعد التحديث

```
الطبقة ٤ (الإصدار ٢)
 ├── Init / Service Manager                (دون تغيير عن الإصدار ١)
 ├── POSIX Compatibility Layer             (دون تغيير عن الإصدار ١)
 ├── Simurgh Store                         ← يحل محل Package/Build Ecosystem القديم
 ├── Diagnostics & Telemetry Manager       ← جديد
 ├── Security / Permission Broker          (موسَّع بإضافة Elevation)
 ├── Account & Session Manager             ← جديد
 ├── Backup Manager                        ← جديد
 └── Profile Policy Layer                  (تحديث: إضافة ملف تعريف RealTime)
```

---

## ٢. Diagnostics & Telemetry Manager

### ٢.١ الغرض
يجب تسجيل كل خطأ أو سلوك غير طبيعي في أي مكوّن من مكوّنات النظام (من النواة الصغرى وحتى تطبيق ما)، وتصنيفه، وإرساله (بموافقة المستخدم) إلى فريق التطوير — بحيث يستطيع المطوّر إصلاح الخلل دون الحاجة إلى بيانات إضافية من المستخدم.

### ٢.٢ بنية من جزأين

- **Log Collector** (الطبقة ٣): البنية التحتية المشتركة لجمع السجلات الخام من كل مكوّنات النظام؛ إلى جانب VFS/Netstack/Compositor.
- **Diagnostics Manager** (هذه الطبقة): الجهة صانعة القرار السياسي — ماذا يُرسَل، ومتى، وبأي مستوى من الموافقة.

### ٢.٣ بنية تقرير الانهيار

```rust
pub struct CrashReport {
    // تحديد دقيق لمكان وطبيعة الخطأ
    pub component_id: String,        // مثلاً "driver-printer-hp-laserjet"
    pub component_version: String,
    pub layer: LayerId,

    // الحالة الكاملة للنظام لحظة وقوع الخطأ
    pub os_version: String,
    pub hardware_manifest_snapshot: HardwareManifest,
    pub active_profile: ProfileKind,

    // التفاصيل التقنية للخطأ نفسه
    pub severity: Severity,
    pub error_code: Option<u32>,
    pub stack_trace: Vec<StackFrame>,
    pub capability_state: Option<CapabilitySnapshot>,
    pub last_ipc_chain: Vec<IpcCallRecord>,
    pub memory_state: Option<MemoryDiagnostic>,

    // من أجل إعادة إنتاج الخطأ، دون بيانات شخصية
    pub preceding_actions: Vec<AnonymizedAction>,  // مثلاً "file_open_attempt"
    pub timestamp: u64,
    pub anonymous_device_id: Uuid,
}
```

ملاحظة تصميم بالغة الأهمية: لا يُدرَج في التقرير أي اسم ملف، أو مسار حقيقي، أو محتوى بيانات شخصية على الإطلاق. بدلاً من ذلك، يُحتفَظ بذاكرة تخزين دائرية (ring buffer) لآخر ٢٠-٣٠ عملية بصيغة مجهولة الهوية (anonymized)، لتُضاف إلى التقرير في حال وقوع انهيار (crash).

### ٢.٤ القرارات المحسومة نهائياً

- **لا يمكن إزالتها:** هذه الخدمة جزء من التثبيت الأساسي، مثل File Manager.
- **صلاحية وصول Admin:** يملك وصولاً كاملاً (محلياً) إلى سجلات جميع المستخدمين.
- **الإرسال:** انتهازي (opportunistic) — أي مستخدم مسجّل الدخول ولديه اتصال بالإنترنت، تُرسَل سجلات النظام الجديدة بأكملها (وليس سجلاته وحده فقط) في الخلفية وبشكل دوري؛ والموافقة الإجمالية على الإرسال هي إعداد عام على مستوى النظام كله، وليست خاصة بكل مستخدم على حدة.

---

## ٣. Simurgh Store (يحل محل Package/Build Ecosystem القديم)

### ٣.١ المبدأ العام
خلافاً للنموذج المفتوح الشبيه بـ Nixpkgs في الإصدار السابق، كل شيء **مركزي، ومُجمَّع مسبقاً (pre-compiled)، ومُختبر من قِبل فريق سيمرغ**. لا يجري أي تجميع (compile) على نظام المستخدم إطلاقاً.

### ٣.٢ خط إنتاج الحزم (من جانب فريق سيمرغ)

```
المصدر البرمجي (مفتوح المصدر أو مغلق المصدر، لينكس أو ويندوز)
   ↓
تحديد مسار التنفيذ: Native | LinuxCompatTested | WindowsCompatTested
   ↓
التجميع/التعبئة
   ↓
اختبار آلي (CI): التثبيت، التشغيل، الخروج النظيف
   ↓
التوقيع الرقمي
   ↓
النشر في المتجر
```

### ٣.٣ بنية الحزمة

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

### ٣.٤ القرارات المحسومة نهائياً

- **لا تثبيت من خارج المتجر:** دون أي استثناء. يتحقق Security Broker من توقيع الحزمة قبل كل عملية `install()`؛ فالحزمة التي لا تحمل توقيعاً صالحاً من فريق سيمرغ لا تُثبَّت إطلاقاً.
- **برامج التشغيل موجودة في المتجر أيضاً:** يقتصر دور HAL/Device Manager على الاكتشاف فقط؛ تثبيت/تحديث برنامج التشغيل يتم دائماً باختيار المستخدم من المتجر، وليس تلقائياً أبداً.
- **التحديثات:** قسم «التحديثات» في المتجر، يشمل نظام التشغيل، التطبيقات، برامج التشغيل، وواجهة المستخدم؛ ولا يتم التثبيت إلا بنقرة من المستخدم.
- **دعم برمجيات ويندوز مغلقة المصدر:** نعم، تُختبَر وتُقدَّم بنفس طريقة برمجيات لينكس (تفاصيل التنفيذ مذكورة في وثيقة الطبقة ٥، وموثقة هناك كسؤال مفتوح).

### ٣.٥ التثبيت لكل مستخدم

يقوم كل مستخدم بالتثبيت/الحذف بشكل مستقل (وفق القسم ٤). ولتجنّب هدر مساحة القرص، يكون الملف الثنائي الفعلي مشتركاً في content-addressed storage؛ ولا ينفصل بحسب كل مستخدم سوى سجل التثبيت، وCapability، والبيانات:

```rust
pub struct AppInstallRecord {
    pub owner_uid: UserId,
    pub package_ref: ContentAddressedRef,
    pub granted_capabilities: Vec<Capability>,
    pub user_data_path: PathBuf,
}
```

---

## ٤. Account & Session Manager (جديد)

### ٤.١ نموذج البيانات

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

### ٤.٢ نموذج العزل
يملك كل مستخدم فضاء Capability منفصلاً خاصاً به (وليس مجرد permission قائم على UID/GID). أي أنه حتى تطبيق فيه خلل تقني (buggy) لا يملك من الناحية التقنية أي وسيلة للوصول إلى ملفات مستخدم آخر.

### ٤.٣ نموذج Admin — بدون وصول دائم (Privilege Elevation)

يعمل Admin في الحالة العادية تماماً مثل مستخدم عادي. وفي كل عملية حساسة (تثبيت برنامج تشغيل، الاطلاع على سجلات/ملفات مستخدم آخر، تغيير إعدادات النظام)، يطلب النظام كلمة مرور/تأكيداً جديداً في كل مرة — دون أي استثناء، حتى عند الاطلاع على ملفات مستخدم آخر للمرة الثانية ضمن الجلسة نفسها.

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

### ٤.٤ تبديل واجهة المستخدم بين المستخدمين

```rust
pub trait SessionManager {
    fn switch_shell(&self, uid: UserId, new_shell: PackageRef) -> Result<(), SessionError>;
}
```
يمكن إجراء ذلك سواء من Login Screen أو من الإعدادات داخل واجهة مستخدم قيد التشغيل.

---

## ٥. Backup Manager (جديد)

### ٥.١ مبدأ التصميم
بما أن التطبيقات المثبَّتة ليست سوى إشارة (reference) إلى حزمة في المتجر، فإن النسخ الاحتياطي لا يحتاج إلى نسخ الملفات الثنائية الكاملة للتطبيقات — بل يكتفي بتسجيل «ما هي التطبيقات التي كانت لدى هذا المستخدم»، وعند الاستعادة تُنزَّل هذه التطبيقات مجدداً من المتجر.

### ٥.٢ البنية

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

### ٥.٣ الجدولة

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

### ٥.٤ وجهة التخزين — متطلب أمني (مبدأ ٣-٢-١)

```rust
pub enum BackupDestination {
    ExternalDrive { device_id: Uuid },
    NetworkLocation { address: String },
    // عمداً، لا يوجد خيار «القرص الرئيسي للنظام»
}
```

### ٥.٥ الاستعادة

يمكن إجراؤها على نفس الجهاز أو على أي جهاز آخر يعمل بسيمرغ: تثبيت تلقائي للتطبيقات من المتجر (عبر آلية Installation Manifest، القسم ٦) + استعادة البيانات + تطبيق الإعدادات. ويستطيع Admin إعداد/تنفيذ نسخة احتياطية منفصلة لكل مستخدم.

---

## ٦. Installation Manifest (مفهوم جديد)

ملف يُعَدّ قبل التثبيت ويحدد ما الذي يجب على المثبِّت (installer) تثبيته تلقائياً من المتجر — بحيث يفصل قرار «ما الذي ينبغي أن يكون افتراضياً» عن كود النظام نفسه.

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

سير العمل: تُثبَّت HAL discovery + النواة الصغرى + الأنظمة الفرعية الحرجة ← يقرأ Init/Installer الملف ← يتصل Package Manager بالمتجر ويقوم تلقائياً بتنزيل/تثبيت كل عنصر مذكور في الملف ← يصبح الإقلاع الأول جاهزاً بواجهة المستخدم والتطبيقات الافتراضية.

يمكن أن تكون له إصدارات مختلفة (تثبيت منزلي، أو مؤسساتي، أو مخصص لمستخدم متقدم)، على غرار Kickstart في لينكس أو Unattend في ويندوز.

---

## ٧. تحديث Profile Policy Layer

يحل هذا القسم محل القسم ٦.١، والقسم ٦.٢، والبند ٦ من القسم ٨ في الوثيقة الأساسية `04-System-Services-Policy-Layer.md`.

### ٧.١ تحديث القسم ٦.١ — نموذج البيانات

```rust
pub enum ProfileKind {
    General,
    Professional,
    Server,
    RealTime,      // وقت فعلي لين (soft real-time) / أولوية لزمن الاستجابة — وليس hard-RT بضمان WCET محدد
    Gaming,
    AI,
}
```

### ٧.٢ تحديث القسم ٦.٢ — جدول ملفات التعريف الستة

| الإعداد | عام | احترافي | خادم | وقت فعلي (Soft RT) | ألعاب | AI |
|---|---|---|---|---|---|---|
| نمط المجدوِل الافتراضي | Interactive | Interactive | Throughput | Interactive (دون/بحد أدنى من aging) | Interactive (أولوية لزمن استجابة الإدخال) | Throughput |
| Power Policy | Balanced | Balanced | Efficiency | Performance | Performance | Performance (تحكم حراري أدق) |
| وصول `hal-direct` الافتراضي | لا | نعم | لا | نعم (مؤقت/perf counter/IRQ pinning) | لا (فقط ضبط GPU خاص) | جزئي (device memory) |
| Compositor | مثبَّت، فعّال | مثبَّت، فعّال | لا يُحمَّل | مثبَّت، فعّال (عند الحاجة) | مثبَّت، فعّال | مثبَّت، غير فعّال |
| خدمة AI runtime | مثبَّتة، غير فعّالة | مثبَّتة، غير فعّالة | مثبَّتة، غير فعّالة | مثبَّتة، غير فعّالة | مثبَّتة، غير فعّالة | فعّالة، مُحمَّلة مسبقاً |
| Unified Memory افتراضياً | لا | اختياري | لا | لا | لا | نعم |
| جدولة واعية بـ NUMA | لا | نعم | نعم | نعم | نعم (اختياري) | نعم |
| عتبة lazy-load | افتراضي (٥٠MB) | افتراضي | افتراضي | أقل (تحميل عدد أقل من الخدمات غير الضرورية) | افتراضي | تجاوز (override) وفق القسم ٩ |

### ٧.٣ ملاحظة معمارية — نقطة بالغة الأهمية حول ملف تعريف «RealTime»

هذا الملف التعريفي **لا** يُنشئ نمط جدولة جديداً في النواة الصغرى — فوفقاً لـ `02-Microkernel-Layer.md`، القسم ٤.٤، يكون اختيار النمط (Interactive/Throughput) مُحدَّداً أصلاً لكل خيط (per-thread) على حدة، وليس على مستوى النظام كله. لا يعمل ملف تعريف «RealTime» إلا على ضبط **الإعدادات الافتراضية على مستوى السياسة** لجعل زمن الاستجابة أكثر قابلية للتنبؤ:

- قيمة أصغر أو صفرية لـ `aging_cap_ms` (لمنع تذبذب الأولوية الناتج عن aging)
- وصول افتراضي أكثر انفتاحاً إلى `hal-direct` من أجل المؤقتات عالية الدقة وتثبيت (pin) الخيط على نواة معينة
- تحميل الخدمات غير الضرورية بصيغة lazy أو تعطيلها افتراضياً، لتقليل الاهتزاز (jitter) الناتج عن تنافس IPC

**هذا الملف التعريفي لا يعادل الوقت الفعلي الصلب (hard real-time، بضمان حتمي لموعد نهائي، كما في الأنظمة الصناعية/أنظمة الطيران ذات الأهمية الأمنية الحرجة).** فوفقاً لـ `02-Microkernel-Layer.md`، القسم ١.١، لا يدخل التحقق الصوري الكامل (formal verification، وهو الشرط المسبق الحقيقي للـ hard-RT) في نطاق هذا الـ MVP؛ بل تم فقط ترك المسار مفتوحاً نحوه. وإذا احتاج الأمر مستقبلاً إلى hard-RT حقيقي، فينبغي التعامل معه كمرحلة منفصلة (على غرار مرحلة التحقق الصوري بطريقة seL4)، لا مجرد إضافة صف جديد إلى هذا الجدول.

### ٧.٤ تحديث القسم ٨ (معايير اكتمال الـ MVP) — بند إضافي

٦. سيناريو مركّب: تفعيل ملفَي تعريف «Server + RealTime» في آن واحد (مثل خادم طابور رسائل منخفض زمن الاستجابة)، وتشغيله دون أي تعارض في الإعدادات.

---

## ٨. جدول القرارات المحسومة نهائياً (هذا الإصدار)

| الموضوع | القرار |
|---|---|
| نموذج المتجر | مركزي، حزم موقَّعة ومُختبرة فقط |
| التثبيت من خارج المتجر | ممنوع دون استثناء |
| تثبيت برامج التشغيل | فقط باختيار المستخدم، أبداً تلقائياً |
| تثبيت البرمجيات | لكل مستخدم، مع مشاركة الملف الثنائي على مستوى القرص |
| نموذج Admin | بدون وصول دائم؛ Elevation بتأكيد في كل مرة |
| إرسال السجلات | انتهازي، على مستوى النظام (وليس لكل مستخدم) |
| وجهة النسخ الاحتياطي | إلزامياً خارج القرص الرئيسي للنظام |
| التثبيت الافتراضي للنظام | عبر Installation Manifest، وليس مُبرمَجاً بشكل ثابت (hardcoded) |

---

## ٩. الأسئلة المفتوحة المتبقية

- **Windows Compat Runtime:** لم يُحسَم بعد الأسلوب الدقيق لتشغيل برمجيات ويندوز مغلقة المصدر (ترجمة واجهة برمجية على غرار Wine مقابل آلة افتراضية كاملة تتطلب ترخيص ويندوز). التفاصيل موثقة في قسم الأسئلة المفتوحة بوثيقة `05-Legacy-Compat-Applications-Layer-v2.md`.
- **hard real-time حقيقي:** إذا برزت مستقبلاً حاجة إلى ضمان حتمي لموعد نهائي (وليس مجرد أولوية لزمن الاستجابة)، فينبغي دراسته كمرحلة منفصلة تتضمن تحققاً صورياً في النواة الصغرى، لا كمجرد توسيع لملف تعريف `RealTime` الحالي.
