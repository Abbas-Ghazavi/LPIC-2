# Linux Disk Management: `fdisk`, Filesystems, Mounting, and Encryption

This guide is a practical reference for Linux disk management and
related LPIC topics. Commands and sample outputs may vary depending on
your distribution, tool versions, and disk layout.

> ⚠️ **Warning:** Partitioning, formatting, and encryption can destroy
> data. Practice in a virtual machine using a disposable test disk.
> Always verify the target device and back up important data before
> making changes. In this guide, `/dev/sdb` represents a test disk. Do
> not blindly copy commands into a production system.

## 1. List Disks and Partitions with `lsblk`

Start by listing block devices, partitions, device types, and mount
points:

``` bash
lsblk
```

Example output:

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

The `TYPE` column identifies devices such as disks (`disk`), partitions
(`part`), and LVM devices. `MOUNTPOINTS` shows where filesystems are
mounted.

To inspect a disk and its partition table:

``` bash
sudo fdisk -l /dev/sdb
```

**Important:** In the example above, `nvme0n1` is the whole disk and
`nvme0n1p3` is one of its partitions. For partitioning practice, use a
disposable test disk such as `/dev/sdb`. Do not modify your root or
other in-use system partitions.

## 2. Open `fdisk`

Open the test disk:

``` bash
sudo fdisk /dev/sdb
```

Example startup message:

``` text
Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help):
```

Changes are not written to disk until you use `w`. Still, review the
partition table carefully before committing changes.

If `fdisk` warns that the disk is in use, stop unless you understand the
situation. Filesystems may need to be unmounted and related swap
disabled. Do not try to repartition the active system disk from the
running system.

## 3. Print the Partition Table with `p` and View Help with `m`

Inside `fdisk`, use `p` to print the partition table:

``` text
Command (m for help): p

Disk /dev/sdb: 10 GiB, ...
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x12345678
```

`Disklabel type: dos` indicates an MBR/DOS partition table. If it says
`gpt`, the disk uses GPT.

To display the available commands:

``` text
Command (m for help): m
```

Common commands include:

  Command   Purpose
  --------- --------------------------------------
  `p`       Print the partition table
  `n`       Create a new partition
  `d`       Delete a partition
  `t`       Change a partition type
  `l`       List known partition types
  `F`       List unpartitioned free space
  `v`       Verify the partition table
  `w`       Write changes and exit
  `q`       Exit without writing unsaved changes

Available commands may vary depending on the partition-table type.

## 4. Create a Partition on an MBR/DOS Disk

MBR normally provides four partition-table entries. These entries can be
used for primary partitions. If one entry is defined as an **extended
partition**, it acts as a container for logical partitions. Logical
partitions are usually numbered starting at 5. The number and size of
logical partitions depend on available disk space and tool limitations.

Open the test disk and check the table type with `p`. If it is `dos`,
create a partition as follows:

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

Steps:

1.  Enter `n` to create a partition.
2.  Select `p` for a primary partition.
3.  Choose a partition number. In a simple lab, you can usually accept
    the default.
4.  At `First sector`, pressing Enter usually accepts the default
    aligned sector.
5.  At `Last sector`, entering `+2G` requests a partition approximately
    2 GiB in size.

You may see a message such as:

``` text
Partition #1 contains a swap signature.
Do you want to remove the signature? [Y]es/[N]o:
```

This means `fdisk` found a previous swap signature in the partition
area. It may be left over from an earlier use of the disk. Remove it
only if you are certain the previous data or structure is no longer
needed. Otherwise, stop and investigate. This message alone does not
mean swap is currently active.

### Create an Extended and a Logical Partition

If you want multiple logical partitions on an MBR disk, first create an
extended partition. The extended partition is a container; you normally
do not create a regular filesystem directly on it. Then create logical
partitions inside its range.

Example workflow on a test disk:

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

Then enter `n` again and select `l` for a logical partition:

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

In this example, partition 2 is the extended container, while partition
5 is a logical partition inside it. Logical partitions must fit within
the extended partition's range and must not overlap other partitions.
Enter `p` to review the result.

> The sector ranges above are illustrative. Actual values depend on the
> disk and its free space. Follow the values displayed by your own
> `fdisk` session.

## 5. Write or Discard Changes

After creating a partition, print the table again:

``` text
Command (m for help): p
```

If the table is correct and you have verified the target disk, write the
changes:

``` text
Command (m for help): w
```

To exit without writing unsaved changes:

``` text
Command (m for help): q
```

`w` writes changes to disk, so a final review is essential.

## 6. Create Partitions on a GPT Disk

Unlike MBR, GPT does not have the four-primary-partition limit and
modern `fdisk` versions commonly support many partitions (the example
below allows up to 128 entries). GPT does not use the MBR
primary/extended/logical partition scheme.

Check the partition-table type:

``` bash
sudo fdisk -l /dev/sdb
```

Look for:

``` text
Disklabel type: gpt
```

To create a partition on a GPT test disk:

``` bash
sudo fdisk /dev/sdb
```

Then use:

``` text
Command (m for help): p
Command (m for help): n
Partition number (1-128, default 1): 1
First sector (..., default ...):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (..., default ...): +2G
Created a new partition 1 of type 'Linux filesystem' and of size 2 GiB.
```

As with MBR, you can accept the default sector values or enter a size
such as `+2G`. Use `p` to review the table, `w` to write the changes, or
`q` to quit without saving pending changes.

Device names depend on the disk type. For example, a partition may be
named `/dev/sdb1` or `/dev/nvme0n1p4`.

### Converting MBR to GPT: Warning

Tools such as `gdisk` and `sgdisk` can work with GPT. On
Fedora/AlmaLinux-based distributions, you can usually install them with:

``` bash
sudo dnf install gdisk
```

On Debian/Ubuntu-based distributions:

``` bash
sudo apt install gdisk
```

The following `sgdisk` command attempts to convert an existing MBR
partition table to GPT:

``` bash
sudo sgdisk -g /dev/sdb
```

A message may look like:

``` text
Found invalid GPT and valid MBR; converting MBR to GPT format in memory.
The operation has completed successfully.
```

**Warning:** Do not run this on a system disk or a disk containing
important data without a migration plan, backups, and compatibility
checks. Changing the partition table can affect booting, partitions, and
data. A success message alone does not guarantee that the system will
work correctly afterward. Practice on a disposable virtual disk.

After changes, check the current state:

``` bash
sudo fdisk -l /dev/sdb
sudo lsblk -f
```

The kernel may still be using the previous partition table and may need
to reread it. Only do this when the disk is in a safe state; a reboot
may sometimes be required.

## 7. Create a Filesystem with `mkfs`

Creating a partition and creating a filesystem are different tasks.
After creating a new partition, you generally need to create a
filesystem before using it. The following example assumes `/dev/sdb1` is
a disposable test partition with no data you need to keep.

Create an ext4 filesystem:

``` bash
sudo mkfs.ext4 /dev/sdb1
```

Example output:

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

`ext4` is a common Linux filesystem, but it is not automatically the
best choice for every situation. The right filesystem depends on the
intended use. Running `mkfs` on a partition containing data can make
previous data inaccessible.

Check the filesystem and UUID:

``` bash
lsblk -f
sudo blkid /dev/sdb1
```

## 8. Mount and Unmount a Partition

To access files on a filesystem, mount it at a directory. Create a mount
point and mount the test partition:

``` bash
sudo mkdir -p /mnt/mydata
sudo mount /dev/sdb1 /mnt/mydata
df -hT /mnt/mydata
```

To unmount it:

``` bash
sudo umount /mnt/mydata
```

Before unmounting, leave the directory and ensure no process is using
files on that filesystem. `/mnt` is not inherently temporary; a manually
mounted filesystem usually will not be mounted automatically after
reboot unless you configure it to do so.

## 9. Configure a Persistent Mount with `/etc/fstab`

### Step 1: Find the UUID

``` bash
sudo blkid /dev/sdb1
```

Example output:

``` text
/dev/sdb1: UUID="35e84912-8acf-4f4b-a1ed-3491ff1cae48" TYPE="ext4"
```

Use the actual UUID from your own system, not the example UUID.

### Step 2: Create the Mount Point

``` bash
sudo mkdir -p /mnt/mydata
```

### Step 3: Back Up and Edit `fstab`

``` bash
sudo cp /etc/fstab /etc/fstab.backup
sudo nano /etc/fstab
```

Add a line at the end of the file, replacing the example UUID with the
real one:

``` text
UUID=35e84912-8acf-4f4b-a1ed-3491ff1cae48  /mnt/mydata  ext4  defaults  0  2
```

The fields represent the device, mount point, filesystem type, mount
options, dump setting, and `fsck` check order. For a non-root ext4
filesystem, the final field is commonly `2`.

### Step 4: Test the Configuration

``` bash
sudo systemctl daemon-reload
sudo mount -a
findmnt /mnt/mydata
df -hT /mnt/mydata
```

If `mount -a` reports an error, do not ignore it. Check the line and
restore the backup if necessary. An incorrect `fstab` entry can affect
the boot process.

## 10. Quick Overview of Filesystems and Encryption

-   **Btrfs:** A Linux filesystem with features such as subvolumes and
    snapshots.
-   **Swap:** Space used for virtual memory. A swap partition is not a
    regular filesystem for storing ordinary files.
-   **LUKS:** A common Linux disk-encryption format for block devices or
    partitions, usually managed with `cryptsetup`.
-   **eCryptfs:** A file-level encrypted filesystem. Files and related
    metadata are encrypted at the filesystem layer. It is less commonly
    used than it once was on many modern distributions.

## 11. File-Level Encryption with eCryptfs

This section is for learning. Check that the package is available and
compatible with your distribution and kernel. On Debian/Ubuntu, the
tools may be installed with:

``` bash
sudo apt install ecryptfs-utils
```

In this example, `/data` is the underlying data directory and `/sedata`
is the directory where the files are presented through eCryptfs. The
paths are arbitrary. Verify them before mounting, and do not use a
directory containing important data for practice.

``` bash
sudo mkdir -p /data /sedata
sudo mount -t ecryptfs /data /sedata
```

The interactive prompts may ask you to choose a key type, passphrase,
cipher, key size, and whether to encrypt filenames. Select options only
when you understand them. Do not choose old or incompatible algorithms
without checking the documentation for your version.

After a successful mount, files written through `/sedata` are accessed
through the encryption layer, while the underlying data in `/data` is
stored in encrypted form. Test remounting and recovery before relying on
this setup, and keep the passphrase safe.

To unmount:

``` bash
sudo umount /sedata
```

> The interactive eCryptfs prompts can differ between versions. Before
> real use, test key management, remounting, and recovery. Encryption
> without proper key management can lead to permanent data loss.

## 12. Encrypt a Disk or Partition with LUKS

LUKS is commonly managed with `cryptsetup`. In the following example,
`/dev/sdb` is a **disposable test disk that can be completely erased**.
If you intend to encrypt only one partition, use that partition's device
name instead, after verifying it with `lsblk` and `fdisk -l`.

Install the tools on Debian/Ubuntu:

``` bash
sudo apt update
sudo apt install cryptsetup
```

On AlmaLinux/Fedora:

``` bash
sudo dnf install cryptsetup
```

### Step 1: Identify the Device

``` bash
lsblk -f
sudo fdisk -l
sudo blkid
```

The target device must not be mounted or in use. The next command
overwrites the existing structure on the selected device.

### Step 2: Create the LUKS Container

``` bash
sudo cryptsetup luksFormat /dev/sdb
```

Example prompts:

``` text
WARNING!
This will overwrite data on /dev/sdb irrevocably.

Are you sure? (Type 'YES' in capital letters): YES
Enter passphrase for /dev/sdb:
Verify passphrase:
Command successful.
```

Choose a strong passphrase and store it securely. If you lose the
passphrase and any recovery keys, the data may be unrecoverable.

### Step 3: Open the Encrypted Container

Open the encrypted device using the mapping name `backup2`:

``` bash
sudo cryptsetup open /dev/sdb backup2
```

Enter the passphrase when prompted. After a successful command, the
mapped device usually appears at:

``` bash
ls -l /dev/mapper/backup2
```

Example:

``` text
/dev/mapper/backup2 -> ../dm-0
```

Check the mapping status:

``` bash
sudo cryptsetup status backup2
```

Example output:

``` text
/dev/mapper/backup2 is active.
type:    LUKS2
device:  /dev/sdb
mode:    read/write
```

Details such as the LUKS version and algorithm depend on your version
and configuration.

To inspect the LUKS header:

``` bash
sudo cryptsetup luksDump /dev/sdb
```

### Step 4: Create a Filesystem Inside the Opened Mapping

Create the filesystem on the mapped device, not on the raw device that
contains the LUKS header:

``` bash
sudo mkfs.ext4 /dev/mapper/backup2
```

This creates a new filesystem; any existing data inside the mapping will
be lost. There is no need to run `dd` with zero input at this stage.

### Step 5: Mount the Filesystem

``` bash
sudo mkdir -p /backup2
sudo mount /dev/mapper/backup2 /backup2
df -hT /backup2
```

You can now create files under `/backup2`. The files appear normally
through the mounted filesystem, while the underlying block device is
encrypted through LUKS.

### Step 6: Unmount and Close the Container

Leave the filesystem and unmount it:

``` bash
cd /
sudo umount /backup2
```

Then close the encrypted mapping:

``` bash
sudo cryptsetup close backup2
```

After closing it, `/dev/mapper/backup2` should no longer be active. To
use the data again, open the container with `cryptsetup open` and mount
the filesystem inside it.

> **Security note:** Do not run `mkfs.ext4` on `/dev/sdb` after
> `luksFormat`; the filesystem belongs on `/dev/mapper/backup2`. Running
> `mkfs` on the wrong device can destroy the encryption header or other
> structures needed to access your data.

## 13. Command Quick Reference

  Task                         Command
  ---------------------------- -----------------------------------------
  List disks and filesystems   `lsblk -f`
  Display a partition table    `sudo fdisk -l /dev/sdb`
  Edit a partition table       `sudo fdisk /dev/sdb`
  Create an ext4 filesystem    `sudo mkfs.ext4 /dev/sdb1`
  Find a UUID                  `sudo blkid /dev/sdb1`
  Mount a partition            `sudo mount /dev/sdb1 /mnt/mydata`
  Unmount a filesystem         `sudo umount /mnt/mydata`
  Check filesystem space       `df -hT`
  Create a LUKS container      `sudo cryptsetup luksFormat /dev/sdb`
  Open LUKS                    `sudo cryptsetup open /dev/sdb backup2`
  Close LUKS                   `sudo cryptsetup close backup2`

## Final Notes

Always check three things separately: **the target disk or partition,
the partition-table type (MBR/GPT), and the filesystem**. Creating a
partition does not create a filesystem; creating a filesystem does not
mount it; and with LUKS, you must open the container before the mapped
device can be used to create a filesystem and mount it.
