
---

# Linux LVM Setup & Root Filesystem Migration Guide

---

## Part 1: Basic LVM Setup on a Secondary Disk

### Objective

Convert `/dev/nvme1n1` (10 GB) into an LVM-backed filesystem and mount it.

### Step-by-Step Commands

```bash
# 1. Initialize Physical Volume
sudo pvcreate /dev/nvme1n1

# 2. Create Volume Group
sudo vgcreate data_vg /dev/nvme1n1

# 3. Create Logical Volume
sudo lvcreate -n data_lv -l 100%FREE data_vg

# 4. Format Filesystem
sudo mkfs.ext4 /dev/data_vg/data_lv

# 5. Mount Volume
sudo mkdir -p /mnt/data
sudo mount /dev/data_vg/data_lv /mnt/data
```

---

## Part 2: Root Filesystem Migration (High-Level Plan)

### Migration Phases

```text
Phase 1: Prepare   → Setup LVM on secondary disk
Phase 2: Target    → Create new root logical volume
Phase 3: Clone     → Copy current root filesystem
Phase 4: Configure → Update fstab, initramfs, GRUB
Phase 5: Switch    → Boot into LVM root
Phase 6: Merge     → Extend VG with original disk
```

---

## Part 3: Immediate Action — Clone Root to LVM

### 1. Prepare the New Root Volume

```bash
# Unmount temporary test volume
sudo umount /mnt/data

# Remove test logical volume
sudo lvremove /dev/data_vg/data_lv -y

# Create a new root logical volume (use entire disk)
sudo lvcreate -n root_lv -l 100%FREE data_vg

# Format it
sudo mkfs.ext4 /dev/data_vg/root_lv
```

---

### 2. Clone the Current System

```bash
# Mount new root volume
sudo mkdir -p /mnt/new_root
sudo mount /dev/data_vg/root_lv /mnt/new_root

# Clone system files
sudo rsync -axHAX --progress / /mnt/new_root/
```

---

### 3. Bind Critical System Directories

```bash
# Bind system directories
for dir in /dev /proc /sys /run; do
  sudo mount --bind $dir /mnt/new_root$dir
done

# Mount existing boot partitions
sudo mount /dev/nvme0n1p16 /mnt/new_root/boot
sudo mount /dev/nvme0n1p15 /mnt/new_root/boot/efi
```

---

### 4. Update `fstab`

```bash
# Get UUID of new root LV
blkid /dev/mapper/data_vg-root_lv
```

```bash
# Edit fstab inside the new root
sudo nano /mnt/new_root/etc/fstab
```

Replace the `/` entry with:

```text
UUID=<NEW_UUID>  /  ext4  defaults  0 1
```

---

## Part 4: Bootloader & Initramfs Update (CRITICAL)

### Chroot into the New System

```bash
sudo chroot /mnt/new_root
```

### Update Initramfs to Include LVM

```bash
update-initramfs -u -k all
```

### Update GRUB Configuration

```bash
update-grub
```

Exit chroot:

```bash
exit
```

---

## Part 5: Reboot & Verify

```bash
sudo reboot
```

After reboot, verify root is on LVM:

```bash
mount | grep ' / '
```

Expected output:

```text
/dev/mapper/data_vg-root_lv on / type ext4
```

---

## Part 6: Final Merge (Optional, After Successful Boot)

Once booted successfully from LVM:

```bash
sudo pvcreate /dev/nvme0n1p1
sudo vgextend data_vg /dev/nvme0n1p1
```

---

## Final Warnings

```text
• Do NOT reboot before updating GRUB
• A broken fstab will brick the system
• Remote servers need out-of-band access
```

---

