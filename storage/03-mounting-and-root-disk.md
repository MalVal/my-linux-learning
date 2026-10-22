# Mounting a New Disk and Understanding the Root Disk

How to attach a new disk permanently, and how the root (`/`) disk works.

**Series:**
1. [Storage fundamentals](01-linux-storage-fundamentals.md)
2. [Partitioning and formatting](02-partitioning-and-formatting.md)
3. **Mounting a new disk and the root disk** (this file)

---

## Part A: Mounting a new disk

### Prerequisites

You have a disk with a partition and a filesystem (see tutorial 2):

```bash
lsblk -f
```

```
NAME   FSTYPE LABEL  UUID                                 MOUNTPOINTS
sdb
└─sdb1 ext4   mydata 3a1f6e1c-8d2b-4b0e-9c5a-1f2e3d4c5b6a
```

`MOUNTPOINTS` is empty, so it is not accessible yet.

---

### Step 1: Create the mount point

A mount point is just an empty folder.

```bash
sudo mkdir -p /mnt/data
```

Convention:
- `/mnt/...` → manual / permanent mounts you choose
- `/media/...` → removable media (usually auto-mounted by the desktop)
- Custom paths like `/data` or `/srv/storage` are fine too.

### Step 2: Mount it (temporary)

```bash
sudo mount /dev/sdb1 /mnt/data
```

Check:

```bash
df -h /mnt/data
lsblk
```

This mount **disappears at reboot**. Test it:

```bash
ls /mnt/data
```

### Step 3: Fix permissions

A freshly formatted ext4 filesystem is owned by `root`. Give yourself access:

```bash
sudo chown -R $USER:$USER /mnt/data
```

Or give a specific user/group:

```bash
sudo chown -R alice:alice /mnt/data
sudo chmod 755 /mnt/data
```

> This applies to ext4/xfs/btrfs (Linux filesystems with permissions). For NTFS/exFAT/FAT32, ownership is set with **mount options** (`uid=`, `gid=`), see below.

### Step 4: Make it permanent with `/etc/fstab`

`/etc/fstab` ("filesystem table") lists everything mounted at boot.

#### 4.1 Get the UUID

```bash
sudo blkid /dev/sdb1
```

```
/dev/sdb1: LABEL="mydata" UUID="3a1f6e1c-8d2b-4b0e-9c5a-1f2e3d4c5b6a" TYPE="ext4"
```

**Why UUID and not `/dev/sdb1`?** Device names can change between boots (plug another disk, change a port...). The UUID is tied to the filesystem and never changes.

#### 4.2 Back up fstab

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

#### 4.3 Edit it

```bash
sudo nano /etc/fstab
```

Add a line at the end:

```
UUID=3a1f6e1c-8d2b-4b0e-9c5a-1f2e3d4c5b6a  /mnt/data  ext4  defaults,nofail  0  2
```

#### 4.4 Test **before** rebooting

```bash
sudo umount /mnt/data          # unmount the manual mount
sudo mount -a                  # mount everything in fstab
df -h /mnt/data                # should be mounted
```

If `mount -a` prints **no error** and the disk is mounted, you're safe. A broken fstab can prevent the system from booting normally, so never skip this test.

---

### Understanding the fstab line

```
UUID=xxxx   /mnt/data   ext4   defaults,nofail   0   2
│           │           │      │                  │   │
│           │           │      │                  │   └─ 6. fsck order
│           │           │      │                  └───── 5. dump (always 0)
│           │           │      └──────────────────────── 4. mount options
│           │           └─────────────────────────────── 3. filesystem type
│           └─────────────────────────────────────────── 2. mount point
└─────────────────────────────────────────────────────── 1. device (UUID/LABEL/path)
```

| Field | Meaning |
|-------|---------|
| 1 | Device: `UUID=...`, `LABEL=...`, or `/dev/...` |
| 2 | Mount point (must exist) |
| 3 | Filesystem type: `ext4`, `xfs`, `btrfs`, `vfat`, `ntfs3`, `exfat`, `swap`... |
| 4 | Options, comma-separated |
| 5 | `dump` backup flag, set `0` |
| 6 | fsck order: `1` for `/`, `2` for other disks, `0` = never check (use `0` for xfs/btrfs) |

### Common mount options

| Option | Effect |
|--------|--------|
| `defaults` | Sensible set: `rw,suid,dev,exec,auto,nouser,async` |
| `nofail` | **Don't fail the boot** if the disk is missing (great for external/secondary disks) |
| `noatime` | Don't update access times, less writing, better for SSDs |
| `ro` | Read-only |
| `noexec` | Forbid running programs from this disk |
| `user` | Allow normal users to mount it |
| `x-systemd.automount` | Mount on first access (lazy) |
| `discard` | Continuous TRIM for SSDs (periodic `fstrim.timer` is usually preferred) |
| `uid=1000,gid=1000` | For NTFS/exFAT/FAT: set owner (find your id with `id`) |

### Example fstab lines

```
# ext4 data disk
UUID=3a1f6e1c-...  /mnt/data   ext4   defaults,nofail,noatime  0  2

# xfs disk
UUID=9b8c7d6e-...  /srv/files  xfs    defaults,nofail          0  0

# NTFS disk shared with Windows (owned by user 1000)
UUID=1A2B3C4D5E6F  /mnt/windows  ntfs3  defaults,nofail,uid=1000,gid=1000,umask=022  0  0

# exFAT USB drive
UUID=ABCD-1234  /mnt/usb  exfat  defaults,nofail,uid=1000,gid=1000  0  0

# Network share (NFS)
192.168.1.10:/share  /mnt/nas  nfs  defaults,_netdev,nofail  0  0
```

---

### Unmounting

```bash
sudo umount /mnt/data
# or
sudo umount /dev/sdb1
```

**"target is busy"?** Something is using it. Find out what:

```bash
sudo lsof +f -- /mnt/data
sudo fuser -vm /mnt/data
```

Also make sure your shell isn't inside that folder (`cd ~`).

For a removable drive, flush and power off safely:

```bash
sync
sudo umount /dev/sdb1
sudo udisksctl power-off -b /dev/sdb
```

---

### Complete example in one block

```bash
# 1. identify
lsblk -o NAME,SIZE,MODEL,MOUNTPOINTS

# 2. partition (whole disk, GPT)
sudo parted /dev/sdb --script mklabel gpt mkpart data ext4 0% 100%

# 3. format
sudo mkfs.ext4 -L mydata /dev/sdb1

# 4. mount point + mount
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data
sudo chown -R $USER:$USER /mnt/data

# 5. permanent
UUID=$(sudo blkid -s UUID -o value /dev/sdb1)
echo "UUID=$UUID  /mnt/data  ext4  defaults,nofail  0  2" | sudo tee -a /etc/fstab
sudo umount /mnt/data && sudo mount -a && df -h /mnt/data
```

---

### Using a new disk for `/home`

Moving `/home` to its own disk:

```bash
sudo mkdir /mnt/newhome
sudo mount /dev/sdb1 /mnt/newhome
sudo rsync -aXS /home/. /mnt/newhome/.
```

Then in fstab:

```
UUID=<uuid>  /home  ext4  defaults  0  2
```

Reboot (or log out all users), verify with `df -h /home`, then delete the old data hidden underneath after confirming everything works:

```bash
sudo umount /home
# the old /home content is now visible, remove it once you are sure
sudo mount /home
```

---

### Troubleshooting

| Problem | Likely cause / fix |
|---------|--------------------|
| `wrong fs type, bad option, bad superblock` | No filesystem, or wrong type. Check `lsblk -f`, install needed tools (`ntfs-3g`, `exfatprogs`) |
| `mount point does not exist` | `sudo mkdir -p /mnt/data` |
| `Permission denied` when writing | Run `chown` on the mount point (ext4/xfs) or set `uid/gid` (NTFS/FAT) |
| `target is busy` on umount | `lsof`, `fuser`, or `cd` out of the folder |
| System drops to **emergency mode** after reboot | Error in `/etc/fstab`. Fix it from the emergency shell (`nano /etc/fstab`), or restore `/etc/fstab.bak`. Use `nofail` to prevent this for non-essential disks |
| Disk name changed (sdb → sdc) | You used `/dev/sdX` in fstab. Switch to `UUID=` |
| NTFS read-only | Windows "fast startup"/hibernation left it dirty. Fully shut down Windows or run `sudo ntfsfix /dev/sdX1` |
| Disk full but `du` shows little | Deleted files still held by a process: `sudo lsof +L1`, restart the process |

---

## Part B: The root disk (`/`)

### What "root" means: don't confuse these

| Term | Meaning |
|------|---------|
| `/` | **Root directory**: top of the whole tree |
| **Root filesystem / root disk / root partition** | The partition mounted on `/`. It contains the OS |
| `/root` | **Home folder** of the administrator account |
| **root user** | The superuser account (UID 0) |

### Find your root disk

```bash
findmnt /
df -h /
lsblk
```

Example:

```
TARGET SOURCE         FSTYPE OPTIONS
/      /dev/nvme0n1p2 ext4   rw,relatime
```

Here the root disk is partition **`/dev/nvme0n1p2`** of the NVMe drive.

### How it boots (simplified)

```
1. Firmware (UEFI/BIOS) starts
2. It reads the ESP (EFI) partition → loads the bootloader (GRUB / systemd-boot)
3. Bootloader loads the kernel + initramfs from /boot
4. Kernel mounts the root filesystem on "/" (told by root=UUID=... on the kernel command line)
5. systemd starts, reads /etc/fstab, mounts all other disks
```

Check what root the kernel was told to use:

```bash
cat /proc/cmdline
```

```
BOOT_IMAGE=/boot/vmlinuz-6.x root=UUID=5c2e... ro quiet splash
```

### Why the root partition is special

- It **must** be mounted first; everything else depends on it.
- It holds `/etc`, so `/etc/fstab` itself lives on it.
- If it's full, the system can misbehave (logs fail, apps crash, you can't log in).
- It cannot be unmounted while the system is running.

### Typical root layouts

**Simple (most desktops/VMs):**

```
/dev/sda1  ESP    512M   /boot/efi
/dev/sda2  root   rest   /
```

**Separate home (easier reinstalls):**

```
/dev/sda1  ESP    512M   /boot/efi
/dev/sda2  root   50G    /
/dev/sda3  home   rest   /home
```

**Server style:**

```
/boot/efi   512M
/boot       1G
/           30G
/var        20G+   (logs, databases, containers)
/home       rest
swap        2-8G
```

### Root disk full? Diagnose

```bash
df -h /
sudo du -h --max-depth=1 / 2>/dev/null | sort -h | tail -15
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h | tail
```

Common culprits and cleanup:

```bash
sudo journalctl --vacuum-size=200M     # shrink systemd logs
sudo apt clean                          # Debian/Ubuntu package cache
sudo dnf clean all                      # Fedora/RHEL
docker system prune                     # Docker leftovers (careful!)
```

### Growing the root disk (VM / cloud)

Common scenario: you enlarged the virtual disk in your hypervisor or cloud console. The OS still sees the old partition size.

**1. Check the disk size is updated**

```bash
lsblk
```

**2. Grow the partition** (example: partition 2 on `/dev/vda`)

```bash
sudo apt install cloud-guest-utils      # provides growpart
sudo growpart /dev/vda 2
```

**3. Grow the filesystem**

```bash
sudo resize2fs /dev/vda2                # ext4
sudo xfs_growfs /                       # xfs (uses the mount point)
sudo btrfs filesystem resize max /      # btrfs
```

**If the root is on LVM:**

```bash
sudo growpart /dev/vda 2
sudo pvresize /dev/vda2
sudo lvextend -r -l +100%FREE /dev/mapper/vg0-root
```

`-r` resizes the filesystem automatically. This all works **online**, without a reboot.

> ⚠️ **Shrinking** is much riskier: xfs can't shrink at all, and ext4 must be unmounted (so not possible for a running root). Back up first and prefer rebuilding.

### Moving a folder to a bigger disk (when root is too small)

Example: move `/var/lib/docker` to a new disk mounted on `/mnt/data`.

```bash
sudo systemctl stop docker
sudo mkdir -p /mnt/data/docker
sudo rsync -aXS /var/lib/docker/ /mnt/data/docker/
sudo mv /var/lib/docker /var/lib/docker.old
sudo mkdir /var/lib/docker
sudo mount --bind /mnt/data/docker /var/lib/docker
sudo systemctl start docker
```

Make the bind mount permanent in fstab:

```
/mnt/data/docker  /var/lib/docker  none  bind  0  0
```

Delete `/var/lib/docker.old` once everything is confirmed working. (Many services also let you change the data path in their config, which is cleaner than a bind mount.)

### Don't do this

- ❌ `rm -rf /` or `rm -rf /*` (deletes the system)
- ❌ Format or repartition the disk containing `/` while the system is running
- ❌ Edit `/etc/fstab` without backup and without `mount -a` test
- ❌ Use `chmod -R 777 /` or `chown -R` on system folders
- ❌ Fill `/` with data. Put big data on its own disk/partition.

---

## Final cheat sheet

```bash
# See things
lsblk -f                      # disks, filesystems, UUIDs, mounts
df -h                         # free space
findmnt                       # mount tree
sudo blkid                    # UUIDs

# New disk workflow
sudo parted /dev/sdb --script mklabel gpt mkpart data ext4 0% 100%
sudo mkfs.ext4 -L mydata /dev/sdb1
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data

# Permanent
sudo cp /etc/fstab /etc/fstab.bak
# add: UUID=<uuid>  /mnt/data  ext4  defaults,nofail  0  2
sudo mount -a                 # test!

# Unmount
sudo umount /mnt/data

# Grow root (ext4, partition 2 on vda)
sudo growpart /dev/vda 2 && sudo resize2fs /dev/vda2
```

🎉 You now know how Linux handles disks from the hardware up to your folders.

⬅️ Back to: [Storage fundamentals](01-linux-storage-fundamentals.md) · [Partitioning and formatting](02-partitioning-and-formatting.md)
