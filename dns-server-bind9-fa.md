# 🌐 آموزش نصب و راه‌اندازی DNS Server با BIND9 در لینوکس

<p align="center">
  <img src="https://img.shields.io/badge/DNS-BIND9-2B7BB9?style=for-the-badge" alt="DNS BIND9">
  <img src="https://img.shields.io/badge/Ubuntu-Debian-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" alt="Ubuntu Debian">
  <img src="https://img.shields.io/badge/AlmaLinux-RHEL%20family-0F4266?style=for-the-badge&logo=almalinux&logoColor=white" alt="AlmaLinux">
  <img src="https://img.shields.io/badge/Level-Beginner-4CAF50?style=for-the-badge" alt="Beginner">
</p>

<p align="center"><em>DNS Server Setup with BIND9 on Linux</em></p>

در این جزوه یک DNS Server کامل را مرحله‌به‌مرحله روی لینوکس راه می‌اندازیم: از نصب BIND9 و ساخت سرور کش، تا تعریف زون مستقیم و معکوس، سرور ثانویه، فایروال و تست. دو مسیر جداگانه داریم: **Ubuntu/Debian** و **AlmaLinux/Rocky** (خانواده RHEL).

> [!NOTE]
> ساختار این جزوه از [آموزش پارس‌پک](https://parspack.com/blog/os/linux/setup-dns-linux) الهام گرفته و دستورها و فایل‌های تنظیم آن برای نسخه‌های فعلی بازبینی و اصلاح شده است. فایل `named.conf.options` هم با کمک یک نمونه‌ی واقعی از GitHub کامل شده است (منابع در انتهای جزوه).

> [!CAUTION]
> تمام مثال‌ها برای شبکه‌ی آزمایشی و ماشین مجازی هستند. سرور DNS را بدون محدودکردن دسترسی (ACL) روی اینترنت باز نگذارید؛ سرور بازِ Recursive می‌تواند در حملات تقویت DNS سوءاستفاده شود.

## 🧭 فهرست مطالب

<!--TOC-->

---

## 📖 ۱. آشنایی با DNS و نقش BIND

سیستم DNS نام‌های قابل‌خواندن، مانند `www.example.com`، را به آدرس IP تبدیل می‌کند و در حالت معکوس هم می‌تواند از IP به نام برسد. نرم‌افزار **BIND9** رایج‌ترین پیاده‌سازی متن‌باز DNS Server در لینوکس است و یک سرور می‌تواند با آن در یکی از سه نقش زیر کار کند:

- &rlm;**Caching / Forwarding:** پاسخ پرس‌وجوها را از سرورهای بالادستی می‌گیرد و مدتی نگه می‌دارد؛ درخواست‌های بعدی سریع‌تر جواب داده می‌شوند.
- &rlm;**Primary (Authoritative):** منبع اصلی اطلاعات یک یا چند زون است و پاسخ «معتبر» (با فلگ `aa` در `dig`) می‌دهد. در متن‌های قدیمی‌تر Master نامیده می‌شود.
- &rlm;**Secondary:** کپی زون را از سرور اصلی می‌گیرد و اگر سرور اصلی از دسترس خارج شود، پاسخ‌گو می‌ماند. در متن‌های قدیمی‌تر Slave نامیده می‌شود.

زون مستقیم (Forward Zone) نام را به IP تبدیل می‌کند و زون معکوس (Reverse Zone) IP را به نام. نام زون معکوس از وارونه‌کردن اکتت‌های شبکه ساخته می‌شود؛ برای شبکه‌ی `192.168.1.0/24` نام آن `1.168.192.in-addr.arpa` است.

رکوردهایی که در این جزوه استفاده می‌کنیم:

<div dir="rtl">

| کاربرد | رکورد |
| --- | --- |
| نگاشت نام به آدرس IPv4 | `A` |
| نگاشت نام به آدرس IPv6 | `AAAA` |
| نام مستعار برای یک نام دیگر | `CNAME` |
| معرفی سرورهای نام (Name Server) یک زون | `NS` |
| نگاشت معکوس IP به نام | `PTR` |
| مشخصات زون: سرور اصلی، ایمیل مدیر، Serial و زمان‌بندی‌ها | `SOA` |
| سرور دریافت ایمیل دامنه | `MX` |

</div>

> [!WARNING]
> در فایل‌های زون، نامِ کامل باید با نقطه تمام شود؛ مثلاً `ns1.example.com.`. اگر نقطه‌ی آخر را نگذارید، BIND نام زون را دوباره به انتهای آن اضافه می‌کند و نتیجه چیزی شبیه `ns1.example.com.example.com.` می‌شود. این یکی از رایج‌ترین خطاهاست.

## 🧰 ۲. پیش‌نیازها و سناریوی آزمایشی

- یک ماشین مجازی لینوکس با **IP ثابت** (برای تنظیم IP ثابت به بخش `nmcli` جزوه‌ی شبکه مراجعه کنید).
- دسترسی `sudo` یا root.
- برای آزمایش، منابع کمی کافی است؛ یک ماشین مجازی کوچک (مثلاً ۱ گیگابایت رم) برای BIND بسنده می‌کند.
- اگر می‌خواهید دامنه‌ی واقعی را از اینترنت به این سرور بدهید، باید NS رکوردها را در پنل رجیستری یا ثبت‌کننده‌ی دامنه تنظیم کنید و پورت 53 هم روی TCP و هم UDP قابل‌دسترس باشد.

> [!NOTE]
> برای DNS کلاسیک روی پورت 53، گواهی SSL/TLS لازم نیست. رمزنگاری پرس‌وجوها (DoT و DoH) موضوع جداگانه‌ای است و برای راه‌اندازی اولیه ضرورتی ندارد.

مقادیر سناریوی آزمایشی در مثال‌های Ubuntu:

<div dir="rtl">

| نقش | نام | آدرس |
| --- | --- | --- |
| سرور اصلی (Primary) | `ns1.example.com` | `192.168.1.10` |
| سرور ثانویه (Secondary) | `ns2.example.com` | `192.168.1.11` |
| وب‌سرور نمونه | `www.example.com` | `192.168.1.20` |
| شبکه‌ی محلی | | `192.168.1.0/24` |

</div>

> [!IMPORTANT]
> نام و آدرس‌های بالا را با مقادیر خودتان عوض کنید. برای شبکه‌ی داخلی از پسوند `.local` استفاده نکنید؛ این پسوند برای mDNS رزرو شده و ممکن است تداخل ایجاد کند. دامنه‌ی `example.com` هم برای مستندات رزرو شده و برای آزمایش مناسب است.

---

## 🐧 ۳. نصب DNS روی Ubuntu و Debian

### گام ۱: نصب BIND9 و ابزارهای تست

```bash
sudo apt update
sudo apt install bind9 dnsutils
```

بسته‌ی `dnsutils` ابزارهایی مانند `dig` را برای تست دارد و ابزارهای بررسی تنظیمات (`named-checkconf` و `named-checkzone`) هم همراه BIND9 نصب می‌شوند. وضعیت سرویس و نسخه را ببینید:

```bash
named -v
sudo systemctl status bind9 --no-pager
```

نام سرویس در Ubuntu/Debian هم `bind9` و هم `named` است و هر دو کار می‌کنند.

### گام ۲: سرور کش و Forwarder با `named.conf.options`

این فایل تنظیمات عمومی BIND را نگه می‌دارد. ابتدا از فایل اصلی نسخه‌ی پشتیبان بگیرید و پوشه‌ی لاگ را بسازید:

```bash
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
sudo mkdir -p /var/log/named
sudo chown bind:bind /var/log/named
```

سپس فایل را باز کنید:

```bash
sudo nano /etc/bind/named.conf.options
```

و محتوای آن را با نسخه‌ی زیر جایگزین کنید:

```conf
// /etc/bind/named.conf.options - general options for BIND9

// Clients allowed to query and use this server as a resolver
acl "trusted" {
    localhost;
    192.168.1.0/24;          // your LAN
};

options {
    directory "/var/cache/bind";
    dump-file "/var/cache/bind/named_dump.db";

    // By default only trusted clients may ask questions.
    // For a public authoritative zone, override this inside the zone block.
    allow-query { trusted; };

    // Recursive resolution is limited to trusted clients (avoid an open resolver)
    recursion yes;
    allow-recursion { trusted; };

    // Zone transfers are denied by default.
    // They are allowed per zone for the secondary server (see step 4).
    allow-transfer { none; };
    notify yes;

    // Optional: send unknown names to upstream resolvers.
    // Remove this block to let BIND resolve from the root servers itself.
    forwarders {
        1.1.1.1;
        8.8.8.8;
    };
    forward first;

    // DNSSEC validation using the built-in trust anchors
    dnssec-validation auto;

    // Conform to RFC 1035
    auth-nxdomain no;

    // Do not reveal the BIND version
    version "none";

    // IPv6 support
    listen-on-v6 { any; };
};

// Logs go to /var/log/named/named.log and to syslog.
// The directory must be /var/log/named/ because the default
// AppArmor profile of named only allows writing there.
logging {
    channel default_out {
        file "/var/log/named/named.log" versions 7 size 100m;
        print-time yes;
        print-category yes;
        print-severity yes;
    };

    category default      { default_out; default_syslog; };
    category config       { default_out; default_syslog; };
    category xfer-in      { default_out; default_syslog; };
    category xfer-out     { default_out; default_syslog; };
    category lame-servers { null; };
};
```

> [!IMPORTANT]
> این فایل بر پایه‌ی نمونه‌ی [`named.conf.options` در مخزن JoeKun/debian-configuration-files](https://github.com/JoeKun/debian-configuration-files/blob/main/etc/bind/named.conf.options) ساخته شده، ولی عیناً کپی نشده است. در آن نمونه از ACL با نام `myself` استفاده شده که در BIND به‌طور پیش‌فرض وجود ندارد و باید جداگانه تعریف شود؛ ما به‌جای آن ACL با نام `trusted` را تعریف کردیم. مسیر لاگ هم به `/var/log/named/` منتقل شد، چون AppArmor در Ubuntu نوشتن در مسیرهایی مثل `/var/log/named.log` را مسدود می‌کند و سرویس ممکن است بالا نیاید.

حالا قبل از ری‌استارت، تنظیمات را بررسی کنید:

```bash
sudo named-checkconf
sudo systemctl restart bind9
sudo systemctl status bind9 --no-pager
```

اگر `named-checkconf` هیچ خروجی نداد، نحو فایل درست است. برای آزمایش کش، یک نام را دو بار پرس‌وجو کنید و زمان پاسخ را مقایسه کنید:

```bash
dig @127.0.0.1 ubuntu.com | grep "Query time"
dig @127.0.0.1 ubuntu.com | grep "Query time"
```

نمونه خروجی:

```text
;; Query time: 85 msec
;; Query time: 0 msec
```

زمان پاسخ دوم باید به‌طور محسوسی کمتر باشد؛ یعنی پاسخ از کش آمده است. تا اینجا سرور کش (Caching/Forwarding) آماده است.

### گام ۳: سرور اصلی (Primary) برای دامنه

ابتدا زون‌ها را در فایل `named.conf.local` تعریف کنید:

```bash
sudo nano /etc/bind/named.conf.local
```

```conf
// Forward zone
zone "example.com" {
    type primary;
    file "/etc/bind/db.example.com";
};

// Reverse zone for 192.168.1.0/24
zone "1.168.192.in-addr.arpa" {
    type primary;
    file "/etc/bind/db.192.168.1";
};
```

سپس فایل زون مستقیم را بسازید:

```bash
sudo nano /etc/bind/db.example.com
```

```dns
$TTL    86400
@       IN      SOA     ns1.example.com. admin.example.com. (
                        2026101201      ; Serial (YYYYMMDDNN)
                        3600            ; Refresh
                        900             ; Retry
                        1209600         ; Expire
                        86400 )         ; Negative cache TTL

; Name servers
@       IN      NS      ns1.example.com.
@       IN      NS      ns2.example.com.

; Addresses
ns1     IN      A       192.168.1.10
ns2     IN      A       192.168.1.11
@       IN      A       192.168.1.20
www     IN      A       192.168.1.20

; Alias
ftp     IN      CNAME   www.example.com.
```

چند نکته درباره‌ی این فایل:

- &rlm;`admin.example.com.` در SOA یعنی ایمیل `admin@example.com`؛ اولین نقطه جای `@` است.
- فیلد اول SOA باید همان سرور اصلی زون (`ns1.example.com.`) باشد.
- &rlm;`@` به نام خود زون اشاره می‌کند، یعنی `example.com.`.
- &rlm;Serial را به‌صورت `YYYYMMDDNN` بنویسید تا افزایش آن راحت باشد.

> [!WARNING]
> هر بار فایل زون را تغییر می‌دهید، **Serial را افزایش دهید**. سرور ثانویه فقط وقتی زون را دوباره می‌گیرد که Serial بزرگ‌تر شده باشد.

حالا فایل زون معکوس:

```bash
sudo nano /etc/bind/db.192.168.1
```

```dns
$TTL    86400
@       IN      SOA     ns1.example.com. admin.example.com. (
                        2026101201      ; Serial
                        3600            ; Refresh
                        900             ; Retry
                        1209600         ; Expire
                        86400 )         ; Negative cache TTL

; Name servers (full names, ending with a dot)
@       IN      NS      ns1.example.com.
@       IN      NS      ns2.example.com.

; PTR records: last octet of the IP -> name
10      IN      PTR     ns1.example.com.
11      IN      PTR     ns2.example.com.
20      IN      PTR     www.example.com.
```

برای هر رکورد `A` که در زون مستقیم دارید، یک `PTR` در زون معکوس بنویسید. فایل‌ها را بررسی کنید و سرویس را ری‌استارت کنید:

```bash
sudo named-checkconf -z
sudo named-checkzone example.com /etc/bind/db.example.com
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192.168.1
sudo systemctl restart bind9
```

خروجی سالم `named-checkzone` چنین است:

```text
zone example.com/IN: loaded serial 2026101201
OK
```

### گام ۴: سرور ثانویه (Secondary)

روی **سرور اصلی**، به هر دو زون در `/etc/bind/named.conf.local` اجازه‌ی انتقال به آدرس سرور ثانویه را بدهید:

```conf
zone "example.com" {
    type primary;
    file "/etc/bind/db.example.com";
    allow-transfer { 192.168.1.11; };
};

zone "1.168.192.in-addr.arpa" {
    type primary;
    file "/etc/bind/db.192.168.1";
    allow-transfer { 192.168.1.11; };
};
```

```bash
sudo named-checkconf
sudo systemctl restart bind9
```

روی **سرور ثانویه** (`192.168.1.11`) هم BIND9 را نصب کنید و فایل `named.conf.options` همان گام ۲ را (با همان ACL) به‌کار ببرید؛ لازم نیست Forwarder داشته باشد. سپس در `/etc/bind/named.conf.local` این زون‌ها را به‌صورت Secondary تعریف کنید:

```conf
zone "example.com" {
    type secondary;
    file "db.example.com";
    primaries { 192.168.1.10; };
};

zone "1.168.192.in-addr.arpa" {
    type secondary;
    file "db.192.168.1";
    primaries { 192.168.1.10; };
};
```

```bash
sudo named-checkconf
sudo systemctl restart bind9
```

نام فایل‌ها نسبی است و کپی زون در `/var/cache/bind/` ذخیره می‌شود؛ AppArmor نوشتن در همین مسیر را اجازه می‌دهد. این فایل‌ها معمولاً به‌صورت باینری ذخیره می‌شوند و برای خواندن مناسب نیستند. برای بررسی انتقال:

```bash
ls -l /var/cache/bind/
sudo rndc zonestatus example.com
dig @192.168.1.11 example.com SOA +short
```

اگر زون منتقل نشد، می‌توانید دوباره آن را درخواست کنید:

```bash
sudo rndc retransfer example.com
```

> [!NOTE]
> ریشه‌ی `type master` و `type slave` در نسخه‌های قدیمی همین دو نقش هستند. نسخه‌های جدید BIND هر دو نام را می‌پذیرند، ولی `primary` و `secondary` نام‌گذاری امروزی است.

### گام ۵: فایروال

اگر `ufw` فعال است، پورت 53 را باز کنید. DNS معمولاً با UDP کار می‌کند، اما پاسخ‌های بزرگ و انتقال زون از TCP استفاده می‌کنند؛ پس هر دو پروتکل لازم‌اند:

```bash
sudo ufw status
sudo ufw allow 53
```

> [!TIP]
> دستور `sudo ufw allow 53` هر دو پروتکل TCP و UDP را باز می‌کند. اگر می‌خواهید فقط شبکه‌ی داخلی دسترسی داشته باشد، از `sudo ufw allow from 192.168.1.0/24 to any port 53` استفاده کنید.

### گام ۶: تست

از روی سرور یا یک کلاینت هم‌شبکه، پرس‌وجوها را مستقیم به سرور بفرستید:

```bash
dig @192.168.1.10 www.example.com
dig @192.168.1.10 -x 192.168.1.20
dig @192.168.1.10 example.com NS +short
```

نمونه خروجی بخش مهم پرس‌وجوی اول:

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4821
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; ANSWER SECTION:
www.example.com.    86400   IN  A   192.168.1.20

;; SERVER: 192.168.1.10#53(192.168.1.10)
```

- &rlm;`status: NOERROR`: پرس‌وجو موفق بوده است.
- &rlm;`aa`: پاسخ از سرور معتبر همان زون آمده است.
- &rlm;`ANSWER SECTION`: همان رکوردی که ساخته بودید، باید با IP درست دیده شود.

برای اینکه خود سیستم از این سرور استفاده کند، DNS اتصال را عوض کنید. در Ubuntu فایل `/etc/resolv.conf` معمولاً توسط `systemd-resolved` مدیریت می‌شود و ویرایش دستی آن پایدار نمی‌ماند؛ بهتر است DNS را در خود اتصال تنظیم کنید:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.dns "192.168.1.10 192.168.1.11"
sudo nmcli connection up "Wired connection 1"
resolvectl status
resolvectl query www.example.com
```

---

## 🎩 ۴. نصب DNS روی AlmaLinux و Rocky (خانواده RHEL)

در این بخش از مقادیر زیر استفاده می‌کنیم. چون این بخش مستقل از بخش Ubuntu است، آدرس‌ها متفاوت‌اند:

<div dir="rtl">

| مورد | مقدار |
| --- | --- |
| آدرس IP سرور | `192.168.1.100` |
| شبکه‌ی محلی | `192.168.1.0/24` |
| نام سرور | `ns1.example.com` |
| وب‌سرور نمونه | `www.example.com` با آدرس `192.168.1.101` |

</div>

> [!WARNING]
> CentOS 7 و CentOS 8 پایان پشتیبانی شده‌اند و دیگر به‌روزرسانی امنیتی نمی‌گیرند. برای سرور تازه از AlmaLinux یا Rocky Linux استفاده کنید؛ دستورها یکسان‌اند و فقط `yum` جای خود را به `dnf` داده است.

### گام ۱: به‌روزرسانی سیستم

```bash
sudo dnf update -y
```

### گام ۲: نصب و فعال‌سازی BIND

```bash
sudo dnf install -y bind bind-utils
sudo systemctl enable --now named
sudo systemctl status named --no-pager
```

### گام ۳: پیکربندی گوش‌دادن و دسترسی در `/etc/named.conf`

به‌طور پیش‌فرض BIND فقط روی `localhost` گوش می‌دهد. فایل را باز کنید:

```bash
sudo cp /etc/named.conf /etc/named.conf.bak
sudo nano /etc/named.conf
```

در بلوک `options`، خطوط مربوط را پیدا کنید و مطابق نمونه تغییر دهید (اگر با `//` کامنت شده‌اند، علامت را بردارید):

```conf
options {
    listen-on port 53 { 127.0.0.1; 192.168.1.100; };
    listen-on-v6 port 53 { ::1; };

    allow-query     { localhost; 192.168.1.0/24; };
    allow-recursion { localhost; 192.168.1.0/24; };
    recursion yes;

    // ... keep the rest of the default options ...
};
```

> [!TIP]
> در این فایل، `listen-on` مشخص می‌کند BIND روی کدام آدرس‌ها گوش بدهد و `allow-query` و `allow-recursion` تعیین می‌کنند چه کسانی اجازه‌ی پرس‌وجو و Recursion دارند. با محدودکردن آن‌ها به شبکه‌ی داخلی، از تبدیل شدن سرور به Open Resolver جلوگیری می‌کنید.

### گام ۴: تعریف زون‌های مستقیم و معکوس

تعریف زون‌ها را به انتهای `/etc/named.conf` اضافه کنید:

```conf
// Forward zone
zone "example.com" IN {
    type primary;
    file "example.com.zone";
    allow-update { none; };
};

// Reverse zone for 192.168.1.0/24
zone "1.168.192.in-addr.arpa" IN {
    type primary;
    file "192.168.1.zone";
    allow-update { none; };
};
```

در RHEL نام فایل‌ها نسبت به پوشه‌ی `/var/named` سنجیده می‌شود.

### گام ۵: ساخت فایل‌های زون

زون مستقیم:

```bash
sudo nano /var/named/example.com.zone
```

```dns
$TTL 86400
@   IN  SOA     ns1.example.com. admin.example.com. (
                2026101201      ; Serial (YYYYMMDDNN)
                3600            ; Refresh
                900             ; Retry
                1209600         ; Expire
                86400 )         ; Negative cache TTL

; Name server
@       IN  NS      ns1.example.com.

; Address of the name server
ns1     IN  A       192.168.1.100

; Host records
www     IN  A       192.168.1.101

; Alias
ftp     IN  CNAME   www.example.com.
```

زون معکوس:

```bash
sudo nano /var/named/192.168.1.zone
```

```dns
$TTL 86400
@   IN  SOA     ns1.example.com. admin.example.com. (
                2026101201      ; Serial
                3600            ; Refresh
                900             ; Retry
                1209600         ; Expire
                86400 )         ; Negative cache TTL

; Name server (full name, ending with a dot)
@       IN  NS      ns1.example.com.

; PTR records
100     IN  PTR     ns1.example.com.
101     IN  PTR     www.example.com.
```

مالکیت و دسترسی فایل‌ها را درست کنید تا کاربر `named` بتواند آن‌ها را بخواند، و برچسب SELinux را هم اصلاح کنید:

```bash
sudo chown root:named /var/named/example.com.zone /var/named/192.168.1.zone
sudo chmod 640 /var/named/example.com.zone /var/named/192.168.1.zone
sudo restorecon -v /var/named/example.com.zone /var/named/192.168.1.zone
```

### گام ۶: بررسی تنظیمات

```bash
sudo named-checkconf /etc/named.conf
sudo named-checkzone example.com /var/named/example.com.zone
sudo named-checkzone 1.168.192.in-addr.arpa /var/named/192.168.1.zone
```

اگر تنظیمات درست باشند، `named-checkconf` چیزی چاپ نمی‌کند و `named-checkzone` در پایان `OK` نشان می‌دهد. سپس سرویس را ری‌استارت کنید:

```bash
sudo systemctl restart named
```

> [!WARNING]
> نام زون و مسیر فایل در دستور `named-checkzone` باید دقیقاً با چیزی که در `named.conf` نوشته‌اید یکی باشد. اگر نام زون یا فایل را اشتباه بدهید، خطا یا پیام «file not found» می‌گیرید.

### گام ۷: فایروال

```bash
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

سرویس `dns` در `firewalld` هر دو پورت UDP و TCP شماره 53 را باز می‌کند. باز کردن فقط UDP کافی نیست، چون پاسخ‌های بزرگ و انتقال زون از TCP استفاده می‌کنند.

### گام ۸: تست

```bash
dig @192.168.1.100 www.example.com +short
dig @192.168.1.100 ns1.example.com
dig @192.168.1.100 -x 192.168.1.101 +short
```

در خروجی کامل، `status: NOERROR`، فلگ `aa` و بخش `ANSWER SECTION` با IP درست، نشانه‌ی سالم بودن سرور هستند.

برای استفاده‌ی خود سرور یا کلاینت از این DNS، DNS را در تنظیمات اتصال بدهید. در خانواده RHEL فایل `/etc/resolv.conf` معمولاً توسط NetworkManager بازنویسی می‌شود:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.dns "192.168.1.100"
sudo nmcli connection up "Wired connection 1"
cat /etc/resolv.conf
```

> [!NOTE]
> برای سرور ثانویه در خانواده RHEL، همان تنظیم‌های `allow-transfer` و `type secondary` گام ۴ بخش Ubuntu را به‌کار ببرید، با این تفاوت که فایل زون را در `/var/named/slaves/` ذخیره کنید، مثلاً `file "slaves/example.com.zone";`. این پوشه برای نوشتن توسط `named` تنظیم شده است.

---

## 🧪 ۵. تست و عیب‌یابی

دستورهای مفید برای بررسی وضعیت سرویس:

```bash
sudo systemctl status bind9 --no-pager
sudo journalctl -u bind9 -n 50 --no-pager
sudo ss -lntup 'sport = :53'
sudo rndc status
sudo rndc reload
sudo named-checkconf -z
```

نام سرویس در RHEL، `named` است. دستور `named-checkconf -z` علاوه بر نحو، بارگذاری زون‌ها را هم امتحان می‌کند و بهترین دستور برای کشف اشتباه‌های فایل زون پیش از ری‌استارت است.

خطاهای رایج:

<div dir="rtl">

| نشانه | علت محتمل | راه‌حل |
| --- | --- | --- |
| پیام `connection timed out; no servers could be reached` | سرویس خاموش است، IP اشتباه است یا فایروال مانع می‌شود | با `ss -lntup` گوش‌دادن روی پورت 53 را ببینید و فایروال را بررسی کنید |
| وضعیت `REFUSED` در خروجی `dig` | آدرس کلاینت در `allow-query` یا `allow-recursion` نیست | شبکه‌ی کلاینت را به ACL اضافه کنید |
| وضعیت `SERVFAIL` | زون بالا نیامده یا مشکلی در پاسخ بالادستی است | `named-checkconf -z` و لاگ سرویس را ببینید |
| وضعیت `NXDOMAIN` برای نامی که ساخته‌اید | رکورد ننوشته شده یا نقطه‌ی انتهای نام جا افتاده | فایل زون را بازبینی کنید و Serial را بالا ببرید |
| سرور ثانویه زون را نگرفته | Serial افزایش نیافته، `allow-transfer` اشتباه است یا TCP/53 بسته است | `rndc retransfer` را امتحان کنید و فایروال و لاگ را ببینید |
| خطای `permission denied` در لاگ | مجوز فایل، AppArmor یا SELinux | مالکیت و مسیر فایل را درست کنید؛ در Ubuntu لاگ را در `/var/log/named/` بنویسید |

</div>

## 🔐 ۶. نکات امنیتی

> [!CAUTION]
> سرور Recursive باز، یعنی سروری که برای همه‌ی اینترنت پرس‌وجو را حل می‌کند، می‌تواند در حملات تقویت DNS استفاده شود. همیشه `allow-recursion` را به شبکه‌ی خودتان محدود کنید.

- &rlm;`allow-transfer` را فقط برای IP سرورهای ثانویه باز کنید و در تنظیم عمومی روی `none` بگذارید.
- نسخه‌ی BIND را با `version "none";` پنهان کنید تا حدس زدن آسیب‌پذیری‌ها دشوارتر شود.
- &rlm;`dnssec-validation auto;` را فعال نگه دارید تا پاسخ‌های امضاشده اعتبارسنجی شوند.
- در محیط واقعی، سرور معتبر (Authoritative) و سرور Recursive را جدا کنید و BIND را مرتب به‌روز نگه دارید.
- فایل‌های تنظیمات را پیش از ویرایش پشتیبان بگیرید و همیشه قبل از ری‌استارت `named-checkconf` را اجرا کنید.

## 📋 ۷. برگه تقلب سریع

<div dir="rtl">

| هدف | دستور |
| --- | --- |
| 📦 نصب در Ubuntu/Debian | `sudo apt install bind9 dnsutils` |
| 📦 نصب در AlmaLinux/Rocky | `sudo dnf install bind bind-utils` |
| ✅ بررسی تنظیمات و زون‌ها | `sudo named-checkconf -z` |
| 📄 بررسی یک فایل زون | `sudo named-checkzone example.com /etc/bind/db.example.com` |
| 🔄 بارگذاری مجدد زون‌ها | `sudo rndc reload` |
| ♻️ ری‌استارت سرویس در Ubuntu | `sudo systemctl restart bind9` |
| ♻️ ری‌استارت سرویس در RHEL | `sudo systemctl restart named` |
| 👂 بررسی گوش‌دادن روی پورت 53 | `sudo ss -lntup 'sport = :53'` |
| 🔎 پرس‌وجوی رکورد A | `dig @192.168.1.10 www.example.com +short` |
| 🔁 پرس‌وجوی معکوس | `dig @192.168.1.10 -x 192.168.1.20 +short` |
| 🗂 مشاهده سرورهای نام | `dig @192.168.1.10 example.com NS +short` |
| 🚚 تست انتقال زون | `dig @192.168.1.10 example.com AXFR` |
| 🧱 باز کردن پورت در Ubuntu | `sudo ufw allow 53` |
| 🧱 باز کردن پورت در RHEL | `sudo firewall-cmd --permanent --add-service=dns` |

</div>

## 📝 تمرین‌های پیشنهادی

1. در یک ماشین مجازی BIND9 را نصب کنید و با دو بار `dig` روی یک نام اینترنتی، اثر کش را ببینید.
2. زون `example.com` را با رکوردهای `A`، `CNAME` و `PTR` بسازید و با `named-checkzone` بررسی کنید.
3. عمداً نقطه‌ی انتهای یک نام را حذف کنید و خروجی `dig` را ببینید؛ سپس اشتباه را اصلاح کنید.
4. یک سرور ثانویه راه بیندازید و Serial را بالا ببرید تا انتقال زون را مشاهده کنید.
5. از کلاینتی که در `allow-transfer` نیست، `AXFR` بزنید و مطمئن شوید رد می‌شود.
6. شبکه‌ی کلاینت را از ACL حذف کنید و خطای `REFUSED` را بازتولید کنید.

## 📚 منابع

- [آموزش نصب DNS Server در لینوکس (گام‌به‌گام) در بلاگ پارس‌پک](https://parspack.com/blog/os/linux/setup-dns-linux)
- [نمونه‌ی فایل `named.conf.options` در مخزن JoeKun/debian-configuration-files](https://github.com/JoeKun/debian-configuration-files/blob/main/etc/bind/named.conf.options)
