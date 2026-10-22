# Linux Storage Fundamentals

A beginner-friendly guide to how Linux sees disks, partitions, filesystems and folders.

**Series:**
1. **Storage fundamentals** (this file)
2. [Partitioning and formatting](02-partitioning-and-formatting.md)
3. [Mounting a new disk and the root disk](03-mounting-and-root-disk.md)

---

## 1. The big picture

Linux storage is built in **layers**. Each layer sits on top of the previous one:

```
Physical disk (SSD / HDD / NVMe / USB)
        │
        ▼
Block device            →  /dev/sda, /dev/nvme0n1
        │
        ▼
Partition table + partitions   →  /dev/sda1, /dev/sda2
        │
        ▼
(Optional) LVM / RAID / encryption (LUKS)
        │
        ▼
Filesystem              →  ext4, xfs, btrfs, ...
        │
        ▼
Mount point             →  /, /home, /mnt/data
        │
        ▼
Files and folders you actually use
```

The key idea: **a disk is useless until it has a filesystem AND is mounted somewhere in the directory tree.**

---

## 2. Linux has no "C:" or "D:" drives

On Windows, each disk/partition gets a drive letter. On Linux there is **one single tree** that starts at `/` (called **root**). Every disk or partition is **attached** ("mounted") to a folder inside this tree.

Example:

```
/                    ← root filesystem (on /dev/nvme0n1p2)
├── boot/
│   └── efi/         ← EFI partition (on /dev/nvme0n1p1)
├── home/            ← could be a separate partition or disk
├── mnt/
│   └── data/        ← a second disk mounted here
└── var/
```

So if you plug in a new disk, you don't get "E:". You choose a folder (for example `/mnt/data`) and say "this disk lives here".

---

## 3. Everything is a file: `/dev`

Hardware is exposed as special files in `/dev`. Storage devices are called **block devices** because data is read and written in blocks.

### Naming conventions

| Name | Meaning |
|------|---------|
| `/dev/sda`, `/dev/sdb`, `/dev/sdc` | SATA / SCSI / USB disks (a = first, b = second...) |
| `/dev/sda1`, `/dev/sda2` | Partitions **1** and **2** of disk `sda` |
| `/dev/nvme0n1` | First NVMe SSD (controller 0, namespace 1) |
| `/dev/nvme0n1p1`, `/dev/nvme0n1p2` | Partitions of that NVMe SSD (note the `p`) |
| `/dev/vda`, `/dev/vdb` | Virtual disks (KVM / cloud VMs) |
| `/dev/mmcblk0`, `/dev/mmcblk0p1` | SD cards / eMMC (Raspberry Pi) |
| `/dev/sr0` | CD/DVD drive |
| `/dev/loop0` | Loop devices (files or snaps pretending to be disks) |
| `/dev/mapper/...` | LVM volumes, encrypted volumes |

> ⚠️ Names like `sda` / `sdb` **can change between reboots** (depends on detection order). That is why we use **UUIDs** in `/etc/fstab` (see tutorial 3).

---

## 4. Looking at your storage (essential commands)

### `lsblk`: list block devices (your best friend)

```bash
lsblk
```

Example output:

```
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 931.5G  0 disk
└─sda1        8:1    0 931.5G  0 part /mnt/data
nvme0n1     259:0    0 476.9G  0 disk
├─nvme0n1p1 259:1    0   512M  0 part /boot/efi
└─nvme0n1p2 259:2    0 476.4G  0 part /
```

How to read it:
- `TYPE = disk` → the whole physical disk
- `TYPE = part` → a partition
- `MOUNTPOINTS` → where it is attached in the tree. **Empty = not mounted.**

Useful variant showing filesystem, label and UUID:

```bash
lsblk -f
```

### `df -h`: free space on mounted filesystems

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p2  468G  120G  325G  27% /
/dev/sda1       916G  200G  670G  18% /mnt/data
tmpfs           7.8G     0  7.8G   0% /dev/shm
```

### `du`: how much space a folder uses

```bash
du -sh /var/log          # total size of one folder
du -h --max-depth=1 /    # size of each top-level folder
```

### `blkid`: UUIDs and filesystem types

```bash
sudo blkid
```

### `findmnt`: nicely displays the mount tree

```bash
findmnt
```

### `sudo fdisk -l`: partition tables of all disks

```bash
sudo fdisk -l
```

---

## 5. Partitions

A **partition** is a slice of a disk. Why use them?

- Separate the system from your data (reinstall the OS without losing files)
- Use different filesystems on the same disk
- Dual boot (Windows + Linux)
- Limit damage (a full `/var/log` won't crash the whole system)

### Partition table types

| | **MBR** (msdos) | **GPT** |
|---|---|---|
| Age | Old (1983) | Modern |
| Max disk size | 2 TB | 9.4 ZB (practically unlimited) |
| Max primary partitions | 4 (use "extended" to get more) | 128 by default |
| Boot mode | Legacy BIOS | UEFI (also works with BIOS via a BIOS boot partition) |
| Recommendation | Only for old hardware | **Use this by default** |

### Typical partitions on a modern UEFI system

| Partition | Size | Filesystem | Mounted on | Purpose |
|-----------|------|------------|------------|---------|
| EFI System Partition (ESP) | 300 MB – 1 GB | FAT32 | `/boot/efi` | Bootloader files |
| Root | 30 GB+ | ext4 / xfs / btrfs | `/` | The system |
| Swap (optional) | 2–8 GB | swap | (none) | Overflow memory / hibernation |
| Home (optional) | Rest of the disk | ext4 | `/home` | User files |

---

## 6. Filesystems

A **filesystem** is the structure written on a partition that organizes files, names, permissions and where the data physically lives. Without one, a partition is just raw bytes.

| Filesystem | Good for | Notes |
|------------|----------|-------|
| **ext4** | General purpose, default on most distros | Stable, simple, reliable. **Pick this if unsure.** |
| **xfs** | Large files, servers (default on RHEL/Fedora server) | Can grow but **cannot shrink** |
| **btrfs** | Snapshots, compression, advanced features | Default on openSUSE / Fedora desktop |
| **zfs** | NAS, big storage pools | Not in the kernel (licence), needs extra install |
| **FAT32 (vfat)** | EFI partition, USB sticks for compatibility | 4 GB max file size |
| **exFAT** | USB drives shared with Windows/macOS | No 4 GB limit |
| **NTFS** | Windows disks | Works through `ntfs-3g` / `ntfs3` |
| **swap** | Swap space | Not a real filesystem, special format |

### Inodes (quick concept)

Each file is described by an **inode** (metadata: owner, permissions, size, location of data blocks). The filename is just a label pointing to an inode. You can run out of inodes before running out of space (many tiny files):

```bash
df -i
```

---

## 7. The directory tree (FHS)

Linux follows the **Filesystem Hierarchy Standard**. What each top-level folder is for:

| Folder | Purpose |
|--------|---------|
| `/` | The root of everything |
| `/bin`, `/sbin`, `/lib` | Essential programs and libraries (often symlinks to `/usr/...` on modern distros) |
| `/boot` | Kernel, initramfs, bootloader files |
| `/dev` | Device files (`sda`, `null`, `tty`...) |
| `/etc` | System-wide **configuration** files (includes `fstab`) |
| `/home` | Users' personal folders (`/home/alice`) |
| `/root` | Home folder of the **root user** (not the same as `/`!) |
| `/mnt` | Convention for **temporary / manual** mounts |
| `/media` | Auto-mounted removable media (USB sticks, DVDs) |
| `/opt` | Optional third-party software |
| `/proc`, `/sys` | Virtual filesystems exposing kernel and hardware info |
| `/run` | Runtime data since boot (RAM) |
| `/srv` | Data served by this machine (web, FTP) |
| `/tmp` | Temporary files (often cleared at reboot) |
| `/usr` | Most installed programs and libraries |
| `/var` | Variable data: logs (`/var/log`), caches, databases, mail |

> 💡 Folders like `/proc`, `/sys`, `/dev`, `/run` don't live on your disk. They are **virtual** and created by the kernel in RAM.

---

## 8. Mounting: the core concept

**Mounting** = attaching a filesystem to a folder (the **mount point**) so its content becomes visible there.

```
Before:   /mnt/data/             → empty folder
Mount:    sudo mount /dev/sda1 /mnt/data
After:    /mnt/data/             → shows the files of /dev/sda1
```

Important facts:
- The mount point must be an **existing folder**.
- If the folder already had files, they are **hidden** (not deleted) while something is mounted on it.
- Unmounting (`umount`) makes the original content visible again.
- Mounts done with `mount` **disappear after reboot** unless you add them to `/etc/fstab`.

---

## 9. Quick glossary

| Term | Meaning |
|------|---------|
| **Block device** | A device read/written in blocks (disk, partition) |
| **Partition** | A slice of a disk |
| **Filesystem** | Structure that organizes files on a partition |
| **Mount point** | Folder where a filesystem is attached |
| **UUID** | Unique ID of a filesystem, stable across reboots |
| **fstab** | `/etc/fstab`, the file listing what to mount at boot |
| **Root (`/`)** | Top of the directory tree |
| **Root user** | The administrator account |
| **ESP** | EFI System Partition |
| **LVM** | Logical Volume Manager: flexible resizable "virtual partitions" |
| **Swap** | Disk space used as extra memory |

---

➡️ Next: [Partitioning and formatting](02-partitioning-and-formatting.md)
