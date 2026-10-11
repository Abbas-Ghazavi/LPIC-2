# راهنمای کامل LVM (Logical Volume Management)

<div dir="rtl" align="right">

## مقدمه

LVM یک لایه انتزاعی بین دیسک‌ها و فایل‌سیستم‌ها ایجاد می‌کند و امکان مدیریت پویا، افزایش و کاهش فضای ذخیره‌سازی را فراهم می‌سازد؛ بدون اینکه در بسیاری از موارد نیاز به پارتیشن‌بندی مجدد داشته باشیم.

### مفاهیم اصلی LVM

* **PV (Physical Volume):** دیسک یا پارتیشنی که برای استفاده در LVM آماده شده است.
* **VG (Volume Group):** مجموعه‌ای از PVها که مانند یک استخر ذخیره‌سازی مشترک عمل می‌کند.
* **LV (Logical Volume):** فضای منطقی ساخته‌شده از VG که می‌توان روی آن فایل‌سیستم ایجاد کرد و آن را Mount کرد.

ساختار کلی LVM به این شکل است:

</div>

```text
Physical Volumes (PV)
       |
       v
 Volume Group (VG)
       |
       v
Logical Volume (LV)
       |
       v
  Filesystem (ext4)
       |
       v
 Mount Point (/mnt)
```

<div dir="rtl" align="right">

---

## ۱. تبدیل دیسک‌ها به Physical Volume

ابتدا دیسک‌های خام را به Physical Volume تبدیل می‌کنیم.

**دستور:**

</div>

```bash
pvcreate /dev/sdb /dev/sdc
```

<div dir="rtl" align="right">

**نمونه خروجی:**

</div>

```text
Physical volume "/dev/sdb" successfully created.
Physical volume "/dev/sdc" successfully created.
```

<div dir="rtl" align="right">

برای مشاهده فهرست PVها:

</div>

```bash
pvs
```

<div dir="rtl" align="right">

برای مشاهده جزئیات بیشتر:

</div>

```bash
pvdisplay
```

<div dir="rtl" align="right">

## ۲. ساخت Volume Group

حالا یک Volume Group با نام `mabedini` از دو دیسک ایجاد می‌کنیم.

</div>

```bash
vgcreate mabedini /dev/sdb /dev/sdc
```

<div dir="rtl" align="right">

**نمونه خروجی:**

</div>

```text
Volume group "mabedini" successfully created
```

<div dir="rtl" align="right">

برای مشاهده فهرست VGها:

</div>

```bash
vgs
```

<div dir="rtl" align="right">

برای مشاهده جزئیات:

</div>

```bash
vgdisplay mabedini
```

<div dir="rtl" align="right">

## ۳. ساخت Logical Volume

از فضای آزاد VG، یک Logical Volume با نام `data` و اندازه ۸ گیگابایت ایجاد می‌کنیم.

</div>

```bash
lvcreate -n data -L 8G mabedini
```

<div dir="rtl" align="right">

**نمونه خروجی:**

</div>

```text
Logical volume "data" created.
```

<div dir="rtl" align="right">

برای مشاهده فهرست LVها:

</div>

```bash
lvs
```

<div dir="rtl" align="right">

مسیر این Logical Volume به شکل زیر خواهد بود:

</div>

```text
/dev/mabedini/data
```

<div dir="rtl" align="right">

## ۴. فرمت کردن Logical Volume

در این مرحله، فایل‌سیستم `ext4` را روی Logical Volume ایجاد می‌کنیم.

**هشدار:** دستور `mkfs.ext4` اطلاعات موجود روی فایل‌سیستم قبلی را از بین می‌برد. قبل از اجرا، مسیر دستگاه را بررسی کنید.

</div>

```bash
mkfs.ext4 /dev/mabedini/data
```

<div dir="rtl" align="right">

**نمونه خروجی:**

</div>

```text
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2097152 4k blocks and 524288 inodes
Filesystem UUID: e071c494-b045-490a-8251-10a6de6e2739
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

<div dir="rtl" align="right">

**نکته:** مسیر `/dev/mabedini/data` معمولاً یک مسیر نمادین (Symlink) به `/dev/mapper/mabedini-data` است. هر دو مسیر به همان Logical Volume اشاره می‌کنند.

---

## ۵. Mount کردن Logical Volume

ابتدا مسیر Mount Point را ایجاد می‌کنیم.

</div>

```bash
mkdir -p /mnt
```

<div dir="rtl" align="right">

سپس Logical Volume را روی آن Mount می‌کنیم.

</div>

```bash
mount /dev/mabedini/data /mnt
```

<div dir="rtl" align="right">

برای بررسی فضای ذخیره‌سازی:

</div>

```bash
df -h /mnt
```

<div dir="rtl" align="right">

### دائمی کردن Mount

برای Mount شدن خودکار پس از راه‌اندازی مجدد سیستم، باید ورودی مناسبی به `/etc/fstab` اضافه کنیم.

ابتدا UUID فایل‌سیستم را پیدا کنید:

</div>

```bash
blkid /dev/mabedini/data
```

<div dir="rtl" align="right">

خروجی شامل UUID فایل‌سیستم خواهد بود. مقدار واقعی UUID را در دستور زیر جایگزین کنید:

</div>

```fstab
UUID=<filesystem-uuid> /mnt ext4 defaults 0 2
```

<div dir="rtl" align="right">

این خط را به انتهای فایل `/etc/fstab` اضافه کنید. استفاده از UUID معمولاً از ارجاع مستقیم به نام دستگاه مطمئن‌تر است.

پس از ذخیره، تنظیمات را بررسی کنید:

</div>

```bash
mount -a
```

<div dir="rtl" align="right">

**نکته:** قبل از اجرای `mount -a` مطمئن شوید UUID و مسیر Mount Point صحیح هستند؛ پیکربندی اشتباه ممکن است باعث بروز مشکل هنگام راه‌اندازی سیستم شود.

---

## ۶. اضافه کردن هارد جدید به Volume Group

فرض کنید یک دیسک جدید با نام `/dev/sdd` به سیستم اضافه شده است.

ابتدا آن را به PV تبدیل می‌کنیم:

</div>

```bash
pvcreate /dev/sdd
```

<div dir="rtl" align="right">

سپس دیسک را به VG موجود اضافه می‌کنیم:

</div>

```bash
vgextend mabedini /dev/sdd
```

<div dir="rtl" align="right">

برای بررسی فضای جدید:

</div>

```bash
pvs
vgs
```

<div dir="rtl" align="right">

**نکته:** قبل از اجرای `pvcreate` مطمئن شوید دیسک جدید است و اطلاعات موردنیازی روی آن وجود ندارد.

---

## ۷. افزایش حجم Logical Volume

فرض کنید می‌خواهیم ۲ گیگابایت به Logical Volume با نام `data` اضافه کنیم.

</div>

```bash
lvextend -L +2G /dev/mabedini/data
```

<div dir="rtl" align="right">

با اجرای این دستور، حجم LV از ۸ به ۱۰ گیگابایت افزایش پیدا می‌کند؛ البته به شرط اینکه فضای آزاد کافی در VG وجود داشته باشد.

**نمونه خروجی:**

</div>

```text
Size of logical volume mabedini/data changed from 8.00 GiB (2048 extents) to 10.00 GiB (2560 extents).
Logical volume mabedini/data successfully resized.
```

<div dir="rtl" align="right">

برای بررسی اندازه جدید:

</div>

```bash
lvs
```

<div dir="rtl" align="right">

## ۸. بزرگ کردن فایل‌سیستم با resize2fs

افزایش حجم LV به‌تنهایی کافی نیست؛ فایل‌سیستم داخل آن نیز باید بتواند از فضای جدید استفاده کند.

برای فایل‌سیستم `ext4` دستور زیر را اجرا می‌کنیم:

</div>

```bash
resize2fs /dev/mabedini/data
```

<div dir="rtl" align="right">

**نمونه خروجی:**

</div>

```text
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mabedini/data is mounted on /mnt; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 2
The filesystem on /dev/mabedini/data is now 2621440 (4k) blocks long.
```

<div dir="rtl" align="right">

برای بررسی فضای نهایی:

</div>

```bash
df -h /mnt
```

### روش ساده‌تر

می‌توان افزایش حجم LV و فایل‌سیستم را با یک دستور انجام داد:

</div>

```bash
lvextend -r -L +2G /dev/mabedini/data
```

<div dir="rtl" align="right">

گزینه `-r` تلاش می‌کند فایل‌سیستم را نیز متناسب با افزایش حجم LV تغییر اندازه دهد.

**تفاوت فایل‌سیستم‌ها:**

* برای `ext2`، `ext3` و `ext4` از `resize2fs` استفاده می‌شود.
* برای `XFS` از `xfs_growfs` استفاده می‌شود؛ برای مثال:

</div>

```bash
xfs_growfs /mnt
```

<div dir="rtl" align="right">

برای XFS، فایل‌سیستم باید Mount شده باشد. همچنین کاهش حجم فایل‌سیستم به روش افزایش حجم آن نیست؛ به‌خصوص `XFS` از کوچک‌سازی پشتیبانی نمی‌کند.

---

## ۹. مراحل معکوس؛ حذف LVM

**هشدار مهم:** مراحل زیر ممکن است باعث از دست رفتن دائمی اطلاعات شوند. فقط روی دیسک‌ها و Volumeهایی اجرا کنید که مطمئن هستید دیگر به آن‌ها نیاز ندارید.

### ۹.۱. Unmount کردن

</div>

```bash
umount /mnt
```

<div dir="rtl" align="right">

اگر دستگاه در حال استفاده باشد، ممکن است عملیات Unmount با خطا مواجه شود.

اگر در فایل `/etc/fstab` ورودی مربوط به این Volume را اضافه کرده‌اید، پس از اطمینان از عدم نیاز به آن، ورودی را نیز حذف کنید.

### ۹.۲. حذف Logical Volume

</div>

```bash
lvremove /dev/mabedini/data
```

<div dir="rtl" align="right">

در صورت نمایش درخواست تأیید، فقط زمانی ادامه دهید که از حذف Volume و اطلاعات آن مطمئن هستید.

### ۹.۳. حذف Volume Group

</div>

```bash
vgremove mabedini
```

<div dir="rtl" align="right">

این دستور Volume Group را حذف می‌کند. قبل از اجرا مطمئن شوید Logical Volumeهای موردنیاز دیگری داخل آن باقی نمانده‌اند.

### ۹.۴. حذف مشخصات Physical Volume

</div>

```bash
pvremove /dev/sdb
pvremove /dev/sdc
pvremove /dev/sdd
```

<div dir="rtl" align="right">

این دستورات باید فقط روی PVهایی اجرا شوند که دیگر عضو ساختار LVM موردنیاز شما نیستند.

### ۹.۵. پاک کردن امضاهای باقی‌مانده با wipefs

</div>

```bash
wipefs /dev/sdb
wipefs /dev/sdc
wipefs /dev/sdd
```

<div dir="rtl" align="right">

دستورات بالا بدون گزینه پاک‌کننده، امضاهای شناسایی‌شده را نمایش می‌دهند.

**هشدار:** برای پاک کردن واقعی امضاها، باید گزینه مناسب را آگاهانه انتخاب کنید. برای مثال، گزینه `-a` امضاهای شناسایی‌شده را پاک می‌کند:

</div>

```bash
wipefs -a /dev/sdb
```

<div dir="rtl" align="right">

این عملیات مخرب است و ممکن است شناسایی فایل‌سیستم یا متادیتای ذخیره‌سازی را مختل کند. گزینه `-f` نیز برای اجبار به انجام عملیات در شرایط خاص است و نباید بدون بررسی استفاده شود.

همچنین `wipefs` به‌تنهایی تضمین نمی‌کند که تمام داده‌های دیسک به‌صورت امن و غیرقابل‌بازیابی پاک شده باشند.

---

## ۱۰. خلاصه دستورات

### ایجاد ساختار LVM

</div>

```bash
# Create Physical Volumes
pvcreate /dev/sdX

# Create Volume Group
vgcreate <vg_name> /dev/sdX

# Create Logical Volume
lvcreate -n <lv_name> -L <size> <vg_name>

# Create ext4 filesystem
mkfs.ext4 /dev/<vg_name>/<lv_name>
```

<div dir="rtl" align="right">

### مدیریت و افزایش حجم

</div>

```bash
# Show Physical Volumes
pvs

# Show Volume Groups
vgs

# Show Logical Volumes
lvs

# Mount Logical Volume
mount /dev/<vg_name>/<lv_name> /mnt

# Add a new Physical Volume to a Volume Group
vgextend <vg_name> /dev/sdX

# Extend Logical Volume
lvextend -L +<size> /dev/<vg_name>/<lv_name>

# Extend LV and resize filesystem
lvextend -r -L +<size> /dev/<vg_name>/<lv_name>

# Resize ext2/ext3/ext4 filesystem manually
resize2fs /dev/<vg_name>/<lv_name>
```

<div dir="rtl" align="right">

### حذف ساختار LVM

</div>

```bash
# Unmount filesystem
umount /mnt

# Remove Logical Volume
lvremove /dev/<vg_name>/<lv_name>

# Remove Volume Group
vgremove <vg_name>

# Remove Physical Volume metadata
pvremove /dev/sdX

# List filesystem signatures
wipefs /dev/sdX
```

<div dir="rtl" align="right">

---

## جمع‌بندی

LVM مدیریت فضای ذخیره‌سازی لینوکس را انعطاف‌پذیرتر می‌کند. با استفاده از سه مفهوم اصلی PV، VG و LV می‌توان دیسک‌های مختلف را در یک ساختار ذخیره‌سازی ترکیب کرد و در صورت نیاز حجم Logical Volume را افزایش داد.

مهم‌ترین نکته این است که افزایش حجم LV و افزایش حجم فایل‌سیستم دو مرحله متفاوت هستند؛ مگر اینکه از گزینه‌ای مانند `-r` استفاده شود که تلاش می‌کند هر دو مرحله را انجام دهد.

</div>

---
