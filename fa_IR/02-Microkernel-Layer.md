# لایه ۲ — میکروکرنل (مبتنی بر Capability)

> **لایه:** ۲ - میکروکرنل (مبتنی بر Capability)
> **وضعیت:** سند معماری
> **زبان پیاده‌سازی:** Rust (`no_std` تا مرز IPC؛ هیچ بخشی از این لایه نباید به heap allocator نامحدود یا panic-unwinding عادی متکی باشد)
> **نوع اجرا:** Kernel-space، Privileged mode
> **الهام طراحی:** seL4 (مدل capability و اثبات‌پذیری)، Zircon (مدل IPC و object handle)

---

## 0. ارتباط با لایه‌ی زیرین و بالادستی

**لایه‌ی زیرین (HAL، لایه ۱):** طبق سند لایه ۱ (بخش ۰)، این ارتباط از سمت میکروکرنل به‌صورت **فراخوانی مستقیم تابع/trait**، نه IPC است — هر دو لایه در یک باینری Privileged واحد لینک می‌شوند. میکروکرنل با `HardwareManifestRaw` (سند لایه ۱، بخش ۹) در لحظه‌ی بوت مقداردهی می‌شود.

**لایه‌ی بالادستی (زیرسیستم‌های user-space، لایه ۳):** ارتباط از طریق **مرز syscall** است — تنها نقطه‌ای در کل معماری که واقعاً یک context switch سخت‌افزاری (از Ring 3/EL0/U-mode به Ring 0/EL1/M-mode) اتفاق می‌افتد:

```
پروسه‌ی لایه ۳  --[syscall/svc/ecall instruction]-->  میکروکرنل (لایه ۲)
پروسه‌ی لایه ۳  <--[بازگشت با نتیجه در رجیستر]-------  میکروکرنل (لایه ۲)
```

سطح syscall (بخش ۶، `SyscallOp`) تنها API است که لایه ۳ به بالا برای صحبت با میکروکرنل دارد. IPC واقعی بین دو پروسه‌ی لایه ۳ (مثلاً اپ با VFS) هم از همین مسیر عبور می‌کند: هر دو طرف با میکروکرنل صحبت می‌کنند (`ipc_call`/`ipc_send_async`)، میکروکرنل واسطه‌ی انتقال پیام/Capability است، نه این‌که دو پروسه مستقیم به هم دسترسی داشته باشند. کریت `kernel-ipc` (بخش ۷) این مرز را به شکل یک API سطح‌بالاتر برای Rust در لایه ۳ کپسوله می‌کند تا کد بالادست مجبور به نوشتن دستورات trap خام نباشد.



میکروکرنل باید **کوچک‌ترین کد ممکن در Privileged mode** باشد. قاعده‌ی طلایی:

> اگر یک قابلیت می‌تواند در user-space پیاده‌سازی شود، در کرنل نیست.

این لایه فقط چهار مسئولیت دارد و نه یک مورد بیشتر: **مدیریت حافظه، زمان‌بندی، IPC، و Capability**. فایل‌سیستم، درایور، شبکه، حتی خود «پروسه» به آن معنای سنتی — همه در لایه ۳ (user-space) هستند.

## 1.1 تصمیم: اثبات‌پذیری (Formal Verification) از همان روز اول طراحی می‌شود

اثبات صوری کامل (به‌سبک seL4) در scope خود MVP نیست، اما **کد از همان ابتدا با محدودیت‌هایی نوشته می‌شود که مسیر اثبات‌پذیری در فاز ۲ را باز نگه دارد**، نه این‌که بعداً مجبور به بازنویسی شویم:

- ممنوعیت `unsafe` بدون کامنت توجیهی مستند (طبق بخش ۷ سند HAL، همین قاعده اینجا هم اجباری‌تر است چون این لایه Privileged است)
- Syscall dispatcher باید به شکل یک **state machine صریح** نوشته شود؛ هر تابع باید effect محدود و قابل ردیابی داشته باشد، بدون side-effect پنهان یا mutation از طریق global state پراکنده
- ساختار Capability Derivation Tree (بخش ۲) دقیقاً از الگوی اثبات‌شده‌ی seL4 پیروی می‌کند، نه یک طراحی سفارشی — چون این بخش قلب امنیتی سیستم است و ریسک بازطراحی از صفر توجیهی ندارد
- توابع حیاتی امنیتی (grant/revoke Capability، retype حافظه) باید pre/post condition خود را به شکل کامنت‌های ساختاریافته مستند کنند، حتی قبل از این‌که ابزار اثبات صوری (مثل Kani یا Prusti) وارد پروژه شود — این کامنت‌ها بعداً مستقیماً پایه‌ی proof annotation می‌شوند

## 2. مدل Capability — هسته‌ی امنیتی کل سیستم‌عامل

به‌جای UID/GID/permission سنتی لینوکس، هر منبع (حافظه، دستگاه، پورت IPC، توکن HAL-Direct) با یک **Capability** نمایش داده می‌شود:

```rust
/// یک Capability = ارجاع غیرقابل‌جعل به یک Kernel Object + مجموعه‌ای از حقوق (rights)
pub struct Capability {
    pub object_ref: KernelObjectRef,
    pub rights: CapabilityRights,   // bitflags: READ, WRITE, EXECUTE, GRANT, DUPLICATE, REVOKE
    pub badge: u64,                 // شناسه‌ی دلخواه برای تفکیک درخواست‌ها هنگام IPC
}

bitflags::bitflags! {
    pub struct CapabilityRights: u32 {
        const READ     = 0b00001;
        const WRITE    = 0b00010;
        const EXECUTE  = 0b00100;
        const GRANT    = 0b01000; // اجازه‌ی انتقال این Capability به پروسه‌ی دیگر
        const DUPLICATE= 0b10000;
    }
}
```

**قواعد کلیدی:**
- هیچ پروسه‌ای به‌طور پیش‌فرض به هیچ منبعی دسترسی ندارد؛ فقط با داشتن Capability معتبر
- Capabilityها فقط از طریق IPC صریح منتقل می‌شوند (`grant`)، هرگز implicit
- Revocation باید ممکن باشد: کرنل باید بتواند یک Capability و تمام مشتقات آن (children در صورت duplicate) را باطل کند (الگوی CDT — Capability Derivation Tree، مشابه seL4)

## 3. مدیریت حافظه (Memory Management)

میکروکرنل خودش صفحه تخصیص نمی‌دهد به معنای malloc عمومی؛ به‌جایش **حافظه‌ی فیزیکی هم یک Capability است** (`UntypedMemory` — دقیقاً مدل seL4):

- هنگام بوت، کل حافظه‌ی فیزیکی گزارش‌شده در Hardware Manifest (از لایه ۱) به شکل چند شیء `UntypedMemory` به اولین پروسه (Root Task در لایه ۳) داده می‌شود
- Root Task مسئول تقسیم این حافظه بین سرویس‌ها است، نه خود کرنل — این یعنی **سیاست تخصیص حافظه در کرنل نیست**، فقط مکانیزم است
- عملیات اصلی کرنل روی حافظه: `retype` (تبدیل UntypedMemory به یک نوع مشخص مثل PageTable، ThreadControlBlock، Endpoint)، `map`, `unmap`

```rust
pub enum KernelObjectType {
    UntypedMemory,
    PageTable,
    ThreadControlBlock,
    Endpoint,        // برای IPC
    Notification,    // برای async signal
    CapabilitySpace, // جدول Capabilityهای یک پروسه
}
```

## 4. زمان‌بندی (Scheduler)

دو مود که از HAL (بخش Timer) پشتیبانی می‌شود، منطبق بر نیاز پروفایل لایه ۴:

| مود | الگوریتم پایه | کاربرد |
|---|---|---|
| Interactive | Priority-based با aging + کوانتوم کوتاه (~1-4ms) | عمومی، گیمینگ |
| Throughput/Batch | الگوریتم سفارشی (بخش ۴.۱) | AI batch، تخصصی |

### 4.1 تصمیم: الگوریتم سفارشی برای Throughput-mode

به‌جای پیاده‌سازی مستقیم CFS یا EEVDF، یک الگوریتم اختصاصی طراحی می‌شود که از پایه برای مدل dual-mode ما بهینه است (نه یک الگوریتم عمومی که بعداً patch می‌خورد). نکات طراحی اولیه:

- **واحد زمان‌بندی نه thread تنها، بلکه «گروه IPC»**: چون در این معماری هر عملیات معمولاً یک زنجیره‌ی IPC بین چند پروسه است (مثلاً اپ → VFS → درایور)، الگوریتم باید هزینه‌ی کل زنجیره را در نظر بگیرد، نه فقط یک ترد منفرد — این چیزی است که نه CFS نه EEVDF به‌طور بومی برایش طراحی نشده‌اند (چون در لینوکس مونولیتیک این زنجیره اصلاً وجود ندارد).
- **آگاهی از NUMA و Compute Device Affinity** به‌عنوان ورودی اول‌درجه‌ی الگوریتم، نه یک لایه‌ی جانبی روی آن (چیزی که در EEVDF لینوکس هم بعداً و به‌سختی اضافه شده)
- **معیار انتخاب weight**: ترکیبی از priority استاتیک (از Profile Policy) + مدت انتظار در صف (شبیه aging) + یک ضریب «هزینه‌ی IPC زنجیره‌ای» که در بالا اشاره شد
- ریسک این مسیر (طراحی از صفر) باید با benchmark سنگین در برابر پیاده‌سازی مرجع CFS-like روی همان workload مدیریت شود — اگر الگوریتم سفارشی در عمل بهتر از CFS ساده نبود، باید بدون تعصب به CFS-like برگردیم؛ این تصمیم باید data-driven باقی بماند، نه ایدئولوژیک

### 4.3 تصمیم نهایی: فرمول Weight در Throughput-mode

**فرمول پایه (نسخه‌ی اول، پارامترهای عددی‌اش با benchmark واقعی تنظیم می‌شوند، ولی خود ساختار الگوریتم قطعی است):**

```
vruntime_next(thread) = vruntime_current(thread) + (actual_runtime_ns / effective_weight)

effective_weight = base_priority_weight
                  × (1 + aging_factor × min(wait_time_ms, aging_cap_ms))
                  × numa_locality_bonus
```

- **`base_priority_weight`**: از Profile Policy (لایه ۴) می‌آید؛ مثلاً پروفایل AI مقدار پایه‌ی بالاتری به تردهای inference می‌دهد
- **`aging_factor` و `aging_cap_ms`**: جلوگیری از starvation؛ هرچه ترد بیشتر در صف منتظر بماند weight مؤثرش بالاتر می‌رود، ولی سقف دارد تا اولویت‌بندی اصلی خراب نشود
- **`numa_locality_bonus`**: تردی که روی core نزدیک به حافظه/دستگاه محاسباتی‌اش زمان‌بندی می‌شود بونوس می‌گیرد (کاهش vruntime افزایشی)، برای تشویق scheduler به حفظ NUMA locality

**مدل حسابداری زنجیره‌ی IPC:** به‌جای این‌که هر ترد در یک زنجیره‌ی IPC (مثلاً اپ → VFS → درایور) جدا حساب شود، تمام تردهای درگیر در یک زنجیره‌ی synchronous IPC یک **Chain Group ID** مشترک می‌گیرند و `vruntime` به سطح گروه انباشته می‌شود، نه به تک‌تک تردها:

```rust
pub struct ChainGroup {
    pub id: ChainGroupId,
    pub member_threads: Vec<ThreadId>,
    pub group_vruntime: u64,
}
```

این یعنی یک زنجیره‌ی IPC طولانی هزینه‌ی زمان‌بندی‌اش را به‌طور منصفانه بین اعضا تقسیم می‌کند، نه این‌که چون چند ترد را درگیر کرده، دو یا سه برابر سهم از CPU بگیرد (که در مدل‌های ساده مثل CFS خام این مشکل وجود دارد چون IPC اصلاً در طراحی اصلی‌اش دیده نشده بود).

**تعیین تکلیف پارامترهای عددی:** مقادیر اولیه‌ی پیشنهادی برای شروع benchmark: `aging_factor = 0.02`, `aging_cap_ms = 50`, `numa_locality_bonus = 0.9` (کاهش ۱۰٪ در vruntime افزایشی برای دسترسی local). این‌ها نقطه‌ی شروعند، نه عدد نهایی — تنظیم دقیق بخشی از فاز تست عملکردی MVP است، نه یک سوال معماری باز.

### 4.4 نکات عمومی زمان‌بند
- انتخاب مود در سطح **per-thread** است، نه سراسری؛ یک ترد AI inference می‌تواند throughput-mode باشد در حالی که UI همان دستگاه interactive-mode است
- زمان‌بند باید از اطلاعات NUMA topology (از HAL) آگاه باشد برای کاهش هزینه‌ی cross-node memory access
- Priority Inheritance اجباری است (برای جلوگیری از priority inversion در IPC زنجیره‌ای)

## 5. IPC — قلب معماری میکروکرنل

چون همه‌چیز (درایور، فایل‌سیستم، شبکه) در لایه ۳ به‌صورت پروسه‌ی جدا اجرا می‌شود، **کارایی IPC مستقیماً کارایی کل سیستم را تعیین می‌کند**. این حیاتی‌ترین بخش برای بهینه‌سازی است.

### 5.1 دو نوع IPC

```rust
// ۱. Synchronous, small message — برای فراخوانی‌های سریع مثل syscall-like RPC
pub struct Endpoint;
pub fn ipc_call(endpoint: Capability, msg: SmallMessage) -> SmallMessage;
// SmallMessage: حداکثر چند word، مستقیم در رجیستر منتقل می‌شود، بدون کپی حافظه

// ۲. Asynchronous, bulk data — برای انتقال داده‌ی حجیم (مثل بافر AI/GPU)
pub struct Notification;
pub fn ipc_send_async(notif: Capability, shared_buffer: Capability);
```

### 5.2 Zero-Copy برای داده‌ی حجیم
برای بافرهای بزرگ (مثلاً فریم تصویر، تنسور AI)، به‌جای کپی، **Capability به یک ناحیه‌ی حافظه‌ی مشترک (Shared Memory Region)** منتقل می‌شود؛ گیرنده فقط آن را در فضای آدرس خودش map می‌کند. کپی صفر است.

```rust
pub fn create_shared_region(size: usize, rights: CapabilityRights) -> Capability;
pub fn map_shared_region(cap: Capability, at: Option<VirtAddr>) -> VirtAddr;
```

### 5.3 IPC Fast Path
برای Endpointهای پرترافیک (مثلاً بین scheduler و یک درایور real-time)، باید یک "fast path" وجود داشته باشد که context switch را به حداقل می‌رساند — این دقیقاً همان بهینه‌سازی معروف L4 است که هزینه‌ی IPC را از ~میکروثانیه به ~صدها نانوثانیه رساند. این باید benchmark و بخشی از معیار پذیرش باشد (بخش ۸).

## 6. Kernel Object Model

```rust
pub enum SyscallOp {
    Send { endpoint: CapId, msg: SmallMessage },
    Recv { endpoint: CapId },
    Call { endpoint: CapId, msg: SmallMessage },   // Send + Recv اتمیک
    Yield,
    CapGrant { target_thread: CapId, cap: CapId, rights: CapabilityRights },
    CapRevoke { cap: CapId },
    Retype { untyped: CapId, target_type: KernelObjectType, count: usize },
    Map { page_table: CapId, frame: CapId, vaddr: VirtAddr, perms: MapPermissions },
}
```

سطح syscall باید **بسیار کوچک** بماند (seL4 حدود ۱۰-۱۵ syscall دارد؛ هدف ما هم مشابه است، نه نزدیک به ۳۵۰-۴۰۰ syscall لینوکس).

## 7. ساختار پوشه‌بندی

```
kernel/
 ├── kernel-core/        # syscall dispatcher، object model، no_std، معماری‌مستقل
 ├── kernel-mm/          # مدیریت UntypedMemory، PageTable ops
 ├── kernel-sched/       # زمان‌بند، Interactive + Throughput mode
 ├── kernel-ipc/         # Endpoint، Notification، Shared Region، fast path
 ├── kernel-cap/         # Capability model، CDT، revocation
 └── kernel-arch-glue/   # پل بین kernel-core و traitهای hal-core (لایه ۱)
```

## 8. معیار پذیرش MVP
1. بوت روی هر سه معماری با تحویل کنترل از HAL و ساخت اولین `UntypedMemory` objects
2. اجرای موفق Root Task حداقلی که یک ترد دوم می‌سازد و با آن IPC synchronous برقرار می‌کند
3. Benchmark: هزینه‌ی `ipc_call` روی fast path زیر ۵۰۰ نانوثانیه روی سخت‌افزار مرجع (هدف اولیه؛ قابل تنظیم بعد از benchmark واقعی)
4. Zero-copy shared memory بین دو پروسه با اثبات (تست) که هیچ کپی داده‌ای رخ نداده
5. تست Revocation: باطل کردن یک Capability و اثبات این‌که مشتقات آن هم دیگر معتبر نیستند
6. Fuzz testing سطح syscall (چون این لایه attack surface اصلی سیستم است)

## 9. سوالات باز باقیمانده
هیچ سوال معماری بازی در این لایه باقی نمانده. تنظیم دقیق ثابت‌های عددی فرمول weight (بخش ۴.۳) طبیعتاً بخشی از فاز benchmark در پیاده‌سازی است، نه یک تصمیم طراحی معلق.
