# DISK MANAGEMENT IN LINUX — Complete Exam Guide

**Main tools:** `fdisk` · `parted` · `mkfs` · `mount` · `lsblk` · `df` · `du`

---

## 📋 TABLE OF CONTENTS

1. [View disk information](#part-1)
2. [Add a second disk (VirtualBox/AWS)](#part-2)
3. [Partition with fdisk](#part-3)
4. [Format partitions](#part-4)
5. [Mount partitions](#part-5)
6. [Permanent mount (/etc/fstab)](#part-6)
7. [Resize partitions](#part-7)
8. [Change partition type](#part-8)
9. [Delete partitions](#part-9)
10. [SWAP partitions](#part-10)
11. [Permissions and owners](#part-11)
12. [Troubleshooting](#part-12)
13. [Quick cheatsheet](#cheatsheet)
14. [Typical exam scenarios](#scenarios)

---

<a name="part-1"></a>
## 📊 PART 1: View disk information

### lsblk — view all disks and partitions

```bash
lsblk
```

Typical output:
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk
├─sda1   8:1    0   19G  0 part /
├─sda2   8:2    0    1K  0 part
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0   10G  0 disk
```

- `sda` = First disk (main) · `sdb` = Second disk (unpartitioned)
- `sda1` = First partition of sda · `MOUNTPOINT` = Where it is mounted

View with filesystem information:
```bash
lsblk -f
```

### fdisk -l — detailed partition listing

```bash
sudo fdisk -l
sudo fdisk -l /dev/sda
sudo fdisk -l /dev/sdb
```

### df — used and available space on mounted partitions

→ `-h` = Human readable · `-T` = show filesystem type
```bash
df -h
df -hT
```

### du — space used by folders

```bash
du -sh /srv/samba          # Total of the folder
du -h /srv/samba           # Breakdown by subfolders
du -h /srv | sort -h       # Sorted by size
```

### blkid — view UUIDs of all partitions

```bash
sudo blkid
```

Output:
```
/dev/sda1: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" TYPE="ext4"
/dev/sda5: UUID="12345678-90ab-cdef-1234-567890abcdef" TYPE="swap"
/dev/sdb1: UUID="11111111-2222-3333-4444-555555555555" TYPE="ext4"
```

---

<a name="part-2"></a>
## 💾 PART 2: Add a second disk

### Option A: VirtualBox (VM powered off)

1. Right-click on the VM → Settings → Storage
2. SATA Controller → "+" icon → Create → VDI → Dynamically allocated
3. Size: 10 GB · Name: `second_disk.vdi` → Create → Accept

→ `/dev/sdb` should appear (new 10 GB disk)
```bash
lsblk
```

### Option B: AWS EC2 (running instance)

1. EC2 → Volumes → Create Volume → Size: 10 GiB
2. Availability Zone: **the same as your instance** → Create Volume
3. Select volume → Actions → Attach Volume → select instance
4. Device name: `/dev/sdf` (AWS changes it to `/dev/xvdf` automatically)

→ `/dev/xvdf` or `/dev/nvme1n1` should appear (modern instances)
```bash
lsblk
```

---

<a name="part-3"></a>
## 🔧 PART 3: Partition with fdisk

### Reference of commands inside fdisk

| Key | Action |
|---|---|
| `n` | New partition |
| `d` | Delete partition |
| `t` | Change type |
| `p` | View current partition table |
| `l` | List available types |
| `w` | Save and exit |
| `q` | Exit WITHOUT saving |

> `p` = Primary (max. 4) · In sectors, **Enter = default value**

---

### Scenario 1: One partition occupying the entire disk

→ `/dev/sdb1` should appear
```bash
sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → [Enter] → w

lsblk
```

### Scenario 2: Two partitions (5 GB each)

First partition:
```bash
sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → +5G → w

lsblk
```

Second partition (rest of the space):
```bash
sudo fdisk /dev/sdb
# n → p → 2 → [Enter] → [Enter] → w

lsblk
# → /dev/sdb1 (5G) and /dev/sdb2 (5G)
```

### Scenario 3: Partition of a specific size

Size reference: `+2G` · `+500M` · `+1T` · `+2048M`

Create an exact 3 GB partition:
```bash
sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → +3G → w
```

---

<a name="part-4"></a>
## 💿 PART 4: Format partitions

### ext4 (most common in Linux)
```bash
sudo mkfs.ext4 /dev/sdb1
```

### xfs
```bash
sudo mkfs.xfs /dev/sdb1
```

### vfat — FAT32, Windows compatible
```bash
sudo mkfs.vfat /dev/sdb1
```

### ntfs — Windows compatible
```bash
sudo apt install -y ntfs-3g
sudo mkfs.ntfs /dev/sdb1
```

### With label
→ `lsblk -f` should show `LABEL: DATA`
```bash
sudo mkfs.ext4 -L DATA /dev/sdb1
lsblk -f
```

---

<a name="part-5"></a>
## 🗂️ PART 5: Mount partitions

### Temporary manual mount
→ `df -h` should show `/dev/sdb1` mounted at `/mnt/data`
```bash
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data
df -h | grep data
ls -la /mnt/data
```

### Mount with specific options
```bash
sudo mount -o rw,uid=1000,gid=1000 /dev/sdb1 /mnt/data
```

Options: `rw` = read/write · `ro` = read only · `uid=1000` = owner · `gid=1000` = group

### Unmount
```bash
sudo umount /mnt/data
# Or by device:
sudo umount /dev/sdb1
```

🛠 If it says "target is busy":
```bash
sudo lsof /mnt/data           # See which process is using it
sudo kill -9 [PID]
sudo umount -l /mnt/data      # Force (with caution)
```

---

<a name="part-6"></a>
## 🔄 PART 6: Permanent mount (/etc/fstab)

### Format of an fstab entry
```
<device>  <mount_point>  <filesystem>  <options>  <dump>  <fsck>
```
- `UUID=...` → unique identifier · `defaults` → standard options (rw, suid, dev, exec, auto, nouser, async)
- `dump`: 0 = no backup · `fsck`: 0 = don't check, 1 = first, 2 = after

### Add automatic mount at boot

Get UUID, edit fstab, test and reboot:
→ `sudo mount -a` without errors = correct. After reboot `df -h | grep data` should show the disk mounted.
```bash
sudo blkid /dev/sdb1
# Copy the UUID

sudo nano /etc/fstab
# Add at the end:
# UUID=11111111-2222-3333-4444-555555555555  /mnt/data  ext4  defaults  0  2

sudo mount -a
df -h | grep data
sudo reboot
# After reboot:
df -h | grep data
```

### Advanced mount options in fstab

```
# Read only
UUID=...  /mnt/data  ext4  ro  0  2

# With specific permissions
UUID=...  /mnt/data  ext4  defaults,uid=1000,gid=1000  0  2

# Without binary execution (security)
UUID=...  /mnt/data  ext4  defaults,noexec  0  2

# Do not mount automatically
UUID=...  /mnt/data  ext4  noauto  0  0
```

---

<a name="part-7"></a>
## 📏 PART 7: Resize partitions

### Method 1: Delete and recreate — LOSES DATA

> ⚠️ This DELETES all data from the partition.

Scenario: change `/dev/sdb1` from 5 GB to 8 GB.
```bash
sudo umount /dev/sdb1
sudo fdisk /dev/sdb
# d → 1
# n → p → 1 → [Enter] → +8G → w

sudo mkfs.ext4 /dev/sdb1
sudo mount /dev/sdb1 /mnt/data
```

### Method 2: Resize without losing data — ext4 only

> ⚠️ The partition must be **unmounted**.

> ⚠️ When recreating the partition with fdisk, make sure to start at the **same initial sector**. When fdisk asks whether to remove the ext4 signature, answer **N**.

```bash
sudo umount /dev/sdb1
sudo fdisk /dev/sdb
# d → 1
# n → p → 1 → [Enter] → [Enter]
# N (do NOT remove ext4 signature)
# w

sudo e2fsck -f /dev/sdb1
sudo resize2fs /dev/sdb1
sudo mount /dev/sdb1 /mnt/data
df -h /mnt/data
```

---

<a name="part-8"></a>
## 🔄 PART 8: Change partition type

### Most used types

| Code | Type |
|---|---|
| `83` | Linux (default) |
| `82` | Linux swap |
| `8e` | Linux LVM |

View full list inside fdisk: key `l`

### Change type — example to Linux LVM (8e)

→ `sudo fdisk -l /dev/sdb` should show `Type: Linux LVM`
```bash
sudo fdisk /dev/sdb
# t → 1 → 8e → w

sudo fdisk -l /dev/sdb
```

---

<a name="part-9"></a>
## 🗑️ PART 9: Delete partitions

### Delete a partition
```bash
sudo fdisk /dev/sdb
# d → 1 → w
```

### Delete all partitions
```bash
sudo fdisk /dev/sdb
# d → 1 → d → 2 → d → 3 → ... → w
```

### Completely wipe a disk

> ⚠️ This PERMANENTLY DELETES everything on the disk. Ctrl+C to cancel.

```bash
# Entire disk:
sudo dd if=/dev/zero of=/dev/sdb bs=1M status=progress

# Only the first 10 MB (partition table):
sudo dd if=/dev/zero of=/dev/sdb bs=1M count=10
```

---

<a name="part-10"></a>
## 🔄 PART 10: SWAP partitions

### Create, activate and make SWAP permanent

```bash
# 1. Create swap partition with fdisk
sudo fdisk /dev/sdb
# n → p → 2 → [Enter] → +2G → w

# 2. Change type to swap (82)
sudo fdisk /dev/sdb
# t → 2 → 82 → w

# 3. Format, activate and verify
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
swapon --show
free -h

# 4. Make permanent in fstab
sudo blkid /dev/sdb2
sudo nano /etc/fstab
# Add: UUID=... none swap sw 0 0
```

### Deactivate swap
```bash
sudo swapoff /dev/sdb2
```

---

<a name="part-11"></a>
## 🔐 PART 11: Permissions and owners

### Change owner and permissions of the mounted disk

→ `ls -ld /mnt/data` should show the correct owner and permissions
```bash
sudo chown ubuntu:ubuntu /mnt/data         # Change owner
sudo chown -R ubuntu:ubuntu /mnt/data      # Recursive
sudo chmod 755 /mnt/data                   # rwxr-xr-x
sudo chmod 777 /mnt/data                   # All permissions
sudo chmod 700 /mnt/data                   # Owner only
ls -ld /mnt/data
```

### Reading the ls -ld output

```
drwxr-xr-x 3 ubuntu ubuntu 4096 Feb 22 10:00 /mnt/data
│└──┘└──┘└──┘
│  │   │   └── Others: r-x
│  │   └────── Group: r-x
│  └────────── Owner: rwx
└───────────── d = directory
```

---

<a name="part-12"></a>
## 🛠️ PART 12: Troubleshooting

### Error: "No space left on device"
```bash
df -h                                  # View disk space
df -i                                  # View inodes (may be full even if there's space)
sudo du -h / | sort -h | tail -20      # Find large files
```

### Error: "mount: wrong fs type, bad option, bad superblock"
Causes: unformatted filesystem or wrong type in fstab.
```bash
sudo blkid /dev/sdb1          # See actual type
sudo mkfs.ext4 /dev/sdb1      # Format if empty
cat /etc/fstab                # Verify fstab
```

### Error: "target is busy" when unmounting
```bash
sudo lsof /mnt/data           # See which process is using it
sudo kill -9 [PID]
sudo umount -l /mnt/data      # Force if necessary
```

### Error in /etc/fstab that prevents booting
1. In GRUB → "Advanced options" → "recovery mode" → "root"
```bash
nano /etc/fstab
# Comment out the problematic line with #
reboot
```

### Disk doesn't appear in lsblk — VirtualBox
Check in VirtualBox → Settings → Storage that the disk appears.
```bash
echo "- - -" | sudo tee /sys/class/scsi_host/host*/scan
ls -la /dev/sd*
```

### Disk doesn't appear in lsblk — AWS
Check in EC2 → Volumes that the status is "in-use".
```bash
ls -la /dev/nvme*
ls -la /dev/xvd*
```

---

<a name="cheatsheet"></a>
## 📝 QUICK CHEATSHEET

```bash
# VIEW DISKS
lsblk                                      # All disks and partitions
lsblk -f                                   # With filesystems and UUIDs
sudo fdisk -l                              # Detailed listing
df -h                                      # Used/available space
df -i                                      # Inodes
du -sh /folder                             # Folder size
sudo blkid                                 # UUIDs of all partitions

# PARTITION (inside fdisk)
sudo fdisk /dev/sdb
# n = new | d = delete | t = change type | p = view | w = save | q = exit without saving

# FORMAT
sudo mkfs.ext4 /dev/sdb1                   # ext4
sudo mkfs.ext4 -L DATA /dev/sdb1           # ext4 with label
sudo mkfs.xfs /dev/sdb1                    # xfs
sudo mkfs.vfat /dev/sdb1                   # FAT32
sudo mkswap /dev/sdb2                      # swap

# MOUNT / UNMOUNT
sudo mkdir /mnt/data
sudo mount /dev/sdb1 /mnt/data
sudo umount /mnt/data
sudo mount -a                              # Mount everything from fstab

# SWAP
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
sudo swapoff /dev/sdb2
swapon --show

# RESIZE (ext4, no data loss)
sudo e2fsck -f /dev/sdb1
sudo resize2fs /dev/sdb1

# PERMISSIONS
sudo chown ubuntu:ubuntu /mnt/data
sudo chown -R ubuntu:ubuntu /mnt/data
sudo chmod 755 /mnt/data
```

---

<a name="scenarios"></a>
## 🎯 TYPICAL EXAM SCENARIOS

### Scenario 1: Add a 10 GB data disk

```bash
lsblk

sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → [Enter] → w

sudo mkfs.ext4 -L DATA /dev/sdb1
sudo mkdir /mnt/data
sudo mount /dev/sdb1 /mnt/data

sudo blkid /dev/sdb1
# Copy UUID
sudo nano /etc/fstab
# Add: UUID=... /mnt/data ext4 defaults 0 2

sudo mount -a
df -h | grep data
```

### Scenario 2: Create a 2 GB SWAP partition

```bash
sudo fdisk /dev/sdb
# n → p → 2 → [Enter] → +2G → w

sudo fdisk /dev/sdb
# t → 2 → 82 → w

sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
swapon --show

sudo blkid /dev/sdb2
# Copy UUID
sudo nano /etc/fstab
# Add: UUID=... none swap sw 0 0
```

### Scenario 3: Split disk into 2 equal partitions (10 GB disk → 5 GB + 5 GB)

```bash
sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → +5G → w

sudo fdisk /dev/sdb
# n → p → 2 → [Enter] → [Enter] → w

sudo mkfs.ext4 /dev/sdb1
sudo mkfs.ext4 /dev/sdb2
sudo mkdir /mnt/data1 /mnt/data2
sudo mount /dev/sdb1 /mnt/data1
sudo mount /dev/sdb2 /mnt/data2
df -h | grep data
```

---

## 🎯 END OF GUIDE

- ✅ View disk information — lsblk, fdisk, df, du, blkid
- ✅ Add disks in VirtualBox and AWS
- ✅ Partition with fdisk — one, several, specific size
- ✅ Format — ext4, xfs, vfat, ntfs, swap
- ✅ Manual and automatic mount · complete /etc/fstab
- ✅ Resize partitions with and without data loss
- ✅ Change partition types · Delete partitions
- ✅ Create and manage SWAP · Permissions and owners
- ✅ Complete troubleshooting

> For the exam: practice the 3 scenarios · memorize the cheatsheet · always verify with `lsblk` after each step · use `sudo mount -a` to test fstab before rebooting.
