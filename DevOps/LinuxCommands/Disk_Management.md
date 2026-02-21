# **Linux Disk Management Commands** 💿

**Complete guide for disk partitioning, filesystems, and storage management**

---

## **Table of Contents** 📑
1. [Disk Information & Overview](#1-disk-information--overview)
2. [Partitioning Tools](#2-partitioning-tools)
3. [Filesystem Management](#3-filesystem-management)
4. [Mounting & Unmounting](#4-mounting--unmounting)
5. [Logical Volume Manager (LVM)](#5-logical-volume-manager-lvm)
6. [RAID Configuration](#6-raid-configuration)
7. [Disk Performance](#7-disk-performance)
8. [Quota Management](#8-quota-management)
9. [DevOps Use Cases](#9-devops-use-cases)
10. [Troubleshooting](#10-troubleshooting)
11. [Best Practices](#11-best-practices)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. Disk Information & Overview** 📊

### **View Disks and Partitions**

```bash
# List block devices
lsblk
# Output:
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# sda      8:0    0  500G  0 disk 
# ├─sda1   8:1    0  512M  0 part /boot/efi
# ├─sda2   8:2    0  100G  0 part /
# └─sda3   8:3    0  399G  0 part /home

# Detailed block device info
lsblk -f  # Show filesystems
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT

# List disk devices
fdisk -l                          # All disks (requires sudo)
sudo fdisk -l /dev/sda           # Specific disk

# Disk by-path, by-uuid, by-id
ls -l /dev/disk/by-id/
ls -l /dev/disk/by-uuid/
ls -l /dev/disk/by-path/

# Partition information
cat /proc/partitions
parted -l                         # GNU parted list
```

### **Disk Usage**

```bash
# Filesystem disk space
df -h                             # Human-readable
df -H                             # SI units
df -i                             # Inode information
df -T                             # Show filesystem type
df -t ext4                        # Only ext4 filesystems
df /dev/sda1                      # Specific partition

# Directory usage
du -sh /var/log                   # Summary
du -sh /*                         # Each top-level directory
du -ah /var/log | sort -rh | head -20  # Top 20

# Disk I/O statistics
iostat                            # Basic I/O stats
iostat -x 1                       # Extended, 1-second interval
iostat -d 2 10                    # Device stats, 2s interval, 10 samples
```

### **Hardware Information**

```bash
# Disk hardware details
sudo hdparm -I /dev/sda           # Drive information
sudo hdparm -Tt /dev/sda          # Benchmark read speed

# SMART status (requires smartmontools)
sudo smartctl -a /dev/sda         # All SMART info
sudo smartctl -H /dev/sda         # Health status
sudo smartctl -t short /dev/sda   # Run short test
sudo smartctl -t long /dev/sda    # Run long test

# Disk controller info
lspci | grep -i storage
lspci | grep -i raid
```

---

## **2. Partitioning Tools** 🔧

### **fdisk - MBR Partitioning**

```bash
# Launch fdisk (interactive)
sudo fdisk /dev/sdb

# Common fdisk commands (interactive):
m    # Help
p    # Print partition table
n    # New partition
d    # Delete partition
t    # Change partition type
w    # Write changes (save)
q    # Quit without saving

# Example: Create new partition
sudo fdisk /dev/sdb
n    # New partition
p    # Primary
1    # Partition number
[Enter]  # First sector (default)
+10G     # Last sector (10GB)
w    # Write and exit

# List partition types
sudo fdisk /dev/sdb
t    # Change type
L    # List partition codes
# 83 = Linux
# 82 = Linux swap
# 8e = Linux LVM
# fd = Linux raid auto
```

### **parted - GPT Partitioning**

```bash
# Launch parted
sudo parted /dev/sdb

# Parted commands (interactive):
print                             # Show partition table
mklabel gpt                       # Create GPT partition table
mklabel msdos                     # Create MBR partition table

mkpart primary ext4 0% 50%        # New partition (0-50% of disk)
mkpart primary ext4 1MiB 10GB     # New partition (1MB to 10GB)
rm 1                              # Remove partition 1
resizepart 1 20GB                 # Resize partition 1 to 20GB

quit                              # Exit

# Non-interactive examples
sudo parted /dev/sdb print
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 0% 100%
sudo parted /dev/sdb set 1 boot on  # Set boot flag
```

### **gdisk - GPT fdisk**

```bash
# Launch gdisk (GPT version of fdisk)
sudo gdisk /dev/sdb

# Similar commands to fdisk
p    # Print partition table
n    # New partition
d    # Delete partition
t    # Change partition type
w    # Write changes
q    # Quit without saving

# Partition type codes
8300  # Linux filesystem
8200  # Linux swap
8e00  # Linux LVM
fd00  # Linux RAID
```

### **cfdisk - User-Friendly Partitioning**

```bash
# Launch cfdisk (curses-based, user-friendly)
sudo cfdisk /dev/sdb

# Navigate with arrow keys
# Options: New, Delete, Resize, Type, Write, Quit
```

---

## **3. Filesystem Management** 📁

### **Create Filesystems**

```bash
# ext4 filesystem (most common)
sudo mkfs.ext4 /dev/sdb1
sudo mkfs.ext4 -L mylabel /dev/sdb1    # With label
sudo mkfs.ext4 -m 1 /dev/sdb1          # 1% reserved (default 5%)

# ext3 filesystem
sudo mkfs.ext3 /dev/sdb1

# XFS filesystem (good for large files)
sudo mkfs.xfs /dev/sdb1
sudo mkfs.xfs -f /dev/sdb1             # Force (overwrite)
sudo mkfs.xfs -L mylabel /dev/sdb1     # With label

# Btrfs filesystem (modern, with snapshots)
sudo mkfs.btrfs /dev/sdb1
sudo mkfs.btrfs -L mylabel /dev/sdb1

# FAT32 filesystem
sudo mkfs.vfat /dev/sdb1
sudo mkfs.vfat -F 32 /dev/sdb1         # Explicit FAT32

# NTFS filesystem
sudo mkfs.ntfs /dev/sdb1
sudo mkfs.ntfs -f /dev/sdb1            # Fast format

# Swap filesystem
sudo mkswap /dev/sdb1
sudo swapon /dev/sdb1                  # Activate swap
sudo swapoff /dev/sdb1                 # Deactivate swap
```

### **Check and Repair Filesystems**

```bash
# Check filesystem (MUST be unmounted)
sudo fsck /dev/sdb1
sudo fsck -n /dev/sdb1                 # Read-only check
sudo fsck -y /dev/sdb1                 # Auto-yes to repairs
sudo fsck -t ext4 /dev/sdb1            # Specific filesystem type

# Ext4-specific check
sudo e2fsck /dev/sdb1
sudo e2fsck -f /dev/sdb1               # Force check
sudo e2fsck -p /dev/sdb1               # Auto repair
sudo e2fsck -n /dev/sdb1               # No changes (read-only)

# XFS check and repair
sudo xfs_repair /dev/sdb1
sudo xfs_repair -n /dev/sdb1           # No modify (check only)

# Check all filesystems in /etc/fstab
sudo fsck -A
```

### **Filesystem Tuning**

```bash
# View filesystem parameters
sudo tune2fs -l /dev/sdb1

# Set filesystem label
sudo tune2fs -L newlabel /dev/sdb1

# Set reserved blocks percentage
sudo tune2fs -m 1 /dev/sdb1            # 1% reserved

# Set maximum mount count
sudo tune2fs -c 30 /dev/sdb1           # Check every 30 mounts
sudo tune2fs -c -1 /dev/sdb1           # Disable mount count check

# Set check interval
sudo tune2fs -i 180d /dev/sdb1         # Check every 180 days
sudo tune2fs -i 0 /dev/sdb1            # Disable time-based check

# Add journal to ext2 (converts to ext3)
sudo tune2fs -j /dev/sdb1

# Set UUID
sudo tune2fs -U random /dev/sdb1
```

### **Filesystem Labels and UUIDs**

```bash
# View filesystem labels
lsblk -f
blkid

# Set label (different tools for different filesystems)
sudo e2label /dev/sdb1 newlabel        # ext2/ext3/ext4
sudo xfs_admin -L newlabel /dev/sdb1   # XFS
sudo fatlabel /dev/sdb1 NEWLABEL       # FAT
sudo ntfslabel /dev/sdb1 newlabel      # NTFS

# View UUID
sudo blkid /dev/sdb1
lsblk -f

# Generate new UUID
sudo tune2fs -U random /dev/sdb1       # ext4
sudo xfs_admin -U generate /dev/sdb1   # XFS
```

---

## **4. Mounting & Unmounting** 🔗

### **Mount Filesystems**

```bash
# Basic mount
sudo mount /dev/sdb1 /mnt

# Mount with specific filesystem type
sudo mount -t ext4 /dev/sdb1 /mnt
sudo mount -t ntfs /dev/sdb1 /mnt
sudo mount -t vfat /dev/sdb1 /mnt

# Mount with options
sudo mount -o ro /dev/sdb1 /mnt              # Read-only
sudo mount -o rw /dev/sdb1 /mnt              # Read-write
sudo mount -o noexec /dev/sdb1 /mnt          # No execution
sudo mount -o nosuid /dev/sdb1 /mnt          # No suid bits
sudo mount -o nodev /dev/sdb1 /mnt           # No device files
sudo mount -o user,rw /dev/sdb1 /mnt         # Allow users to mount

# Mount by UUID (recommended)
sudo mount UUID=abc123-def456 /mnt

# Mount by label
sudo mount LABEL=mydata /mnt

# Remount with different options
sudo mount -o remount,ro /mnt                # Remount read-only
sudo mount -o remount,rw /mnt                # Remount read-write
```

### **Unmount Filesystems**

```bash
# Basic unmount
sudo umount /mnt
sudo umount /dev/sdb1

# Force unmount
sudo umount -f /mnt

# Lazy unmount (detach now, cleanup later)
sudo umount -l /mnt

# Check what's using the mountpoint
lsof /mnt
fuser -m /mnt                                 # Show processes
fuser -km /mnt                                # Kill processes using it
```

### **/etc/fstab - Permanent Mounts**

```bash
# Edit fstab
sudo nano /etc/fstab

# Format:
# <device> <mountpoint> <filesystem> <options> <dump> <pass>

# Examples:
UUID=abc123-def456  /data  ext4  defaults  0  2
/dev/sdb1          /backup ext4  defaults  0  2
/dev/sdb2          none    swap  sw        0  0

# Mount by UUID (recommended)
UUID=abc123-def456  /mnt/data  ext4  defaults,noatime  0  2

# Mount with specific options
UUID=abc123-def456  /mnt/data  ext4  rw,noatime,nodiratime  0  2

# NFS mount
192.168.1.100:/share  /mnt/nfs  nfs  defaults  0  0

# CIFS/SMB mount
//server/share  /mnt/smb  cifs  username=user,password=pass  0  0

# Test fstab without rebooting
sudo mount -a                                 # Mount all in fstab
sudo findmnt --verify                         # Verify fstab

# fstab options:
# defaults     = rw,suid,dev,exec,auto,nouser,async
# noatime      = Don't update access times (performance)
# nodiratime   = Don't update directory access times
# ro           = Read-only
# rw           = Read-write
# user         = Allow users to mount
# noauto       = Don't mount at boot
# nofail       = Don't fail boot if device missing
```

---

## **5. Logical Volume Manager (LVM)** 📦

### **LVM Concepts**

```
Physical Volume (PV) → Physical Disk (/dev/sdb)
Volume Group (VG)    → Collection of PVs
Logical Volume (LV)  → Virtual partition (like /dev/vg0/lv_data)

Advantages:
✅ Resize volumes dynamically
✅ Snapshots for backups
✅ Span multiple disks
✅ Move data between disks
```

### **Create LVM**

```bash
# 1. Create Physical Volume
sudo pvcreate /dev/sdb
sudo pvcreate /dev/sdc /dev/sdd        # Multiple disks

# View physical volumes
sudo pvs
sudo pvdisplay

# 2. Create Volume Group
sudo vgcreate vg_data /dev/sdb
sudo vgcreate vg_data /dev/sdb /dev/sdc  # Multiple disks

# View volume groups
sudo vgs
sudo vgdisplay

# 3. Create Logical Volume
sudo lvcreate -L 10G -n lv_app vg_data     # 10GB
sudo lvcreate -L 50%VG -n lv_data vg_data  # 50% of VG
sudo lvcreate -l 100%FREE -n lv_logs vg_data  # All remaining space

# View logical volumes
sudo lvs
sudo lvdisplay

# 4. Create filesystem on LV
sudo mkfs.ext4 /dev/vg_data/lv_app

# 5. Mount
sudo mkdir /mnt/app
sudo mount /dev/vg_data/lv_app /mnt/app

# Add to /etc/fstab:
/dev/vg_data/lv_app  /mnt/app  ext4  defaults  0  2
```

### **Resize LVM**

```bash
# Extend volume group (add disk)
sudo vgextend vg_data /dev/sde

# Extend logical volume
sudo lvextend -L +10G /dev/vg_data/lv_app      # Add 10GB
sudo lvextend -L 50G /dev/vg_data/lv_app       # Resize to 50GB
sudo lvextend -l +100%FREE /dev/vg_data/lv_app # Use all free space

# Resize filesystem (ext4)
sudo resize2fs /dev/vg_data/lv_app

# Resize filesystem (XFS)
sudo xfs_growfs /mnt/app

# Reduce logical volume (ext4 only, risky!)
# MUST unmount first
sudo umount /mnt/app
sudo e2fsck -f /dev/vg_data/lv_app
sudo resize2fs /dev/vg_data/lv_app 20G
sudo lvreduce -L 20G /dev/vg_data/lv_app
sudo mount /dev/vg_data/lv_app /mnt/app
```

### **LVM Snapshots**

```bash
# Create snapshot
sudo lvcreate -L 5G -s -n lv_app_snap /dev/vg_data/lv_app

# Mount snapshot
sudo mkdir /mnt/snap
sudo mount /dev/vg_data/lv_app_snap /mnt/snap

# Restore from snapshot
sudo umount /mnt/app
sudo lvconvert --merge /dev/vg_data/lv_app_snap
# Reboot or reactivate

# Remove snapshot
sudo lvremove /dev/vg_data/lv_app_snap
```

---

## **6. RAID Configuration** 🛡️

### **RAID Levels**

```
RAID 0 (Striping):
- 2+ disks, no redundancy
- ✅ Speed, ❌ No fault tolerance
- Capacity: Sum of all disks

RAID 1 (Mirroring):
- 2+ disks, full redundancy
- ✅ Fault tolerance, ❌ 50% capacity loss
- Capacity: Size of smallest disk

RAID 5 (Striping with Parity):
- 3+ disks, single disk redundancy
- ✅ Balance of speed/redundancy
- Capacity: (N-1) × smallest disk

RAID 6 (Double Parity):
- 4+ disks, two disk redundancy
- ✅ High fault tolerance
- Capacity: (N-2) × smallest disk

RAID 10 (1+0):
- 4+ disks, mirroring + striping
- ✅ Speed + redundancy
- Capacity: 50% of total
```

### **Software RAID (mdadm)**

```bash
# Install mdadm
sudo apt install mdadm        # Debian/Ubuntu
sudo yum install mdadm        # RHEL/CentOS

# Create RAID 1
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# Create RAID 5
sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd

# Create RAID 10
sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde

# View RAID status
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# Add spare disk
sudo mdadm --add /dev/md0 /dev/sdf

# Remove failed disk
sudo mdadm --fail /dev/md0 /dev/sdb
sudo mdadm --remove /dev/md0 /dev/sdb

# Stop RAID
sudo mdadm --stop /dev/md0

# Assemble RAID
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc

# Save RAID configuration
sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf

# Create filesystem on RAID
sudo mkfs.ext4 /dev/md0
sudo mount /dev/md0 /mnt/raid
```

---

## **7. Disk Performance** ⚡

### **Benchmark Disk Speed**

```bash
# dd write test
sudo dd if=/dev/zero of=/tmp/test bs=1M count=1024 conv=fdatasync
# 1GB write test

# dd read test
sudo dd if=/tmp/test of=/dev/null bs=1M count=1024

# hdparm read test
sudo hdparm -Tt /dev/sda

# fio (flexible I/O tester)
sudo fio --name=random-write --ioengine=libaio --iodepth=32 --rw=randwrite \
  --bs=4k --direct=1 --size=1G --numjobs=1 --runtime=60 --group_reporting
```

### **Monitor Disk I/O**

```bash
# iostat - I/O statistics
iostat -x 1                        # Extended stats, 1s interval
iostat -d 2 10                     # Device stats, 2s, 10 samples

# iotop - Top for I/O
sudo iotop                         # Interactive
sudo iotop -o                      # Only processes with I/O

# dstat - Versatile stats
dstat -d                           # Disk stats
dstat -d --disk-util              # Disk utilization

# sar - System activity report
sar -d 1 10                        # Disk stats, 1s, 10 samples
```

---

## **8. Quota Management** 📊

### **Enable Quotas**

```bash
# Edit /etc/fstab, add usrquota,grpquota
/dev/sdb1  /home  ext4  defaults,usrquota,grpquota  0  2

# Remount
sudo mount -o remount /home

# Create quota files
sudo quotacheck -cum /home
sudo quotacheck -cgm /home

# Enable quotas
sudo quotaon /home
```

### **Set Quotas**

```bash
# Set user quota (interactive)
sudo edquota -u username

# Set quota limits directly
sudo setquota -u username 1000000 1200000 0 0 /home
# soft_block hard_block soft_inode hard_inode

# Set quota for group
sudo edquota -g groupname
sudo setquota -g groupname 5000000 6000000 0 0 /home

# Copy quota from one user to another
sudo edquota -p sourceuser -u newuser
```

### **Check Quotas**

```bash
# Check user quota
quota -u username
quota

# Check group quota
quota -g groupname

# Report quota usage
sudo repquota /home
sudo repquota -a                   # All filesystems
```

---

## **9. DevOps Use Cases** 🚀

### **Automated Disk Monitoring**

```bash
#!/bin/bash
# disk-monitor.sh - Alert on disk usage

THRESHOLD=80

df -H | grep -vE '^Filesystem|tmpfs|cdrom' | awk '{print $5 " " $1}' | while read output; do
    usage=$(echo $output | awk '{print $1}' | cut -d'%' -f1)
    partition=$(echo $output | awk '{print $2}')
    
    if [ $usage -ge $THRESHOLD ]; then
        echo "WARNING: $partition is ${usage}% full"
        # Send alert (email, Slack, etc.)
    fi
done
```

### **LVM Snapshot for Backup**

```bash
#!/bin/bash
# backup-with-snapshot.sh

LV_PATH="/dev/vg_data/lv_app"
SNAPSHOT="lv_app_snap"
MOUNT="/mnt/snap"
BACKUP="/backup/app-$(date +%Y%m%d).tar.gz"

# Create snapshot
sudo lvcreate -L 5G -s -n $SNAPSHOT $LV_PATH

# Mount snapshot
sudo mkdir -p $MOUNT
sudo mount /dev/vg_data/$SNAPSHOT $MOUNT

# Backup from snapshot (application can continue running)
sudo tar -czf $BACKUP -C $MOUNT .

# Cleanup
sudo umount $MOUNT
sudo lvremove -f /dev/vg_data/$SNAPSHOT

echo "Backup completed: $BACKUP"
```

### **Automatic Filesystem Expansion**

```bash
#!/bin/bash
# auto-expand-filesystem.sh

THRESHOLD=90
LV_PATH="/dev/vg_data/lv_app"
MOUNT_POINT="/mnt/app"

USAGE=$(df -h $MOUNT_POINT | tail -1 | awk '{print $5}' | cut -d'%' -f1)

if [ $USAGE -ge $THRESHOLD ]; then
    echo "Disk usage at ${USAGE}%, expanding..."
    
    # Extend by 10GB
    sudo lvextend -L +10G $LV_PATH
    sudo resize2fs $LV_PATH
    
    echo "Filesystem expanded"
fi
```

---

## **10. Troubleshooting** 🔧

### **Disk Full Issues**

```bash
# Find what's using space
df -h
du -sh /* | sort -rh | head -10
du -ah /var | sort -rh | head -20

# Find large files
find / -type f -size +1G 2>/dev/null
find /var -type f -size +100M -exec ls -lh {} \;

# Check for deleted but open files
lsof | grep deleted
lsof +L1

# Clear common space hogs
sudo journalctl --vacuum-time=7d      # Systemd journal
sudo apt clean                        # Package cache
sudo docker system prune -a           # Docker
```

### **Cannot Unmount - Device Busy**

```bash
# Find processes using mountpoint
lsof /mnt
fuser -m /mnt

# Kill processes
fuser -km /mnt

# Lazy unmount
sudo umount -l /mnt
```

### **Filesystem Corruption**

```bash
# Boot from live CD/USB
# Run fsck (filesystem MUST be unmounted)
sudo fsck -y /dev/sda1

# Force check on next boot
sudo touch /forcefsck

# Ext4 superblock corruption
sudo e2fsck -b 32768 /dev/sda1       # Use backup superblock
```

---

## **11. Best Practices** ⭐

### **Do's**

```
✅ Use UUIDs in /etc/fstab (not device names)
✅ Label filesystems for easy identification
✅ Regular filesystem checks (tune2fs -c)
✅ Monitor disk health (SMART)
✅ Use LVM for flexibility
✅ Regular backups before disk operations
✅ Test fstab with 'mount -a' before reboot
✅ Use noatime mount option for performance
✅ Document disk configuration
✅ Monitor disk usage proactively
```

### **Don'ts**

```
❌ Don't resize mounted filesystems (ext4)
❌ Don't run fsck on mounted filesystem
❌ Don't ignore SMART warnings
❌ Don't fill disk to 100%
❌ Don't use RAID as backup replacement
❌ Don't forget about reserved blocks
❌ Don't mix partition tools (fdisk+parted)
❌ Don't resize without backup
```

---

## **12. Interview Cheat Sheet** 🎯

### **Q1: Difference between partition and logical volume?**
```
Partition:
- Fixed size on physical disk
- Difficult to resize
- Created with fdisk/parted
- Direct mapping to disk space

Logical Volume (LVM):
- Virtual partition
- Easy to resize
- Can span multiple disks
- Supports snapshots
- More flexible

Use partitions for simplicity.
Use LVM for flexibility/production.
```

### **Q2: How to add new disk and mount permanently?**
```
Steps:
1. Identify disk: lsblk
2. Partition: sudo fdisk /dev/sdb
3. Create filesystem: sudo mkfs.ext4 /dev/sdb1
4. Create mountpoint: sudo mkdir /mnt/data
5. Get UUID: sudo blkid /dev/sdb1
6. Edit fstab: 
   UUID=abc123  /mnt/data  ext4  defaults  0  2
7. Test: sudo mount -a
8. Verify: df -h

or with LVM:
1. pvcreate /dev/sdb
2. vgcreate vg_data /dev/sdb
3. lvcreate -l 100%FREE -n lv_data vg_data
4. mkfs.ext4 /dev/vg_data/lv_data
5. Add to fstab
```

### **Q3: Explain RAID levels?**
```
RAID 0: Striping (speed, no redundancy)
RAID 1: Mirroring (redundancy, 50% capacity)
RAID 5: Striping + Parity (balance, N-1 capacity)
RAID 6: Double parity (2-disk failure, N-2 capacity)
RAID 10: Mirror + Stripe (best performance, 50% capacity)

DevOps choice:
- OS: RAID 1
- Database: RAID 10
- Data: RAID 5/6
- Backup: RAID 5

Hardware RAID > Software RAID (performance)
```

### **Q4: How to expand filesystem when disk full?**
```
With LVM (easy):
1. sudo lvextend -L +10G /dev/vg/lv
2. sudo resize2fs /dev/vg/lv       # ext4
   sudo xfs_growfs /mount           # xfs

Without LVM (harder):
1. Backup data
2. Add new disk
3. Create new partition
4. Move data
5. Update fstab

Prevention:
- Monitor disk usage (df -h)
- Set up alerts (>80%)
- Use LVM for flexibility
- Regular cleanup
```

### **Q5: Common disk management commands?**
```
Information:
lsblk                    # List block devices
df -h                    # Disk space
du -sh dir               # Directory size
fdisk -l                 # List partitions
blkid                    # UUIDs and labels

Partitioning:
fdisk /dev/sdb           # MBR partition
parted /dev/sdb          # GPT partition
mkfs.ext4 /dev/sdb1      # Create filesystem

Mounting:
mount /dev/sdb1 /mnt     # Mount 
umount /mnt              # Unmount
mount -a                 # Mount all (fstab)

LVM:
pvcreate /dev/sdb        # Physical volume
vgcreate vg /dev/sdb     # Volume group
lvcreate -L 10G -n lv vg # Logical volume
lvextend -L +5G /dev/vg/lv  # Extend

Monitoring:
iostat -x 1              # I/O statistics
iotop                    # I/O by process
```

---

**💿 Master Disk Management for Reliable Storage Operations!**

*Understanding disk management is critical for maintaining robust server infrastructure.*
