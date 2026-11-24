---
title: "Storage on Linux: Block Devices, Partitions, File Systems and LVM"
date: 2026-11-22
draft: true
description: "How Linux turns a disk into the directory tree you use every day: block devices and device nodes, major and minor numbers, partitions and partition tables, what a file system actually stores, the VFS and mounting, and logical volume management with LVM. With a hands-on lab you can run safely on loop devices."
summary: "A disk is just a long row of numbered blocks. Everything between that and /home/you/notes.txt is a stack of layers: block device, partition, maybe RAID or encryption, maybe LVM, a file system, and a mount. Here is each layer, what it does and the commands that show it."
tags: ["linux", "storage", "block devices", "file systems", "lvm", "lfcs"]
categories: ["Programming"]
---

When you save a file, you think in names and folders: `/home/you/notes.txt`. The disk thinks in nothing of the sort. To the hardware, an SSD is a long row of numbered blocks, and all it knows how to do is "read block 1,048,576" or "write these 4 KiB to block 22". Everything in between is software, and on Linux that software is a stack of small, separate layers you can inspect and rearrange.

This post walks up that stack from the bottom:

1. **Block devices**: how the kernel represents a disk
2. **Partitions**: cutting one disk into several
3. **File systems**: turning blocks into files and directories
4. **Mounting and the VFS**: joining file systems into one tree
5. **LVM**: a flexible layer between disks and file systems
6. A **hands-on lab** that builds the whole stack on fake disks, so you can experiment without touching real hardware

## The big picture

Here's a typical server, from the hardware at the bottom to the path a program opens at the top:

```text
  open("/srv/data/report.csv")             ← a program
            │
  ┌─────────▼──────────────────────────┐
  │ VFS (virtual file system switch)   │   one directory tree, many file systems
  └─────────┬──────────────────────────┘
  ┌─────────▼──────────┐
  │ ext4 / XFS / Btrfs │   file system: names, inodes, free space, journal
  └─────────┬──────────┘
  ┌─────────▼──────────┐
  │ /dev/vg0/data      │   LVM logical volume (device-mapper)       ┐
  └─────────┬──────────┘                                            │
  ┌─────────▼──────────┐                                            │ all of these
  │ /dev/mapper/crypt  │   optional: LUKS encryption (dm-crypt)     │ are block
  └─────────┬──────────┘                                            │ devices
  ┌─────────▼──────────┐                                            │
  │ /dev/md0           │   optional: software RAID (mdadm)          │
  └─────────┬──────────┘                                            │
  ┌─────────▼──────────┐                                            │
  │ /dev/sda2          │   partition                                │
  └─────────┬──────────┘                                            │
  ┌─────────▼──────────┐                                            │
  │ /dev/sda           │   whole disk                               ┘
  └─────────┬──────────┘
      block layer, I/O scheduler, driver (SCSI/SATA, NVMe, virtio)
            │
        physical disk
```

The key idea, and the reason Linux storage is so flexible: **every layer from the disk up to LVM takes a block device and presents a new block device.** A partition is a block device. A RAID array is a block device. An encrypted volume and an LVM logical volume are block devices. So you can stack them in almost any order, and the file system on top neither knows nor cares what's underneath.

Not every system uses every layer. A laptop might be just `disk → partition → ext4`. A cloud VM is often `virtual disk → partition → XFS`, or `disk → LVM → XFS` on Red Hat-family systems, whose installer sets up LVM by default.[^1]

You can see your own stack with one command:

```bash
$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1       259:0    0 476.9G  0 disk
├─nvme0n1p1   259:1    0     1G  0 part /boot/efi
├─nvme0n1p2   259:2    0     2G  0 part /boot
└─nvme0n1p3   259:3    0 473.9G  0 part
  └─luks-3f1c 253:0    0 473.9G  0 crypt
    ├─vg0-root 253:1   0    80G  0 lvm  /
    ├─vg0-swap 253:2   0     8G  0 lvm  [SWAP]
    └─vg0-home 253:3   0   385G  0 lvm  /home
```

Read it as a tree: one NVMe disk, three partitions, the third encrypted, and inside the encryption an LVM volume group with three logical volumes. By the end of this post every column here should make sense.

## Block devices

### Everything is a file, including the disk

Unix exposes hardware as special files under `/dev`. There are two kinds of device file, and `ls -l` shows which is which in the first character:

```bash
$ ls -l /dev/sda /dev/sda1 /dev/tty0 /dev/null
brw-rw---- 1 root disk 8, 0 Nov 22 09:12 /dev/sda
brw-rw---- 1 root disk 8, 1 Nov 22 09:12 /dev/sda1
crw--w---- 1 root tty  4, 0 Nov 22 09:12 /dev/tty0
crw-rw-rw- 1 root root 1, 3 Nov 22 09:12 /dev/null
```

- **`b`: block device.** Storage that's addressed in fixed-size blocks, that you can read or write at any position (random access), and whose I/O goes through the kernel's block layer and page cache. Disks, partitions, USB sticks, RAID arrays, LVM volumes.
- **`c`: character device.** A stream of bytes with no fixed blocks and usually no seeking: terminals, serial ports, `/dev/null`, `/dev/random`.

So a **block device** is the kernel's abstraction for "a thing that stores a fixed number of equally sized blocks, any of which you can read or write". It says nothing about files. It's just an array:

```text
 /dev/sda:   [ block 0 ][ block 1 ][ block 2 ] ... [ block N-1 ]
             each block 512 bytes (logical sector) or 4096 bytes
```

You can treat it exactly like that. This reads the first 512 bytes of a disk, which on an MBR-partitioned disk is the boot sector and partition table:

```bash
sudo dd if=/dev/sda bs=512 count=1 | xxd | tail -n 4
```

And this is why `dd if=image.iso of=/dev/sdb` writes a bootable USB stick: it copies bytes straight onto the block array, ignoring whatever file system was there. (It's also why getting `of=` wrong is so destructive.)

### Major and minor numbers

The `8, 0` and `8, 1` in the `ls -l` output, where a normal file shows its size, are the device's **major** and **minor** numbers. The name `/dev/sda` is only a label; the numbers are what the kernel actually uses. The major number picks the driver, the minor number picks which device that driver handles.

The official registry lists the static allocations. Major 8 is the SCSI disk driver (which also handles SATA and USB disks), and each disk gets 16 minors: the whole disk plus up to 15 partitions, so `sda` is 8:0, `sda1` is 8:1, `sdb` is 8:16.[^2] Major 7 is loop devices. Newer drivers like NVMe and device-mapper get their numbers dynamically, which is why they show up as 259 and 253 in the `lsblk` output above. The registry itself notes that Linux now has extensive support for dynamic device numbering, with sysfs and udev handling the naming.[^2]

### Device names

The name tells you roughly what kind of device it is:

| Name | What it is |
|---|---|
| `sda`, `sdb`, … | SCSI/SATA/SAS/USB disks. Partitions: `sda1`, `sda2` |
| `nvme0n1` | NVMe controller 0, namespace 1. Partitions: `nvme0n1p1` (note the `p`) |
| `vda`, `xvda` | Paravirtualised disks in KVM (virtio) and Xen/older AWS VMs |
| `mmcblk0` | SD card or eMMC (phones, Raspberry Pi). Partitions: `mmcblk0p1` |
| `loop0` | Loop device: a regular file presented as a block device |
| `dm-0`, `dm-1` | Device-mapper devices: LVM volumes, LUKS, multipath. Friendlier names live in `/dev/mapper/` |
| `md0` | Software RAID array (mdadm) |
| `sr0` | Optical drive |
| `zram0` | Compressed RAM disk, often used for swap |

### Don't trust `sda` to stay `sda`

Kernel names are handed out in the order devices are detected, and that order can change between boots: add a USB disk, swap a cable, and yesterday's `sdb` is today's `sdc`. For anything that must survive a reboot, like `/etc/fstab`, use a **persistent name**. udev creates symlinks for them:

```bash
$ ls -l /dev/disk/by-uuid/
lrwxrwxrwx 1 root root 15 Nov 22 09:12 0d4c…-a1b2 -> ../../nvme0n1p2
lrwxrwxrwx 1 root root 10 Nov 22 09:12 7e3a…-9f00 -> ../../dm-1
```

- **`by-uuid`**: the UUID of the *file system* (or LUKS header, or swap area), written when you ran `mkfs`. The usual choice for fstab.
- **`by-partuuid`**: the UUID of the GPT *partition*, independent of what's on it.
- **`by-label`**: a human-chosen label, set with `mkfs -L` or `e2label`. Readable, but not guaranteed unique.
- **`by-id`**: built from the hardware model and serial number. Good for "this exact physical disk", e.g. in RAID or ZFS setups.
- **`by-path`**: built from the bus path (which controller, which port). Stable as long as the cabling is.

`blkid` and `lsblk -f` show the UUIDs and labels directly.

### Inspecting block devices

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL   # tree with chosen columns
lsblk -f                                           # file system type, label, UUID, usage
lsblk -t                                           # topology: sector sizes, scheduler
sudo blkid                                         # what signatures are on each device
sudo blockdev --getsize64 /dev/sda                 # exact size in bytes
cat /proc/partitions                               # the kernel's raw list
ls /sys/block/                                     # sysfs: one directory per block device
cat /sys/block/sda/queue/rotational                # 1 = spinning disk, 0 = SSD
cat /sys/block/sda/queue/scheduler                 # I/O scheduler, active one in [brackets]
```

`lsblk` reads everything from sysfs and udev, so it works without root.[^3]

### Sector sizes

Two numbers matter, and `lsblk -t` (columns `LOG-SEC` and `PHY-SEC`) shows both:

- **Logical sector size**: the smallest unit you can address. Almost always 512 bytes, for compatibility.
- **Physical sector size**: the size the drive really works in internally. Usually 4096 bytes on modern disks ("Advanced Format").

If a partition starts in the middle of a physical sector, every 4 KiB write turns into a read-modify-write of two physical sectors. Modern partitioning tools align partitions to 1 MiB boundaries by default, which sidesteps the problem. Just don't override them with odd start sectors.

### What happens between your write and the disk

You rarely need this in practice, but it explains a lot of behaviour. When a program writes to a file:

1. The data lands in the **page cache** in RAM, and `write()` returns. The disk hasn't been touched yet. This is why writes feel instant, and why pulling a USB stick without unmounting loses data.
2. Later, the kernel's writeback threads (or an explicit `fsync()`/`sync`) flush dirty pages. The file system works out which blocks they belong to and hands the **block layer** a request.
3. The block layer queues requests, optionally merges and reorders them with an **I/O scheduler** (`mq-deadline`, `bfq`, `kyber`, or `none`, which is common for fast NVMe drives), and passes them to the driver. Since the multi-queue block layer (blk-mq) became the only path, each CPU can submit to its own queue, which is what lets NVMe drives reach millions of operations per second.[^4]
4. The driver talks to the hardware.

The `free` command's `buff/cache` column is mostly this page cache. It's not "used" memory in the worrying sense; the kernel gives it back the moment programs need it.

### Loop devices: disks made of files

A **loop device** makes a regular file look like a block device. That's how you mount an ISO image (`mount -o loop image.iso /mnt`), how Snap packages are mounted, and the perfect way to experiment with partitions, file systems and LVM without risking a real disk. The lab at the end uses them.

```bash
truncate -s 1G /tmp/fake.img                 # a 1 GiB sparse file
sudo losetup -fP --show /tmp/fake.img        # attach to the first free loop device
/dev/loop0
losetup -l                                   # list attached loop devices
sudo losetup -d /dev/loop0                   # detach
```

## Partitions

A **partition** is a contiguous range of blocks on a disk, described in a **partition table** at the start of the disk. The kernel reads the table and creates a block device for each partition (`sda1`, `sda2`, …), so every partition can hold a different file system, swap, an LVM physical volume, or something else entirely.

### MBR vs GPT

There are two partition table formats you'll meet:[^5]

| | MBR (msdos, DOS) | GPT (GUID Partition Table) |
|---|---|---|
| Age | 1983, IBM PC | Part of the UEFI specification |
| Max disk size | 2 TiB (with 512-byte sectors) | Effectively unlimited (8 ZiB) |
| Partitions | 4 primary, or 3 primary + 1 extended holding logical partitions | 128 by default, all equal |
| Redundancy | One copy in sector 0 | Primary at the start, backup at the end, with CRC checksums |
| Identifiers | One-byte type code (`83` Linux, `8e` LVM, `82` swap) | GUID per type, plus a unique GUID and optional name per partition |
| Boot | BIOS | UEFI (needs an EFI System Partition, FAT32), or BIOS with GRUB and a small BIOS boot partition |

Use GPT for anything new. MBR survives on old BIOS machines and some small devices.

### Partitioning tools

| Tool | Notes |
|---|---|
| `fdisk` | Interactive, handles both MBR and GPT nowadays. The default choice |
| `gdisk` | Like fdisk, GPT only, can convert MBR to GPT |
| `parted` | Interactive or scriptable, can resize partitions |
| `sfdisk` | Scriptable; `sfdisk -d /dev/sda > table.txt` dumps a table you can restore with `sfdisk /dev/sda < table.txt` |
| `cfdisk` | Text-menu UI |

A typical `fdisk` session for a new data disk: `g` (new GPT table), `n` (new partition, accept defaults to use the whole disk), `t` to set the type if it's going to be LVM or swap, `w` to write. After changing the table of a disk that's in use, tell the kernel to re-read it with `partprobe /dev/sdb` (or `partx -u`).

Do you need a partition at all? Not strictly: you can put a file system or an LVM physical volume directly on `/dev/sdb`. But a partition table makes it obvious to people and tools that the disk is in use, so most guides, including Red Hat's, recommend a single partition covering the disk even when it's going to be an LVM physical volume.[^6]

## File systems

A block device is a big array of numbered blocks. A **file system** is the data structure, written onto that array, that turns it into named files in directories, with sizes, owners, permissions and timestamps. It answers three questions:

1. **Where is this file's data?** Which blocks belong to `report.csv`, and in what order?
2. **What's free?** Which blocks can new data go into?
3. **What's in this directory?** Which names exist here, and which file does each name point to?

"Formatting" a partition with `mkfs` means writing an empty version of these structures onto it.

### Inside a classic Unix file system

ext4 is the classic design, and its ideas carry over to most others:[^7]

```text
 ┌────────────┬──────────────┬───────────────┬──────────────────────────────────┐
 │ superblock │ group        │ inode table   │ data blocks                      │
 │ (FS size,  │ descriptors, │ (one fixed-   │ (file contents, and directory    │
 │ block size,│ block & inode│ size record   │ contents: lists of name → inode  │
 │ UUID, …)   │ bitmaps      │ per file)     │ number)                          │
 └────────────┴──────────────┴───────────────┴──────────────────────────────────┘
                                + a journal somewhere on the disk
```

- **Superblock**: metadata about the file system as a whole: size, block size, number of inodes, UUID, label, state. It's so important that ext4 keeps backup copies.
- **Inode** (index node): one per file. It holds everything about the file *except its name*: type, permissions, owner, size, timestamps, link count, and where the data is (in ext4, a tree of *extents*: "blocks 10,000–10,255, then 50,000–50,031").
- **Directory**: just a special file whose contents are a list of `name → inode number` entries.
- **Bitmaps** or free-space trees: which blocks and inodes are free.

That split between names and inodes explains several things that otherwise look odd:

```bash
$ ls -i notes.txt                # the inode number
1835014 notes.txt
$ stat notes.txt                 # the inode's contents
$ ln notes.txt hardlink.txt      # a second name for the same inode (link count 2)
$ ln -s notes.txt symlink.txt    # a new tiny file containing the path "notes.txt"
$ df -i                          # inode usage per file system
```

- **Hard links** are just two directory entries pointing at the same inode. That's why they can't cross file systems (inode numbers are per file system) and why deleting one name leaves the file intact.
- **Deleting a file** (`rm`, which calls `unlink`) removes a name. The inode and its blocks are only freed when the link count reaches zero *and* no process still has the file open.
- **"No space left on device" with free space showing** often means you've run out of *inodes*, not blocks: millions of tiny files on a file system created with too few. `df -i` shows it. ext4 fixes the inode count at `mkfs` time; XFS and Btrfs allocate them dynamically.
- **Moving a file within one file system is instant**, whatever its size: `mv` just writes a new directory entry and removes the old one. Across file systems, it has to copy the data.

### Journaling and copy-on-write

What if the power fails halfway through creating a file, after the inode was allocated but before the directory entry was written? Older file systems were left inconsistent and needed a full `fsck` scan on boot, which on a large disk could take hours.

**Journaling** file systems (ext3, ext4, XFS) first write a description of the change to a journal, then apply it. After a crash, they replay or discard the last few journal entries, which takes seconds. By default ext4 journals only metadata, not file contents (`data=ordered` mode), so a crash can lose recent writes but won't corrupt the structure.

**Copy-on-write** file systems (Btrfs, ZFS, bcachefs) never overwrite data in place. A change is written to new blocks, and then a single pointer is switched to point at them. The old version stays intact until it's no longer referenced, which makes crash consistency natural and snapshots almost free.[^8]

### Which file system?

| | ext4 | XFS | Btrfs | ZFS |
|---|---|---|---|---|
| Typical home | Debian, Ubuntu default | RHEL and derivatives default | Fedora Workstation, openSUSE default | Out-of-tree module (OpenZFS); TrueNAS, Proxmox, Ubuntu option |
| Design | Journaling, extents | Journaling, extents, built for large files and parallel I/O | Copy-on-write | Copy-on-write, combines volume manager and FS |
| Grow | Online | Online | Online | Add devices to the pool |
| Shrink | Offline only (unmounted) | **Not possible** | Online | Not for a pool, generally |
| Snapshots | No (use LVM) | No (use LVM) | Yes, built in | Yes, built in |
| Checksums on data | No (metadata only) | No (metadata only) | Yes | Yes |
| Built-in RAID, compression | No | No | Yes (RAID 5/6 still not recommended) | Yes |

Rules of thumb: ext4 when you just want something boring and robust, XFS for big servers and large files (and accept that it can't shrink, which matters when planning LVM), Btrfs or ZFS when you want snapshots, checksums and compression built in.

There are also file systems you use daily without thinking about it:

- **`vfat`/`exfat`**: FAT32 for the EFI System Partition and USB sticks, exFAT for large removable drives.
- **`tmpfs`**: lives in RAM (and swap). `/tmp` on many distributions, `/dev/shm`, `/run`. Gone at reboot.
- **Pseudo file systems** with no storage at all: `proc` (`/proc`, processes and kernel state), `sysfs` (`/sys`, devices and drivers), `devtmpfs` (`/dev`), `cgroup2`. They're file systems because the kernel finds it convenient to expose information as files.
- **`overlayfs`**: stacks a writable layer over read-only layers. This is how container images work.
- **Network file systems**: NFS, SMB/CIFS. Not block-based at all; the server owns the file system.
- **FUSE** file systems run in user space: `sshfs`, `rclone mount`, NTFS via `ntfs-3g`.

`cat /proc/filesystems` lists the types the running kernel supports right now.[^9]

### Creating and checking file systems

```bash
sudo mkfs.ext4 -L data /dev/sdb1          # ext4 with a label
sudo mkfs.xfs -L data /dev/sdb1           # XFS
sudo mkfs.btrfs -L data /dev/sdb1         # Btrfs
sudo mkswap /dev/sdb2 && sudo swapon /dev/sdb2   # swap is not a file system, but similar

sudo wipefs /dev/sdb1                     # list signatures on a device
sudo wipefs -a /dev/sdb1                  # erase them (destructive!)

sudo fsck /dev/sdb1                       # check (unmounted!) — dispatches to e2fsck, etc.
sudo xfs_repair /dev/sdb1                 # XFS has its own repair tool
sudo tune2fs -l /dev/sdb1                 # dump ext4 superblock info
sudo xfs_info /mountpoint                 # XFS geometry
```

Never run `fsck` on a mounted file system (other than read-only checks); it can corrupt it.

## Mounting and the VFS

Windows gives each file system a drive letter. Linux has **one directory tree**, starting at `/`, and attaches file systems to directories in it. Attaching is **mounting**; the directory is the **mount point**.

```bash
sudo mkdir -p /srv/data
sudo mount /dev/sdb1 /srv/data          # file system type is auto-detected
sudo mount -o ro,noexec /dev/sdb1 /srv/data
findmnt                                 # the mount tree
findmnt /srv/data                       # what's mounted here, with options
df -hT                                  # usage and type per mounted file system
sudo umount /srv/data                   # "target is busy"? → lsof +f -- /srv/data, or fuser -vm
```

Whatever was in `/srv/data` before the mount is hidden, not deleted, and reappears after `umount`. (A classic puzzle: `du` and `df` disagree because files were written into a directory before something was mounted on top of it.)

What makes this work is the kernel's **Virtual File System** (VFS) layer. Every program uses the same system calls (`open`, `read`, `write`, `stat`, `rename`), and the VFS routes each one to whichever file system owns that part of the tree, through a common set of operations every file system implements.[^10] That's why `cat` works identically on ext4, NFS, `/proc` and a FUSE-mounted Google Drive.

A **bind mount** makes a directory visible at a second place in the tree: `mount --bind /srv/data/www /var/www`. Container runtimes and Kubernetes `hostPath` volumes use this constantly.

### Making mounts permanent: `/etc/fstab`

Mounts done by hand are gone at reboot. Permanent ones go in `/etc/fstab`, one line per file system, six fields:[^11]

```text
# <device>                                 <mount point> <type> <options>          <dump> <pass>
UUID=7e3a9c1e-…-9f00                       /             xfs    defaults           0      0
UUID=0d4c51aa-…-a1b2                       /boot         ext4   defaults           0      2
/dev/vg0/home                              /home         ext4   defaults,noatime   0      2
UUID=b21f…                                 none          swap   defaults           0      0
UUID=55aa…                                 /srv/backup   ext4   defaults,nofail    0      2
tmpfs                                      /tmp          tmpfs  size=2G,mode=1777  0      0
```

- **Device**: a `UUID=`, `LABEL=`, `PARTUUID=` or a stable path. LVM paths like `/dev/vg0/home` are stable too, because they're named after the volume group and volume. Avoid `/dev/sdX`.
- **Options**: `defaults` is `rw,suid,dev,exec,auto,nouser,async`. Useful extras: `noatime` (don't update access times on every read), `ro`, `noexec`/`nosuid`/`nodev` for hardening, and **`nofail`** for disks that might be absent. Without `nofail`, a missing non-root disk drops the boot into emergency mode.
- **Dump**: legacy, leave it at 0.
- **Pass**: `fsck` order at boot. `1` for root, `2` for others, `0` to skip. XFS and Btrfs don't use boot-time fsck, so `0`.

On systemd systems, fstab entries are converted into `.mount` units at boot. After editing it, always test before rebooting:

```bash
sudo systemctl daemon-reload
sudo findmnt --verify       # syntax and sanity check
sudo mount -a               # mount everything not yet mounted; errors show now, not at boot
```

## Logical Volume Management (LVM)

Partitions have a fundamental limitation: they're fixed ranges of one disk. If `/var` fills up and the free space is on the other side of `/home`, or on a different disk, a partition can't use it. LVM adds a layer of indirection that removes that limitation.

### The three building blocks

```text
  Logical volumes   ┌──────────── lv_root ───────────┐┌─── lv_home ───┐┌ free ┐
  (what you mkfs)   │          /dev/vg0/root         ││ /dev/vg0/home ││      │
                    └────────────────────────────────┘└───────────────┘└──────┘
                    ┌───────────────────────── vg0 ─────────────────────────────┐
  Volume group      │   one pool of extents (4 MiB each by default)             │
                    └───────────────────────────────────────────────────────────┘
                    ┌──────── PV ──────────┐┌──────── PV ─────────┐┌─── PV ─────┐
  Physical volumes  │      /dev/sda2       ││      /dev/sdb1      ││  /dev/sdc  │
                    └──────────────────────┘└─────────────────────┘└────────────┘
```

- **Physical volume (PV)**: a block device that LVM is allowed to use: a partition, a whole disk, a RAID array, a LUKS device, even a loop device. `pvcreate` writes a small LVM header onto it.[^12]
- **Volume group (VG)**: a pool made from one or more PVs. Its space is divided into equal-sized **physical extents** (PE), 4 MiB by default.
- **Logical volume (LV)**: a block device carved out of the VG's pool. It behaves like a partition (you put a file system on it), but its extents can come from any PV in the group, in any order, and can be added or moved later. Red Hat's documentation puts it as a virtual block storage device that a file system, database or application can use.[^13]

Under the hood, an LV is a **device-mapper** table: a kernel-level map saying "LV block 0–1023 → PV `sda2` blocks 2048–3071, LV block 1024–2047 → PV `sdb1` blocks …".[^14] That's why LVs show up as `dm-N` and in `/dev/mapper/vg0-root`, with `/dev/vg0/root` as a friendlier symlink. You can see the raw map with `sudo dmsetup table`.

### What LVM gives you

- **Resize without repartitioning.** Grow an LV into any free space in the VG, online, then grow the file system on it.
- **Span disks.** An LV can be bigger than any single disk in the group.
- **Add capacity.** New disk → `pvcreate` → `vgextend` → the pool is bigger.
- **Move data live.** `pvmove` migrates extents off a disk while the file system stays mounted, so you can replace a disk without downtime.
- **Snapshots.** Point-in-time copies of an LV, for consistent backups or a safety net before an upgrade.
- **Thin provisioning.** Create LVs whose declared size exceeds the real space, allocating only what's actually written.
- **RAID, striping, caching.** LVM can mirror or stripe LVs and use an SSD as a cache for a slow disk.[^13]

The costs: one more layer to understand when things go wrong, and if a PV in a non-redundant VG dies, every LV with extents on it is damaged. LVM spreads data across disks; it doesn't protect it unless you ask for RAID.

### The commands

LVM commands are pleasingly regular. Each object has the same family of verbs:

| | Physical volume | Volume group | Logical volume |
|---|---|---|---|
| Create | `pvcreate` | `vgcreate` | `lvcreate` |
| Short list | `pvs` | `vgs` | `lvs` |
| Details | `pvdisplay` | `vgdisplay` | `lvdisplay` |
| Change size | `pvresize` | `vgextend` / `vgreduce` | `lvextend` / `lvreduce` / `lvresize` |
| Remove | `pvremove` | `vgremove` | `lvremove` |
| Other | `pvmove` | `vgrename`, `vgchange -ay` | `lvrename`, `lvconvert` |

The everyday workflow:

```bash
# Prepare disks
sudo pvcreate /dev/sdb1 /dev/sdc1

# Pool them
sudo vgcreate vg_data /dev/sdb1 /dev/sdc1

# Carve out volumes
sudo lvcreate -n web  -L 20G      vg_data      # fixed size
sudo lvcreate -n logs -l 50%FREE  vg_data      # half of what's left
sudo lvcreate -n db   -l 100%FREE vg_data      # the rest

# Use one
sudo mkfs.xfs /dev/vg_data/web
sudo mount /dev/vg_data/web /srv/web

# See what you have
sudo pvs; sudo vgs; sudo lvs
```

### Growing a volume (the common case)

The single most common LVM operation: a file system is filling up.

```bash
sudo vgs                                           # is there free space (VFree) in the VG?
sudo lvextend -r -L +10G /dev/vg_data/web          # grow LV by 10 GiB and resize the FS
sudo lvextend -r -l +100%FREE /dev/vg_data/web     # or take everything that's left
```

The **`-r`** (`--resizefs`) flag is the important part. Growing the LV only makes the block device bigger; the file system on it still thinks it's the old size until it's told otherwise. `-r` runs the right tool for you (`resize2fs` for ext4, `xfs_growfs` for XFS). If you forget it, run that tool yourself afterwards. Both ext4 and XFS grow online, while mounted.

No free space in the VG? Add a disk first:

```bash
sudo pvcreate /dev/sdd
sudo vgextend vg_data /dev/sdd
```

And on a cloud VM where you enlarged the virtual disk itself, work from the bottom up: grow the partition (`growpart /dev/sda 2`), then the PV (`pvresize /dev/sda2`), then the LV and file system (`lvextend -r`).

### Shrinking a volume (be careful)

Shrinking is the dangerous direction, because the file system must be made smaller *before* the block device under it, or you chop off the end of your data.

- **XFS can't be shrunk at all.** The only way is backup, recreate smaller, restore.
- **ext4 can be shrunk, but only unmounted.** `lvreduce -r` handles the ordering for you (it shrinks the file system first, then the LV):

```bash
sudo umount /srv/logs
sudo lvreduce -r -L 5G /dev/vg_data/logs
sudo mount /srv/logs
```

Never run plain `lvreduce` without `-r` on a volume with a file system unless you've already shrunk the file system yourself, to a size no larger than the new LV.

### Removing a disk from a volume group

```bash
sudo pvmove /dev/sdb1                 # move all extents off sdb1 onto other PVs (online)
sudo vgreduce vg_data /dev/sdb1       # take it out of the group
sudo pvremove /dev/sdb1               # wipe the LVM header
```

`pvmove` needs enough free extents on the remaining PVs to hold everything.

### Snapshots

A classic LVM snapshot is a copy-on-write view of an LV at one moment. Creating it is instant; it only needs space for blocks that change *afterwards*:

```bash
sudo lvcreate -s -n web_snap -L 2G /dev/vg_data/web    # 2 GiB to store changes
sudo mount -o ro,nouuid /dev/vg_data/web_snap /mnt/snap  # nouuid needed for XFS
# … back up /mnt/snap at leisure, while /srv/web keeps changing …
sudo umount /mnt/snap
sudo lvremove /dev/vg_data/web_snap
```

Or use it as an undo button: take a snapshot before a risky upgrade, and if it goes wrong, `lvconvert --merge /dev/vg_data/web_snap` rolls the origin back (the merge completes the next time the origin is activated, e.g. after unmounting or a reboot).

Two caveats: if more data changes than the snapshot has room for, the snapshot becomes invalid, so watch `Data%` in `lvs`. And classic snapshots slow down writes to the origin, since every first write to a block has to copy the old contents first. They're meant to be short-lived.

### Thin provisioning

A normal LV reserves all its extents up front. A **thin pool** is an LV that hands out space only as it's written, so you can create thin volumes whose total declared size exceeds the pool:[^15]

```bash
sudo lvcreate --type thin-pool -n pool0 -L 50G vg_data
sudo lvcreate --thin -n vm1 -V 40G vg_data/pool0
sudo lvcreate --thin -n vm2 -V 40G vg_data/pool0     # 80 GiB promised from 50 GiB
sudo lvs                                             # watch Data% of pool0
```

Thin snapshots (`lvcreate -s vg_data/vm1`, no size needed) are cheap and don't slow the origin the way classic ones do, which is why VM platforms and Kubernetes CSI drivers like TopoLVM use thin pools. The danger is overcommit: if the pool reaches 100%, writes to every thin volume in it fail. Monitor it, or set `thin_pool_autoextend_threshold` in `/etc/lvm/lvm.conf` so LVM grows the pool automatically while the VG has free space.

## Two more layers worth knowing

These aren't LVM, but they sit in the same stack and you'll see them in `lsblk`.

**Software RAID with mdadm.** Combines several disks into one block device (`/dev/md0`) with redundancy (RAID 1, 5, 6, 10) or striping (RAID 0). Status lives in `/proc/mdstat`.

```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
cat /proc/mdstat
```

LVM can do RAID itself (`lvcreate --type raid1 -m 1 …`), and both use the same kernel MD code underneath. mdadm is more common for whole-disk arrays; LVM RAID is handy when you want redundancy per volume.

**Encryption with LUKS (dm-crypt).** Another device-mapper target: you open an encrypted block device with a passphrase, and get a decrypted block device in `/dev/mapper/` to put LVM or a file system on.

```bash
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup open /dev/sdb1 secure       # → /dev/mapper/secure
sudo mkfs.ext4 /dev/mapper/secure
```

Order matters for what's protected and what's flexible. The common laptop layout, "LVM on LUKS" (as in the `lsblk` example at the top), encrypts one partition and puts LVM inside, so one passphrase unlocks root, home and swap.

## Hands-on lab: build the whole stack on fake disks

Everything below runs on loop devices backed by files in `/tmp`, so it can't damage your real disks. You need root and the `lvm2` package. Use a VM if you're nervous.

```bash
# 1. Two fake 1 GiB disks
truncate -s 1G /tmp/disk1.img /tmp/disk2.img
D1=$(sudo losetup -fP --show /tmp/disk1.img)
D2=$(sudo losetup -fP --show /tmp/disk2.img)
echo "$D1 $D2"                                    # e.g. /dev/loop0 /dev/loop1
lsblk "$D1" "$D2"

# 2. A GPT partition table with one LVM partition on the first disk
sudo sfdisk "$D1" <<'EOF'
label: gpt
type=E6D6D379-F507-44C2-A23C-238F2A3DF928
EOF
lsblk "$D1"                                       # now has a ...p1 partition
P1="${D1}p1"

# 3. Physical volumes: the partition, and the second disk unpartitioned
sudo pvcreate "$P1" "$D2"
sudo pvs

# 4. One volume group across both
sudo vgcreate vglab "$P1" "$D2"
sudo vgs                                          # ~2 GiB, in 4 MiB extents

# 5. A logical volume bigger than either disk alone
sudo lvcreate -n data -L 1.5G vglab
sudo lvs -o +devices                              # extents come from both PVs
lsblk                                             # see the tree

# 6. A file system, mounted
sudo mkfs.ext4 -L labdata /dev/vglab/data
sudo mkdir -p /mnt/lab
sudo mount /dev/vglab/data /mnt/lab
df -hT /mnt/lab
echo "hello" | sudo tee /mnt/lab/hello.txt

# 7. Snapshot, then change the original
sudo lvcreate -s -n data_snap -L 200M vglab/data
echo "changed" | sudo tee /mnt/lab/hello.txt
sudo mkdir -p /mnt/snap
sudo mount -o ro /dev/vglab/data_snap /mnt/snap
cat /mnt/lab/hello.txt /mnt/snap/hello.txt        # "changed" vs "hello"
sudo umount /mnt/snap
sudo lvremove -y vglab/data_snap

# 8. Grow the LV and the file system online, into the remaining free space
sudo lvextend -r -l +100%FREE vglab/data
df -h /mnt/lab                                    # bigger, still mounted

# 9. Look at what LVM really is
sudo dmsetup table                                # the device-mapper map
ls -l /dev/vglab/ /dev/mapper/

# 10. Clean up, top to bottom
sudo umount /mnt/lab
sudo vgremove -y vglab                            # removes its LVs too
sudo pvremove "$P1" "$D2"
sudo losetup -d "$D1" "$D2"
rm /tmp/disk1.img /tmp/disk2.img
```

The GUID in step 2 is the GPT type code for "Linux LVM". It's optional; LVM works on a partition of any type, but setting it tells other tools what's there.

Worth trying next, on the same setup: build it with XFS instead of ext4 and try to shrink it (you can't), `pvmove` everything off the second disk and `vgreduce` it out, or put LUKS between the partition and LVM.

## Troubleshooting: common puzzles

| Symptom | Likely cause | How to check / fix |
|---|---|---|
| `No space left on device`, but `df -h` shows free space | Out of inodes | `df -i`; delete small files, or recreate the FS with more inodes |
| `df` shows the disk full, `du` finds much less | Deleted files still held open by a process | `sudo lsof +L1`; restart the process (classic with rotated logs) |
| `du` shows more than `df` used, or files "vanish" | Something is mounted over a directory that had files in it | Unmount and look, or `mount --bind / /mnt/root` and inspect underneath |
| Boot drops into emergency mode after adding a disk | fstab entry for a missing device, or a typo | Fix fstab; add `nofail`; always run `mount -a` before rebooting |
| `lvextend` worked, `df` shows the old size | File system not resized | `resize2fs /dev/vg/lv` or `xfs_growfs /mountpoint`; use `-r` next time |
| New partition doesn't show up | Kernel hasn't re-read the table | `partprobe /dev/sdX` or `partx -u /dev/sdX` |
| `umount: target is busy` | A process has a file open or a shell is `cd`'d inside | `lsof +f -- /mnt/x` or `fuser -vm /mnt/x` |
| LVs missing after attaching disks from another machine | VG not activated | `vgscan`, then `vgchange -ay` |
| Device names changed after reboot | Kernel names aren't stable | Use UUIDs, labels or LVM paths in fstab |

## Cheat sheet

```text
Layers (bottom → top):  disk → partition → [RAID] → [LUKS] → [LVM PV→VG→LV] → file system → mount
Block vs char device:   b = fixed blocks, random access, page cache | c = byte stream
See the stack:          lsblk · lsblk -f · blkid · findmnt · df -hT · df -i
Stable names:           /dev/disk/by-uuid | by-partuuid | by-label | by-id | /dev/<vg>/<lv>
Fake disks:             truncate -s 1G f.img; losetup -fP --show f.img; losetup -d /dev/loopN
Partition tables:       GPT (use this) | MBR (≤2 TiB, 4 primary)
Partition tools:        fdisk · gdisk · parted · sfdisk (scripts) · partprobe (re-read)
Make FS:                mkfs.ext4 | mkfs.xfs | mkfs.btrfs ; -L label ; mkswap + swapon
Check FS:               fsck (unmounted!) | xfs_repair ; wipefs to see/erase signatures
Inode = file minus its name ; directory = name → inode list ; hard link = 2nd name
Grow/shrink:            ext4 grow online, shrink offline | XFS grow only | Btrfs both online
Mount:                  mount DEV DIR · umount DIR · mount --bind A B
fstab fields:           device  mountpoint  type  options  dump  pass   (test: mount -a)
LVM objects:            PV (pvcreate) → VG (vgcreate, extents) → LV (lvcreate) → mkfs
LVM list:               pvs · vgs · lvs -o +devices
Grow LV + FS:           lvextend -r -L +10G /dev/vg/lv     (add disk: pvcreate + vgextend)
Shrink LV + FS:         ext4 only, unmounted: lvreduce -r -L 5G /dev/vg/lv
Replace disk:           pvmove /dev/old → vgreduce vg /dev/old → pvremove /dev/old
Snapshot:               lvcreate -s -n snap -L 2G vg/lv ; rollback: lvconvert --merge vg/snap
Thin:                   lvcreate --type thin-pool … ; lvcreate --thin -V 40G vg/pool ; watch Data%
Under it all:           device-mapper → dmsetup ls / dmsetup table
```

If you came here from Kubernetes: a PersistentVolume backed by a CSI driver like TopoLVM is exactly this stack, automated. The driver runs `lvcreate` on the node, `mkfs`, and bind-mounts the result into the Pod. My [CKA storage post](/posts/cka-storage/) picks up from there.

## References

[^1]: [Configuring LVM logical volumes using the web console](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_systems_using_the_rhel_9_web_console/using-the-web-console-for-configuring-lvm-logical-volumes_system-management-using-the-rhel-8-web-console), Red Hat Enterprise Linux 9 documentation.
[^2]: [Linux allocated devices (4.x+ version)](https://docs.kernel.org/admin-guide/devices.html), Linux kernel documentation.
[^3]: [lsblk(8)](https://man7.org/linux/man-pages/man8/lsblk.8.html), Linux manual page.
[^4]: [Multi-Queue Block IO Queueing Mechanism (blk-mq)](https://docs.kernel.org/block/blk-mq.html), Linux kernel documentation.
[^5]: [Partitioning](https://wiki.archlinux.org/title/Partitioning), ArchWiki.
[^6]: [Managing LVM physical volumes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_logical_volumes/managing-lvm-physical-volumes_configuring-and-managing-logical-volumes), Red Hat Enterprise Linux 9 documentation.
[^7]: [ext4 Data Structures and Algorithms](https://docs.kernel.org/filesystems/ext4/index.html), Linux kernel documentation.
[^8]: [File systems](https://wiki.archlinux.org/title/File_systems), ArchWiki.
[^9]: [filesystems(5)](https://man7.org/linux/man-pages/man5/filesystems.5.html), Linux manual page.
[^10]: [Overview of the Linux Virtual File System](https://docs.kernel.org/filesystems/vfs.html), Linux kernel documentation.
[^11]: [fstab(5)](https://man7.org/linux/man-pages/man5/fstab.5.html), Linux manual page.
[^12]: [LVM](https://wiki.archlinux.org/title/LVM), ArchWiki.
[^13]: [Configuring and managing logical volumes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/configuring_and_managing_logical_volumes/index), Red Hat Enterprise Linux 9 documentation.
[^14]: [Device Mapper](https://docs.kernel.org/admin-guide/device-mapper/index.html), Linux kernel documentation.
[^15]: [lvmthin(7)](https://man7.org/linux/man-pages/man7/lvmthin.7.html), Linux manual page.
