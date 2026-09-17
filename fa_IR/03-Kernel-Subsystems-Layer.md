# لایه ۳ — زیرسیستم‌های هسته‌ای (User-Space Services)

> **لایه:** ۳ - زیرسیستم‌های هسته‌ای (User-Space Services)
> **وضعیت:** سند معماری
> **زبان پیاده‌سازی:** Rust برای همه‌ی موارد به‌جز جایی که صراحتاً استثنا ذکر شده
> **نوع اجرا:** User-space، ولی با اولویت زمان‌بندی بالا (near-kernel priority)، هرکدام پروسه‌ی جدا و ایزوله با Capability محدود

---

## 0. ارتباط با لایه‌ی زیرین و بالادستی

**لایه‌ی زیرین (میکروکرنل، لایه ۲):** هر سرویس این لایه (Device Manager، VFS، Netstack، Compositor، mm-service) یک پروسه‌ی معمولی user-space است که فقط از طریق **syscall + Capability** با میکروکرنل صحبت می‌کند (دقیقاً همان مرز توضیح‌داده‌شده در سند لایه ۲، بخش ۰). این سرویس‌ها هیچ امتیاز ویژه‌ای نسبت به یک اپلیکیشن معمولی لایه ۵ ندارند؛ تنها تفاوت‌شان Capabilityهای بیشتری است که Root Task هنگام راه‌اندازی به آن‌ها اعطا کرده (مثلاً Capability دسترسی به MMIO یک دستگاه خاص).

**لایه‌ی بالادستی (سرویس‌های سیستمی، لایه ۴):** ارتباط از طریق **پروتکل IPC سطح‌بالا** است که با IDL مشترک (بخش ۳ همین سند) تعریف شده — نه syscall خام. مثال‌ها:
- POSIX Compat Layer (لایه ۴) وقتی `open()` صدا زده می‌شود، داخلی یک `FsRequest::Open` به VFS Service می‌فرستد
- Package Manager (لایه ۴) برای نصب یک بسته با VFS Service و Device Manager (برای فضای دیسک/دستگاه هدف) IPC می‌زند
- Security Broker (لایه ۴) خودش Capabilityهایی که این لایه صادر می‌کند (مثل دسترسی به یک درایور خاص) را نهایتاً از طریق میکروکرنل (لایه ۲) اعتبارسنجی و منتقل می‌کند، اما تصمیم سیاستی «کی اجازه دارد» در لایه ۴ گرفته می‌شود

نکته‌ی مهم معماری: **لایه ۴ هرگز مستقیم به میکروکرنل (لایه ۲) syscall نمی‌زند** مگر برای عملیات مدیریتی خیلی محدود (مثل صدور Capability که خودش نیازمند دسترسی میکروکرنلی است)؛ مسیر اصلی داده همیشه از طریق پروتکل‌های این لایه (۳) عبور می‌کند.



این لایه شامل تمام «سیستم‌عاملی که همه فکر می‌کنند بخشی از کرنل است» می‌شود، اما در واقعیت هیچ‌کدام Privileged نیستند. هر جزء این لایه یک پروسه‌ی جدا در سطح میکروکرنل است که فقط به Capabilityهایی که صراحتاً به آن داده شده دسترسی دارد.

**اولین پروسه‌ای که میکروکرنل اجرا می‌کند «Root Task» نام دارد** و مسئول راه‌اندازی اولیه‌ی این زیرسیستم‌ها و توزیع Capabilityهای اولیه (از جمله سهم هرکدام از UntypedMemory) است.

## 2. اجزای اصلی

### 2.1 Device Manager (مدیر درایورها)

```
device-manager/
 ├── دریافت Hardware Manifest (از طریق IPC با Root Task که آن را از HAL گرفته)
 ├── راه‌اندازی یک پروسه‌ی جدا برای هر درایور (Driver Process Isolation)
 ├── صدور Capability محدود به هر درایور:
 │    - فقط MMIO region مربوط به همان دستگاه (نه کل حافظه)
 │    - فقط IRQ همان دستگاه
 └── Restart Policy: کرش یک درایور → Device Manager آن را در یک پروسه‌ی جدید
     دوباره بالا می‌آورد بدون این‌که سیستم بخوابد (این مهم‌ترین دستاورد کل معماری در برابر لینوکس مونولیتیک است)
```

**مدل درایورنویسی:**
```rust
#[async_trait]
pub trait DeviceDriver {
    fn probe(manifest_entry: &ComputeDevice) -> Result<Self, DriverError> where Self: Sized;
    async fn handle_irq(&mut self, irq: IrqId);
    async fn handle_request(&mut self, req: DriverRequest) -> DriverResponse;
}
```
هر درایور یک crate جدا در Rust است که این trait را پیاده می‌کند و در یک sandbox پروسه‌ای مستقل کامپایل و اجرا می‌شود.

### 2.2 VFS Service (فایل‌سیستم)

- به‌جای این‌که فایل‌سیستم بخشی از کرنل باشد (مثل لینوکس)، هر فایل‌سیستم (ext4-compat، btrfs-compat، فایل‌سیستم بومی جدید) یک **سرویس جدا** است که پشت یک VFS Router قرار می‌گیرد
- ارتباط بین اپلیکیشن و VFS از طریق IPC (لایه ۲) با یک پروتکل پیام استاندارد (`FsRequest`/`FsResponse`) انجام می‌شود
- برای کارایی: کش صفحه (page cache) به‌صورت Shared Memory Region بین VFS Service و Kernel Memory Manager پیاده می‌شود تا از کپی اضافه جلوگیری شود

```rust
pub enum FsRequest {
    Open { path: PathBuf, flags: OpenFlags },
    Read { handle: FileHandle, offset: u64, len: usize },
    Write { handle: FileHandle, offset: u64, shared_buf: Capability },
    Stat { path: PathBuf },
    // ...
}
```

- **فایل‌سیستم بومی پیشنهادی برای این پروژه:** طراحی جدید با copy-on-write و checksumming (شبیه فلسفه‌ی ZFS/btrfs)، نوشته‌شده کامل در Rust (می‌توان از crateهای موجود مثل الهام‌گیری از پروژه Redox's RedoxFS شروع کرد، ولی برای MVP endianness و on-disk format باید از صفر و مستند تعریف شود)

### 2.3 Network Stack

- پیاده‌سازی TCP/IP کامل در user-space (الگو گرفته از Fuchsia's Netstack یا پروژه‌ی smoltcp در اکوسیستم Rust)
- ارتباط با کارت شبکه از طریق Device Manager + Capability محدود به آن دستگاه
- **تصمیم: پشتیبانی از kernel-bypass مشابه DPDK از همان MVP** (نه فاز ۲) — چون هم پروفایل AI (شبکه‌ی توزیع‌شده‌ی کم‌تاخیر بین گره‌های inference) و هم پروفایل گیمینگ (شبکه‌ی آنلاین کم‌تاخیر) به این وابسته‌اند و افزودن دیرهنگام آن معمولاً یعنی بازطراحی API. مکانیزم: یک اپلیکیشن با Capability ویژه مستقیم بافر NIC را (از طریق `hal-direct`، لایه ۱) map می‌کند و بدون عبور از مسیر معمول IPC/Netstack داده می‌فرستد.

```rust
pub trait KernelBypassNetworking {
    fn request_direct_nic_access(&self, token: CapabilityToken, nic: DeviceId)
        -> Result<DirectNicHandle, NetstackError>;
    fn poll_rx_ring(&self, handle: DirectNicHandle) -> &[RawPacket];
    fn submit_tx_ring(&self, handle: DirectNicHandle, packets: &[RawPacket]);
}
```

این مسیر کاملاً موازی و مستقل از مسیر معمول Netstack (بخش بالا) است؛ اپلیکیشن‌های عمومی همچنان از مسیر استاندارد (با overhead IPC معمول ولی امنیت و ایزوله‌سازی کامل) استفاده می‌کنند، و فقط اپ‌هایی با Capability صریح از Security Broker (لایه ۴) به این مسیر دسترسی دارند.

### 2.4 Compositor Service — جایگاه UI در معماری

این پاسخ به سوال کلیدی «UI کجای معماری است؟» است. تصمیم: **یک سرویس Compositor در همین لایه (زیرساخت مشترک)، در برابر Window Managerها/DEها که در لایه ۵ به‌عنوان اپلیکیشن بومی و قابل‌انتخاب توسط کاربر قرار می‌گیرند.**

```
┌─────────────────────────────────────────┐
│ لایه ۵: DEها/WMها (GNOME-like, KDE-like,   │  ← کاربر آزادانه انتخاب می‌کند
│          tiling WM, یا هیچ‌کدام برای سرور) │
├─────────────────────────────────────────┤
│ لایه ۳: Compositor Service (این بخش)      │  ← زیرساخت مشترک، همیشه یکسان
└─────────────────────────────────────────┘
```

**تصمیم پروتکل:** پروتکل کاملاً بومی و جدید (نه سازگار با Wayland)، چون هدف اصلی معماری بردن مدل Capability تا بالاترین لایه‌ی UI است. اگر روی Wayland سوار شویم، مدل permission قدیمی آن به‌ناچار زیر پروتکل جدید نشت می‌کند. اگر لازم شد بعداً نرم‌افزار لینوکسیِ GTK/Qt هم پشتیبانی شود، از طریق **Linux Compat Runtime (لایه ۵)** انجام می‌شود، نه با آلوده کردن این سرویس زیرساختی به یک پروتکل خارجی.

```rust
pub trait DisplayProtocol {
    fn create_surface(&self, client: ProcessId) -> Result<SurfaceHandle, CompositorError>;
    fn commit_buffer(&self, surface: SurfaceHandle, buf: Capability); // zero-copy، buf از حافظه‌ی GPU
    fn destroy_surface(&self, surface: SurfaceHandle);
    fn input_event_stream(&self, client: ProcessId) -> EventStream;   // async، از طریق Notification (لایه ۲)
    fn output_topology(&self) -> Vec<OutputInfo>;                      // چند مانیتور، رزولوشن، refresh rate
}
```

**ارتباط با Compute Device Discovery (لایه ۱):** Compositor مستقیم با Device Manager (بخش ۲.۱ همین سند) برای دسترسی GPU صحبت می‌کند و از zero-copy shared memory (لایه ۲، بخش ۵.۲) برای انتقال فریم بین اپ و صفحه استفاده می‌کند — بدون کپی اضافه، برخلاف مدل‌های قدیمی‌تر X11.

**ارتباط با Profile Policy (لایه ۴):** این سرویس عضو `services_enabled_by_default` است، اما **در پروفایل AI (سرور headless) پیش‌فرض بارگذاری نمی‌شود** — دقیقاً طبق مدل Discovery+Policy: HAL همچنان GPU را کشف می‌کند (برای مصرف محاسباتی AI)، فقط سرویس نمایش تصویر لازم نیست بار شود.

### 2.5 Memory Manager Service (سطح بالا)

توجه: این با «مدیریت حافظه‌ی خام» در میکروکرنل (لایه ۲) فرق دارد. این سرویس سیاست‌گذار است:
- تصمیم swapping/paging (اگر سیستم swap دارد)
- OOM policy (کدام پروسه در کمبود حافظه قربانی شود) — قابل تنظیم توسط Profile Policy در لایه ۴
- مدیریت Unified Memory برای دستگاه‌های محاسباتی (هماهنگی بین CPU/GPU/NPU memory pool، بر پایه‌ی قابلیت CXL که در Hardware Manifest گزارش شده)

## 3. IPC Protocol Definition — قرارداد بین سرویس‌ها

برای جلوگیری از سردرگمی نسخه‌بندی، تمام پیام‌های بین‌سرویسی این لایه با **IDL مشترک** تعریف می‌شوند (پیشنهاد: فرمتی شبیه Cap'n Proto یا یک IDL سفارشی سبک به‌جای protobuf، برای اجتناب از هزینه‌ی serialization در مسیر پرترافیک). خروجی IDL کد Rust تایپ-امن تولید می‌کند که در `kernel-ipc` (لایه ۲) استفاده می‌شود.

## 4. ساختار پوشه‌بندی

```
subsystems/
 ├── root-task/            # اولین پروسه، توزیع‌کننده‌ی Capability اولیه
 ├── device-manager/        
 ├── drivers/
 │    ├── driver-framework/    # trait مشترک DeviceDriver + sandbox runtime
 │    ├── driver-nvme/
 │    ├── driver-gpu-generic/
 │    ├── driver-npu-generic/
 │    └── ...
 ├── vfs-service/
 │    ├── vfs-router/
 │    ├── fs-native/           # فایل‌سیستم بومی جدید
 │    └── fs-compat-ext4/      # فقط خواندن/نوشتن سازگار، برای مهاجرت داده از لینوکس
 ├── netstack/
 └── mm-service/
```

## 5. معیار پذیرش MVP
1. Root Task با موفقیت Device Manager، VFS Service حداقلی، و یک درایور بلوکی ساده (مثل virtio-blk روی QEMU) را بالا می‌آورد
2. کرش عمدی یک درایور (panic تزریق‌شده در تست) → Device Manager آن را restart می‌کند بدون این‌که بقیه‌ی سیستم متاثر شود (این باید یک تست خودکار در CI باشد، نه فقط ادعا)
3. VFS Service قادر به mount یک فایل‌سیستم ساده و انجام read/write پایه از طریق IPC
4. Netstack قادر به دریافت/ارسال یک بسته‌ی ICMP echo (ping) روی QEMU با virtio-net
4.1. مسیر kernel-bypass: اپلیکیشن تست با Capability ویژه بتواند مستقیم روی rx/tx ring یک NIC مجازی (virtio-net با پشتیبانی بایپس) بدون عبور از Netstack بسته بفرستد و بگیرد؛ benchmark latency آن باید به‌طور محسوس (حداقل ۳۰-۴۰٪) کمتر از مسیر استاندارد باشد
4.2. Compositor Service: یک کلاینت تست بتواند یک surface بسازد، بافر commit کند، و آن بافر بدون کپی اضافه (zero-copy تایید شده با ابزار profiling) روی صفحه نمایش داده شود (حتی خروجی headless/فایل برای MVP کافی است)
5. Benchmark: throughput VFS read/write نسبت به یک سیستم مرجع (فعلاً لینوکس روی همان QEMU) گزارش شود — نباید بیش از ۲۰-۳۰٪ افت کند در MVP (به‌خاطر overhead IPC)

## 6. تصمیمات نهایی‌شده
- **`fs-compat-ext4`:** ابزار مهاجرت یک‌باره‌ی داده از سیستم‌های لینوکسی، نه بخش دائمی معماری. بعد از پایداری فایل‌سیستم بومی، این ماژول می‌تواند به یک CLI مستقل مهاجرت (خارج از هسته‌ی سیستم‌عامل) منتقل شود تا پیچیدگی VFS Router دائمی باقی نماند.
- **DPDK-style kernel-bypass:** از همان MVP (طبق بخش ۲.۳ بالا).
- **Compositor/UI placement:** طبق بخش ۲.۴ بالا حل شد.

## 7. سوالات باز باقیمانده
فعلاً موردی در این لایه باز نمانده؛ در صورت بروز تصمیم جدید حین پیاده‌سازی MVP، اینجا اضافه می‌شود.
