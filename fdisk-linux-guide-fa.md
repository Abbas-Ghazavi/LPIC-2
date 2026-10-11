<h1 dir="rtl" align="right">آموزش <code>fdisk</code>، پارتیشن‌بندی و رمزنگاری دیسک در لینوکس</h1>

<p dir="rtl" align="right">این یادداشت برای مرور دستورات مدیریت دیسک در لینوکس و تمرین مباحث مرتبط با LPIC تهیه شده است. مثال‌ها آموزشی هستند و خروجی‌ها ممکن است با توجه به توزیع لینوکس، نسخه ابزارها و دیسک شما متفاوت باشند.</p>

<blockquote dir="rtl" align="right">
<p>⚠️ <strong>هشدار مهم:</strong> پارتیشن‌بندی، ساخت فایل‌سیستم و رمزنگاری می‌توانند اطلاعات را از بین ببرند. دستورات این راهنما را روی ماشین مجازی و دیسک آزمایشی اجرا کنید. قبل از هر تغییری، نام دیسک را چند بار بررسی کنید و از داده‌های مهم نسخه پشتیبان داشته باشید. در مثال‌ها <code>/dev/sdb</code> یک دیسک آزمایشی است؛ آن را کورکورانه کپی نکنید.</p>
</blockquote>

<h2 dir="rtl" align="right">۱. مشاهده دیسک‌ها و پارتیشن‌ها با <code>lsblk</code></h2>

<p dir="rtl" align="right">ابتدا دیسک‌ها، پارتیشن‌ها، نوع دستگاه و محل Mount را بررسی می‌کنیم:</p>

``` bash
lsblk
```

<p dir="rtl" align="right">نمونه خروجی:</p>

``` text
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0                 11:0    1  9.4G  0 rom  /run/media/user/AlmaLinux
nvme0n1            259:0    0   20G  0 disk
├─nvme0n1p1        259:1    0    1M  0 part
├─nvme0n1p2        259:2    0    2G  0 part /boot
└─nvme0n1p3        259:3    0   18G  0 part
  ├─almalinux-root 253:0    0   16G  0 lvm  /
  └─almalinux-swap 253:1    0    2G  0 lvm  [SWAP]
```

<p dir="rtl" align="right">ستون <code>TYPE</code> مشخص می‌کند مورد نمایش‌داده‌شده دیسک (<code>disk</code>)، پارتیشن (<code>part</code>)، یا دستگاهی مانند LVM است. ستون <code>MOUNTPOINTS</code> محل استفاده از فایل‌سیستم را نشان می‌دهد.</p>

<p dir="rtl" align="right">برای دیدن اطلاعات دقیق‌تر دیسک و جدول پارتیشن‌ها:</p>

``` bash
sudo fdisk -l /dev/sdb
```

<p dir="rtl" align="right"><strong>نکته:</strong> در مثال بالا، <code>nvme0n1</code> دیسک کامل و <code>nvme0n1p3</code> یک پارتیشن از آن است. برای تمرین ساخت پارتیشن جدید، دیسک آزمایشیِ خالی مانند <code>/dev/sdb</code> را انتخاب کنید؛ پارتیشن ریشه یا پارتیشنی را که سیستم از آن استفاده می‌کند تغییر ندهید.</p>

<h2 dir="rtl" align="right">۲. ورود به محیط تعاملی <code>fdisk</code></h2>

<p dir="rtl" align="right">برای بازکردن دیسک آزمایشی:</p>

``` bash
sudo fdisk /dev/sdb
```

<p dir="rtl" align="right">نمونه پیام آغازین:</p>

``` text
Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help):
```

<p dir="rtl" align="right">تغییرات تا زمانی که <code>w</code> را اجرا نکنید روی دیسک نوشته نمی‌شوند. بااین‌حال، قبل از ثبت تغییرات، خروجی‌ها را با دقت بررسی کنید.</p>

<p dir="rtl" align="right">اگر ابزار هشدار دهد که دیسک در حال استفاده است، ادامه ندهید مگر اینکه دقیقاً بدانید چه می‌کنید. معمولاً فایل‌سیستم‌ها باید Unmount و Swap مربوطه خاموش شده باشد؛ روی دیسک سیستم‌عامل این کار را در حال اجرای همان سیستم انجام ندهید.</p>

<h2 dir="rtl" align="right">۳. دیدن جدول پارتیشن با <code>p</code> و راهنمای <code>m</code></h2>

<p dir="rtl" align="right">داخل محیط <code>fdisk</code> حرف <code>p</code> جدول پارتیشن را نمایش می‌دهد:</p>

``` text
Command (m for help): p

Disk /dev/sdb: 10 GiB, ...
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x12345678
```

<p dir="rtl" align="right">در این خروجی، <code>Disklabel type: dos</code> نشان‌دهنده جدول پارتیشن MBR/DOS است. اگر <code>gpt</code> دیده شود، جدول از نوع GPT است.</p>

<p dir="rtl" align="right">برای دیدن فرمان‌های موجود:</p>

``` text
Command (m for help): m
```

<p dir="rtl" align="right">فرمان‌های پرکاربرد:</p>

  فرمان   کاربرد
<p dir="rtl" align="right">------- --------------------------------- <code>p</code>     نمایش جدول پارتیشن <code>n</code>     ساخت پارتیشن جدید <code>d</code>     حذف پارتیشن <code>t</code>     تغییر نوع پارتیشن <code>l</code>     نمایش نوع‌های شناخته‌شده <code>F</code>     نمایش فضای آزادِ پارتیشن‌بندی‌نشده <code>v</code>     بررسی جدول پارتیشن <code>w</code>     ثبت تغییرات و خروج <code>q</code>     خروج بدون ثبت تغییرات</p>

<p dir="rtl" align="right">فرمان‌های قابل نمایش ممکن است بر اساس نوع جدول پارتیشن متفاوت باشند.</p>

<h2 dir="rtl" align="right">۴. ساخت پارتیشن در MBR/DOS</h2>

<p dir="rtl" align="right">MBR به‌طور معمول چهار ورودی پارتیشن دارد. این چهار ورودی می‌توانند پارتیشن Primary باشند؛ اما اگر یکی از ورودی‌ها به‌صورت Extended تعریف شود، همان ورودی نقش ظرفی برای Logical Partitionها را دارد. شماره پارتیشن‌های Logical معمولاً از ۵ شروع می‌شود. تعداد و اندازه واقعی آن‌ها به فضای دیسک و محدودیت‌های ابزار بستگی دارد.</p>

<p dir="rtl" align="right">ابتدا دیسک آزمایشی را باز کنید و نوع جدول را با <code>p</code> بررسی کنید. اگر جدول آن <code>dos</code> است، برای ساخت پارتیشن:</p>

``` text
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (..., default ...):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (..., default ...): +2G
Created a new partition 1 of type 'Linux' and of size 2 GiB.
```

<p dir="rtl" align="right">مراحل:</p>

<p dir="rtl" align="right">1.  <code>n</code> را برای ساخت پارتیشن وارد کنید. 2.  <code>p</code> را برای Primary انتخاب کنید. 3.  شماره پارتیشن را انتخاب کنید؛ در تمرین ساده می‌توان مقدار پیش‌فرض را پذیرفت. 4.  در پرسش <code>First sector</code> معمولاً با فشردن Enter مقدار پیش‌فرضِ هم‌تراز انتخاب می‌شود. 5.  در <code>Last sector</code> مقدار <code>+2G</code> یعنی اندازه‌ای حدود ۲ GiB درخواست می‌کنید.</p>

<p dir="rtl" align="right">اگر چنین پیامی دیدید:</p>

``` text
Partition #1 contains a swap signature.
Do you want to remove the signature? [Y]es/[N]o:
```

<p dir="rtl" align="right">یعنی <code>fdisk</code> اثری از امضای Swap قدیمی در محدوده پارتیشن پیدا کرده است. این امضا ممکن است باقی‌مانده استفاده قبلی از همان محدوده باشد. فقط اگر مطمئن هستید داده یا ساختار قبلی دیگر لازم نیست، حذف آن را تأیید کنید؛ در غیر این صورت توقف کنید و وضعیت دیسک را بررسی کنید. این پیام به‌تنهایی به معنی فعال‌بودن Swap نیست.</p>

<h3 dir="rtl" align="right">ساخت Extended و Logical در MBR</h3>

<p dir="rtl" align="right">اگر جدول MBR از قبل یک Primary داشته باشد و بخواهید چند پارتیشن Logical بسازید، ابتدا یک ورودی Extended ایجاد می‌کنید. این ورودی فضای ظرف را اشغال می‌کند و فایل‌سیستم معمولی روی خود آن ساخته نمی‌شود؛ سپس Logicalها را داخل محدوده آن می‌سازید.</p>

<p dir="rtl" align="right">نمونه روند در محیط آزمایشی:</p>

``` text
Command (m for help): n
Partition type
   p   primary
   e   extended
Select (default p): e
Partition number (1-4, default 2): 2
First sector (..., default ...):
Last sector (..., default ...): +5G
```

<p dir="rtl" align="right">سپس دوباره <code>n</code> را وارد کنید و این بار <code>l</code> را برای Logical انتخاب کنید:</p>

``` text
Command (m for help): n
Partition type
   p   primary
   l   logical (numbered from 5)
Select (default p): l
Adding logical partition 5
First sector (..., default ...):
Last sector (..., default ...):
Created a new partition 5 of type 'Linux' and of size ...
```

<p dir="rtl" align="right">در این نمونه، پارتیشن Extended شماره ۲ فقط ظرف است و Logical داخل آن شماره ۵ گرفته است. فضای Logical باید در محدوده Extended باشد و با پارتیشن‌های دیگر تداخل نداشته باشد. برای دیدن نتیجه، <code>p</code> را وارد کنید.</p>

<blockquote dir="rtl" align="right">
<p>مثال‌های عددی بالا شماتیک‌اند؛ محدوده سکتورها به دیسک و فضای آزاد آن بستگی دارد. خروجی واقعی خود <code>fdisk</code> را ملاک قرار دهید.</p>
</blockquote>

<h2 dir="rtl" align="right">۵. ثبت یا لغو تغییرات</h2>

<p dir="rtl" align="right">بعد از ساخت پارتیشن، دوباره جدول را ببینید:</p>

``` text
Command (m for help): p
```

<p dir="rtl" align="right">اگر جدول دقیقاً مطابق انتظار است و نام دیسک را بررسی کرده‌اید، برای ثبت تغییرات:</p>

``` text
Command (m for help): w
```

<p dir="rtl" align="right">برای خروج بدون ثبت تغییراتِ ذخیره‌نشده:</p>

``` text
Command (m for help): q
```

<p dir="rtl" align="right"><code>w</code> تغییرات را روی دیسک می‌نویسد؛ بنابراین پیش از آن بررسی نهایی ضروری است.</p>

<h2 dir="rtl" align="right">۶. ساخت پارتیشن در GPT</h2>

<p dir="rtl" align="right">GPT برخلاف MBR محدودیت چهار ورودی Primary را ندارد و در نسخه‌های جدید <code>fdisk</code> معمولاً می‌تواند تعداد زیادی پارتیشن (در خروجی نمونه تا ۱۲۸ ورودی) داشته باشد. در GPT مفهوم Primary/Extended/Logical به شکل MBR وجود ندارد.</p>

<p dir="rtl" align="right">برای بررسی جدول:</p>

``` bash
sudo fdisk -l /dev/sdb
```

<p dir="rtl" align="right">در خروجی باید این مورد را ببینید:</p>

``` text
Disklabel type: gpt
```

<p dir="rtl" align="right">برای ساخت پارتیشن در دیسک آزمایشی GPT:</p>

``` bash
sudo fdisk /dev/sdb
```

<p dir="rtl" align="right">سپس:</p>

``` text
Command (m for help): p
Command (m for help): n
Partition number (1-128, default 1): 1
First sector (..., default ...):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (..., default ...): +2G
Created a new partition 1 of type 'Linux filesystem' and of size 2 GiB.
```

<p dir="rtl" align="right">در GPT نیز می‌توان برای شروع و پایان، مقادیر پیش‌فرض را پذیرفت یا اندازه‌ای مانند <code>+2G</code> وارد کرد. برای بررسی نتیجه <code>p</code> را بزنید؛ برای ثبت <code>w</code> و برای لغو <code>q</code> را وارد کنید.</p>

<p dir="rtl" align="right">نام پارتیشن به نوع نام‌گذاری دستگاه بستگی دارد؛ برای نمونه ممکن است <code>/dev/sdb1</code> یا در NVMe، <code>/dev/nvme0n1p4</code> باشد.</p>

<h3 dir="rtl" align="right">تبدیل MBR به GPT؛ هشدار</h3>

<p dir="rtl" align="right">برای کار با GPT ابزارهایی مانند <code>gdisk</code> و <code>sgdisk</code> وجود دارند. نصب در توزیع‌های مبتنی بر Fedora/AlmaLinux معمولاً با این دستور انجام می‌شود:</p>

``` bash
sudo dnf install gdisk
```

<p dir="rtl" align="right">در Debian/Ubuntu معمولاً:</p>

``` bash
sudo apt install gdisk
```

<p dir="rtl" align="right">دستور زیر در <code>sgdisk</code> تلاش می‌کند جدول MBR موجود را به GPT تبدیل کند:</p>

``` bash
sudo sgdisk -g /dev/sdb
```

<p dir="rtl" align="right">نمونه پیام ممکن است شبیه این باشد:</p>

``` text
Found invalid GPT and valid MBR; converting MBR to GPT format in memory.
The operation has completed successfully.
```

<p dir="rtl" align="right"><strong>هشدار:</strong> این فرمان را روی دیسک سیستم یا دیسکی که داده مهم دارد بدون برنامه مهاجرت، پشتیبان‌گیری و بررسی سازگاری اجرا نکنید. تبدیل جدول پارتیشن ممکن است روی بوت، پارتیشن‌ها و داده‌ها اثر بگذارد و پیام موفقیت به‌تنهایی تضمین نمی‌کند که کل سیستم پس از تبدیل درست کار می‌کند. برای تمرین از دیسک خالی ماشین مجازی استفاده کنید.</p>

<p dir="rtl" align="right">پس از هر تغییر می‌توانید وضعیت را بررسی کنید:</p>

``` bash
sudo fdisk -l /dev/sdb
sudo lsblk -f
```

<p dir="rtl" align="right">اگر کرنل هنوز جدول قبلی را استفاده می‌کند، ابزار ممکن است نیاز به بازخوانی جدول داشته باشد. این کار را فقط وقتی انجام دهید که دیسک در وضعیت امنی باشد؛ گاهی راه‌اندازی مجدد لازم می‌شود.</p>

<h2 dir="rtl" align="right">۷. ساخت فایل‌سیستم با <code>mkfs</code></h2>

<p dir="rtl" align="right">ساخت پارتیشن و ساخت فایل‌سیستم دو کار متفاوت‌اند. بعد از اینکه پارتیشن جدید را ایجاد کردید، برای استفاده معمول باید روی آن فایل‌سیستم بسازید. مثال زیر فرض می‌کند پارتیشن آزمایشی <code>/dev/sdb1</code> است و داده‌ای روی آن لازم ندارید.</p>

<p dir="rtl" align="right">برای ساخت فایل‌سیستم ext4:</p>

``` bash
sudo mkfs.ext4 /dev/sdb1
```

<p dir="rtl" align="right">نمونه خروجی:</p>

``` text
mke2fs 1.47.2 (...)
Creating filesystem with ... 4k blocks and ... inodes
Filesystem UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Superblock backups stored on blocks:
        ...
Allocating group tables: done
Writing inode tables: done
Creating journal (... blocks): done
Writing superblocks and filesystem accounting information: done
```

<p dir="rtl" align="right"><code>ext4</code> فایل‌سیستم رایجی در لینوکس است؛ اما «بهترین انتخاب برای همه شرایط» نیست و انتخاب فایل‌سیستم به کاربرد بستگی دارد. اجرای <code>mkfs</code> روی پارتیشن دارای اطلاعات می‌تواند دسترسی به داده‌های قبلی را از بین ببرد.</p>

<p dir="rtl" align="right">برای بررسی فایل‌سیستم و UUID:</p>

``` bash
lsblk -f
sudo blkid /dev/sdb1
```

<h2 dir="rtl" align="right">۸. Mount و Unmount کردن پارتیشن</h2>

<p dir="rtl" align="right">برای دسترسی به فایل‌های یک فایل‌سیستم، آن را روی یک مسیر (Mount point) متصل می‌کنیم. ابتدا مسیر را بسازید و سپس پارتیشن را Mount کنید:</p>

``` bash
sudo mkdir -p /mnt/mydata
sudo mount /dev/sdb1 /mnt/mydata
df -hT /mnt/mydata
```

<p dir="rtl" align="right">برای جداکردن فایل‌سیستم:</p>

``` bash
sudo umount /mnt/mydata
```

<p dir="rtl" align="right">قبل از <code>umount</code> از مسیر خارج شوید و مطمئن شوید برنامه‌ای از فایل‌های آن استفاده نمی‌کند. <code>/mnt</code> به‌خودی‌خود موقت‌بودن را تعیین نمی‌کند؛ Mount دستی معمولاً پس از راه‌اندازی مجدد خودکار برقرار نمی‌شود، مگر آنکه تنظیمی برای آن تعریف کرده باشید.</p>

<h2 dir="rtl" align="right">۹. Mount دائمی با <code>/etc/fstab</code></h2>

<h3 dir="rtl" align="right">مرحله ۱: پیدا کردن UUID</h3>

``` bash
sudo blkid /dev/sdb1
```

<p dir="rtl" align="right">نمونه خروجی:</p>

``` text
/dev/sdb1: UUID="35e84912-8acf-4f4b-a1ed-3491ff1cae48" TYPE="ext4"
```

<p dir="rtl" align="right">UUID واقعی دستگاه خود را استفاده کنید؛ UUID نمونه را عیناً کپی نکنید.</p>

<h3 dir="rtl" align="right">مرحله ۲: ساخت Mount point</h3>

``` bash
sudo mkdir -p /mnt/mydata
```

<h3 dir="rtl" align="right">مرحله ۳: پشتیبان‌گیری و ویرایش <code>fstab</code></h3>

``` bash
sudo cp /etc/fstab /etc/fstab.backup
sudo nano /etc/fstab
```

<p dir="rtl" align="right">در انتهای فایل، خطی با UUID واقعی خود اضافه کنید:</p>

``` text
UUID=35e84912-8acf-4f4b-a1ed-3491ff1cae48  /mnt/mydata  ext4  defaults  0  2
```

<p dir="rtl" align="right">فیلدها به‌ترتیب دستگاه، Mount point، نوع فایل‌سیستم، گزینه‌ها، مقدار dump و ترتیب بررسی <code>fsck</code> هستند. برای فایل‌سیستم ext4 غیرریشه‌ای، مقدار آخر معمولاً <code>2</code> است.</p>

<h3 dir="rtl" align="right">مرحله ۴: آزمایش تنظیمات</h3>

``` bash
sudo systemctl daemon-reload
sudo mount -a
findmnt /mnt/mydata
df -hT /mnt/mydata
```

<p dir="rtl" align="right">اگر <code>mount -a</code> خطا داد، آن را نادیده نگیرید؛ خط را بررسی و در صورت نیاز از نسخه پشتیبان برای بازگرداندن <code>fstab</code> استفاده کنید. خطای <code>fstab</code> ممکن است روی فرایند بوت اثر بگذارد.</p>

<h2 dir="rtl" align="right">۱۰. آشنایی کوتاه با فایل‌سیستم‌ها و رمزنگاری</h2>

<p dir="rtl" align="right">• <strong>Btrfs:</strong> فایل‌سیستم لینوکس با قابلیت‌هایی مانند subvolume و snapshot.</p>
<p dir="rtl" align="right">• <strong>Swap:</strong> فضای مورد استفاده برای حافظه مجازی؛ Swap پارتیشن به‌خودی‌خود فایل‌سیستم معمولی برای نگهداری فایل‌ها نیست.</p>
<p dir="rtl" align="right">• <strong>LUKS:</strong> قالب رایج لینوکس برای رمزنگاری بلوکیِ دستگاه یا پارتیشن، معمولاً با ابزار <code>cryptsetup</code>.</p>
<p dir="rtl" align="right">• <strong>eCryptfs:</strong> فایل‌سیستم رمزنگاری‌شده در سطح فایل؛ فایل‌ها و فراداده‌های مرتبط در لایه فایل‌سیستم رمزنگاری می‌شوند. این ابزار در بسیاری از توزیع‌های جدید کمتر از گذشته استفاده می‌شود.</p>

<h2 dir="rtl" align="right">۱۱. رمزنگاری در سطح فایل با eCryptfs</h2>

<p dir="rtl" align="right">این بخش برای آشنایی آموزشی است. پیش از شروع بررسی کنید بسته در توزیع شما در دسترس و با کرنل شما سازگار باشد. در Debian/Ubuntu ممکن است ابزارها با این فرمان نصب شوند:</p>

``` bash
sudo apt install ecryptfs-utils
```

<p dir="rtl" align="right">در این مثال، <code>/data</code> مسیر داده‌های زیرین و <code>/sedata</code> مسیر نمایان‌شدن فایل‌ها از طریق eCryptfs است. نام مسیرها دلخواه‌اند؛ مسیرها را قبل از Mount بررسی کنید و از مسیر دارای اطلاعات مهم استفاده نکنید.</p>

``` bash
sudo mkdir -p /data /sedata
sudo mount -t ecryptfs /data /sedata
```

<p dir="rtl" align="right">در حالت تعاملی ممکن است از شما گزینه‌هایی مانند نوع کلید، عبارت عبور (passphrase)، الگوریتم رمز، اندازه کلید و رمزنگاری نام فایل‌ها پرسیده شود. برای تمرین، گزینه‌های پیشنهادی ابزار را فقط در صورتی انتخاب کنید که معنای آن‌ها را می‌دانید؛ الگوریتم‌های قدیمی یا گزینه‌های ناسازگار را بدون بررسی انتخاب نکنید.</p>

<p dir="rtl" align="right">پس از Mount موفق، فایل‌هایی که از مسیر <code>/sedata</code> می‌نویسید از طریق لایه رمزنگاری‌شده قابل مشاهده‌اند و داده‌های زیرین در <code>/data</code> به شکل رمزنگاری‌شده ذخیره می‌شوند. برای اطمینان از رفتار دقیق و بازیابی داده‌ها، حتماً دستورالعمل همان نسخه eCryptfs را مطالعه کنید و عبارت عبور را امن نگه دارید.</p>

<p dir="rtl" align="right">برای جداکردن:</p>

``` bash
sudo umount /sedata
```

<blockquote dir="rtl" align="right">
<p>توجه: مثال تعاملی eCryptfs در نسخه‌های مختلف ممکن است متفاوت باشد. قبل از استفاده واقعی، سازوکار نگهداری کلید، Mount مجدد و بازیابی را آزمایش کنید. رمزنگاری بدون مدیریت درست کلید، خطر از دست‌رفتن دائمی داده‌ها را دارد.</p>
</blockquote>

<h2 dir="rtl" align="right">۱۲. رمزنگاری دیسک یا پارتیشن با LUKS</h2>

<p dir="rtl" align="right">LUKS معمولاً از طریق <code>cryptsetup</code> مدیریت می‌شود. در مثال زیر <code>/dev/sdb</code> یک <strong>دیسک آزمایشی کاملاً قابل‌پاک‌شدن</strong> است. اگر می‌خواهید فقط یک پارتیشن را رمزنگاری کنید، به‌جای آن نام همان پارتیشن را بدهید؛ این انتخاب را با <code>lsblk</code> و <code>fdisk -l</code> بررسی کنید.</p>

<p dir="rtl" align="right">نصب ابزار در Debian/Ubuntu:</p>

``` bash
sudo apt update
sudo apt install cryptsetup
```

<p dir="rtl" align="right">در AlmaLinux/Fedora:</p>

``` bash
sudo dnf install cryptsetup
```

<h3 dir="rtl" align="right">مرحله ۱: شناسایی دستگاه</h3>

``` bash
lsblk -f
sudo fdisk -l
sudo blkid
```

<p dir="rtl" align="right">دستگاه مورد نظر نباید Mount یا در حال استفاده باشد. دستور بعدی ساختار قبلی دستگاه انتخاب‌شده را بازنویسی می‌کند.</p>

<h3 dir="rtl" align="right">مرحله ۲: ساخت محفظه LUKS</h3>

``` bash
sudo cryptsetup luksFormat /dev/sdb
```

<p dir="rtl" align="right">نمونه پیام:</p>

``` text
WARNING!
This will overwrite data on /dev/sdb irrevocably.

Are you sure? (Type 'YES' in capital letters): YES
Enter passphrase for /dev/sdb:
Verify passphrase:
Command successful.
```

<p dir="rtl" align="right">عبارت عبور قوی انتخاب کنید و آن را در محل امن نگه دارید. اگر عبارت عبور و کلیدهای بازیابی را از دست بدهید، ممکن است داده‌ها قابل بازیابی نباشند.</p>

<h3 dir="rtl" align="right">مرحله ۳: بازکردن محفظه رمزنگاری‌شده</h3>

<p dir="rtl" align="right">با نام دلخواه <code>backup2</code> دستگاه رمزنگاری‌شده را باز می‌کنیم:</p>

``` bash
sudo cryptsetup open /dev/sdb backup2
```

<p dir="rtl" align="right">فرمان از شما عبارت عبور را می‌خواهد. پس از موفقیت، دستگاه نگاشت‌شده معمولاً در مسیر زیر ظاهر می‌شود:</p>

``` bash
ls -l /dev/mapper/backup2
```

<p dir="rtl" align="right">نمونه:</p>

``` text
/dev/mapper/backup2 -> ../dm-0
```

<p dir="rtl" align="right">وضعیت نگاشت را بررسی کنید:</p>

``` bash
sudo cryptsetup status backup2
```

<p dir="rtl" align="right">نمونه خروجی:</p>

``` text
/dev/mapper/backup2 is active.
type:    LUKS2
device:  /dev/sdb
mode:    read/write
```

<p dir="rtl" align="right">جزئیات خروجی مانند نوع LUKS و الگوریتم به نسخه و تنظیمات شما بستگی دارد.</p>

<p dir="rtl" align="right">برای دیدن اطلاعات سربرگ LUKS:</p>

``` bash
sudo cryptsetup luksDump /dev/sdb
```

<h3 dir="rtl" align="right">مرحله ۴: ساخت فایل‌سیستم داخل دستگاه بازشده</h3>

<p dir="rtl" align="right">فایل‌سیستم را روی دستگاه نگاشت‌شده بسازید، نه روی دستگاه خامی که سربرگ LUKS دارد:</p>

``` bash
sudo mkfs.ext4 /dev/mapper/backup2
```

<p dir="rtl" align="right">این کار فایل‌سیستم جدید ایجاد می‌کند؛ هر داده‌ای که قبلاً داخل نگاشت بوده باشد از دست می‌رود. اجرای <code>dd</code> با ورودی صفر در این مرحله لازم نیست.</p>

<h3 dir="rtl" align="right">مرحله ۵: Mount کردن فایل‌سیستم</h3>

``` bash
sudo mkdir -p /backup2
sudo mount /dev/mapper/backup2 /backup2
df -hT /backup2
```

<p dir="rtl" align="right">اکنون می‌توانید فایل‌ها را داخل <code>/backup2</code> ایجاد کنید. داده‌ها در مسیر Mountشده به‌صورت عادی دیده می‌شوند، اما در دستگاه زیرین از طریق لایه LUKS رمزنگاری می‌شوند.</p>

<h3 dir="rtl" align="right">مرحله ۶: Unmount و بستن محفظه</h3>

<p dir="rtl" align="right">ابتدا از فایل‌سیستم خارج شوید و آن را جدا کنید:</p>

``` bash
cd /
sudo umount /backup2
```

<p dir="rtl" align="right">سپس نگاشت رمزنگاری‌شده را ببندید:</p>

``` bash
sudo cryptsetup close backup2
```

<p dir="rtl" align="right">بعد از بستن، مسیر <code>/dev/mapper/backup2</code> دیگر نباید فعال باشد. برای استفاده مجدد، محفظه را با <code>cryptsetup open</code> باز کنید و سپس فایل‌سیستم داخل آن را Mount کنید.</p>

<blockquote dir="rtl" align="right">
<p><strong>نکته امنیتی:</strong> هرگز <code>mkfs.ext4</code> را روی <code>/dev/sdb</code> پس از اجرای <code>luksFormat</code> اجرا نکنید؛ فایل‌سیستم باید روی <code>/dev/mapper/backup2</code> ساخته شود. اجرای <code>mkfs</code> روی دستگاه اشتباه می‌تواند سربرگ رمزنگاری یا ساختار مورد نیاز برای دسترسی به داده‌ها را نابود کند.</p>
</blockquote>

<h2 dir="rtl" align="right">۱۳. مرور سریع فرمان‌ها</h2>

  هدف                          فرمان
<p dir="rtl" align="right">---------------------------- ----------------------------------------- مشاهده دیسک‌ها                <code>lsblk -f</code> نمایش جدول پارتیشن           <code>sudo fdisk -l /dev/sdb</code> ویرایش پارتیشن‌بندی           <code>sudo fdisk /dev/sdb</code> ساخت فایل‌سیستم ext4          <code>sudo mkfs.ext4 /dev/sdb1</code> یافتن UUID                   <code>sudo blkid /dev/sdb1</code> Mount                        <code>sudo mount /dev/sdb1 /mnt/mydata</code> Unmount                      <code>sudo umount /mnt/mydata</code> مشاهده مصرف فضای فایل‌سیستم   <code>df -hT</code> ساخت محفظه LUKS              <code>sudo cryptsetup luksFormat /dev/sdb</code> بازکردن LUKS                 <code>sudo cryptsetup open /dev/sdb backup2</code> بستن LUKS                    <code>sudo cryptsetup close backup2</code></p>

<h2 dir="rtl" align="right">نکته پایانی</h2>

<p dir="rtl" align="right">همیشه سه مورد را جداگانه بررسی کنید: <strong>دیسک یا پارتیشن هدف، جدول پارتیشن (MBR/GPT)، و فایل‌سیستم</strong>. ساخت پارتیشن به‌تنهایی فایل‌سیستم نمی‌سازد؛ ساخت فایل‌سیستم به‌تنهایی Mount انجام نمی‌دهد؛ و در LUKS، ابتدا باید محفظه را باز کنید تا دستگاه نگاشت‌شده برای ساخت فایل‌سیستم و Mount در دسترس باشد.</p>
