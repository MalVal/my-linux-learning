# Partitioning and Formatting a Disk

Hands-on guide to create partitions and filesystems on Linux.

**Series:**
1. [Storage fundamentals](01-linux-storage-fundamentals.md)
2. **Partitioning and formatting** (this file)
3. [Mounting a new disk and the root disk](03-mounting-and-root-disk.md)

---

## ⚠️ Read this first

> **Partitioning and formatting DESTROY data.** One wrong device name (`sda` instead of `sdb`) and you can wipe your system disk.
>
> Before every destructive command:
> 1. Run `lsblk` and identify the disk by **size** and **model**.
> 2. Make sure it has **no mountpoint** you care about.
> 3. Re-read the device name in your command.

Find the right disk:

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,MOUNTPOINTS
```

In this tutorial, the new empty disk is **`/dev/sdb`**. Replace it with yours.

---

## 1. Workflow overview

```
1. Identify the disk         lsblk
2. Create partition table    GPT (or MBR)
3. Create partition(s)       fdisk / parted / gdisk
4. Create filesystem         mkfs.ext4 / mkfs.xfs ...
5. Mount it                  (see tutorial 3)
```

> 💡 For a **dedicated data disk**, you can skip partitioning and format the whole disk (`mkfs.ext4 /dev/sdb`). It works, but a partition table is cleaner and safer (other OSes and tools recognize it).

---

## 2. Tools

| Tool | Type | Notes |
|------|------|-------|
| `fdisk` | Interactive, text | Standard, supports GPT and MBR |
| `gdisk` | Interactive, text | GPT-focused (package `gdisk`) |
| `parted` | Interactive or **one-liner** | Great for scripts |
| `cfdisk` | Text UI with menus | Easiest for beginners |
| `GParted` / `gnome-disks` | Graphical | Good if you have a desktop |

We'll use **`fdisk`** (universal) and show the `parted` version too.

---

## 3. Create a partition with `fdisk`

```bash
sudo fdisk /dev/sdb
```

You are now in the interactive prompt. Nothing is written until you type `w`.

### Step by step

| Key | Action |
|-----|--------|
| `p` | Print the current partition table (should be empty) |
| `g` | Create a new **GPT** partition table |
| `n` | New partition |
| ↳ partition number | Press **Enter** (default 1) |
| ↳ first sector | Press **Enter** (default) |
| ↳ last sector | Press **Enter** to use the whole disk, or type `+100G` for 100 GB |
| `t` | (Optional) Change partition type |
| `p` | Check the result |
| `w` | **Write** changes and exit |
| `q` | Quit **without** saving (if you made a mistake) |

Example session:

```
Command (m for help): g
Created a new GPT disklabel (GUID: ...).

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-..., default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (...): 

Created a new partition 1 of type 'Linux filesystem' and of size 931.5 GiB.

Command (m for help): w
The partition table has been altered.
Syncing disks.
```

Check it:

```bash
lsblk /dev/sdb
```

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0 931.5G  0 disk
└─sdb1   8:17   0 931.5G  0 part
```

### MBR instead of GPT?

In `fdisk`, press `o` instead of `g` to create an MBR (DOS) table. Only do this for old BIOS systems or disks under 2 TB with special needs.

---

## 4. The same with `parted` (one-liners)

```bash
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart primary ext4 0% 100%
```

Two partitions (50 / 50):

```bash
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart data1 ext4 0% 50%
sudo parted /dev/sdb --script mkpart data2 ext4 50% 100%
```

Show the result:

```bash
sudo parted /dev/sdb print
```

> Note: `parted` ext4 here is only a *hint* for the partition type. It does **not** create the filesystem. You still need `mkfs`.

---

## 5. Create the filesystem (format)

Now format the **partition** (`/dev/sdb1`), not the disk.

### ext4 (recommended default)

```bash
sudo mkfs.ext4 /dev/sdb1
```

With a **label** (handy name):

```bash
sudo mkfs.ext4 -L mydata /dev/sdb1
```

### xfs

```bash
sudo mkfs.xfs /dev/sdb1
```

Add `-f` if it complains about an existing filesystem and you are sure.

### btrfs

```bash
sudo mkfs.btrfs -L mydata /dev/sdb1
```

### FAT32 (for USB sticks / EFI)

```bash
sudo mkfs.vfat -F 32 -n MYUSB /dev/sdb1
```

### exFAT (USB shared with Windows/macOS)

```bash
sudo apt install exfatprogs     # Debian/Ubuntu
sudo mkfs.exfat -L MYUSB /dev/sdb1
```

### NTFS

```bash
sudo apt install ntfs-3g
sudo mkfs.ntfs -f -L MYDISK /dev/sdb1
```

### Verify

```bash
lsblk -f /dev/sdb
```

```
NAME   FSTYPE FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sdb
└─sdb1 ext4   1.0   mydata 3a1f6e1c-8d2b-4b0e-9c5a-1f2e3d4c5b6a
```

You now have a **UUID**: you'll need it for `/etc/fstab`.

---

## 6. Swap partition (optional)

Swap is space used when RAM is full (and for hibernation).

```bash
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
swapon --show
```

Make it permanent in `/etc/fstab`:

```
UUID=<swap-uuid>  none  swap  sw  0  0
```

### Swap **file** (no partition needed)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

fstab line:

```
/swapfile  none  swap  sw  0  0
```

> On btrfs or some filesystems, `fallocate` swap files need special handling. Check your distro docs.

---

## 7. Useful options when formatting ext4

| Option | Effect |
|--------|--------|
| `-L name` | Set a label |
| `-m 1` | Reserve only 1% for root (default is 5%, wasteful on big data disks) |
| `-T largefile` | Fewer inodes, better for huge files (videos, backups) |

Example for a big data disk:

```bash
sudo mkfs.ext4 -L bigdata -m 1 /dev/sdb1
```

Change the reserved percentage later:

```bash
sudo tune2fs -m 1 /dev/sdb1
```

---

## 8. Checking and repairing a filesystem

**Always unmount first.**

```bash
sudo umount /dev/sdb1
sudo fsck /dev/sdb1          # ext4 (auto-detects)
sudo xfs_repair /dev/sdb1    # xfs
sudo btrfs check /dev/sdb1   # btrfs
```

Never run `fsck` on a mounted filesystem.

---

## 9. Quick intro to LVM (optional but powerful)

**LVM** adds a flexible layer between disks and filesystems. You can combine several disks, resize volumes live, and take snapshots.

```
Physical disks → PV (Physical Volume)
PVs            → VG (Volume Group)   = a pool of storage
VG             → LV (Logical Volume) = like a partition, but resizable
LV             → filesystem
```

Create it:

```bash
sudo pvcreate /dev/sdb1                  # 1. mark partition as PV
sudo vgcreate vg_data /dev/sdb1          # 2. create a volume group
sudo lvcreate -n lv_files -L 200G vg_data   # 3. create a 200 GB logical volume
sudo mkfs.ext4 /dev/vg_data/lv_files     # 4. format it
```

Inspect:

```bash
sudo pvs
sudo vgs
sudo lvs
```

Resize later (grow by 50 GB and resize the filesystem at once):

```bash
sudo lvextend -r -L +50G /dev/vg_data/lv_files
```

The device is then `/dev/vg_data/lv_files` (or `/dev/mapper/vg_data-lv_files`).

---

## 10. Cheat sheet

```bash
lsblk -f                                  # see disks, filesystems, UUIDs
sudo fdisk /dev/sdX                       # partition (g, n, w)
sudo mkfs.ext4 -L label /dev/sdX1         # format
sudo blkid /dev/sdX1                      # get UUID
sudo fsck /dev/sdX1                       # check (unmounted!)
```

---

➡️ Next: [Mounting a new disk and the root disk](03-mounting-and-root-disk.md)
