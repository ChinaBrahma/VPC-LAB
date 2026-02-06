
---

# Generalized Playbook: Root Partition Migration to LVM

This playbook describes a **safe, repeatable process** for migrating a Linux system’s root filesystem (`/`) from a **standard partition** to **LVM**, using a secondary disk as a staging target.

---

## 1. Prerequisites

Before beginning, ensure the following conditions are met:

* A Linux system with a **non-LVM root filesystem**
* A **secondary disk or partition** (e.g., `/dev/sdb`, `/dev/nvme1n1`)
* **Root or sudo access**
* A **verified full backup or snapshot** of the system

> ⚠️ Root filesystem migration is a **destructive operation** if misconfigured.
> Never proceed without a rollback plan.

---

## 2. Phase 1: Initialize LVM Structure

Prepare the target disk to host the new root filesystem.

### Initialize the Physical Volume (PV)

```bash
sudo pvcreate /dev/sdX
```

> Replace `sdX` with the target disk (example: `/dev/sdb`, `/dev/nvme1n1`).

---

### Create the Volume Group (VG)

```bash
sudo vgcreate system_vg /dev/sdX
```

---

### Create the Root Logical Volume (LV)

```bash
sudo lvcreate -n root_lv -l 100%FREE system_vg
```

---

### Format the Logical Volume

```bash
sudo mkfs.ext4 /dev/system_vg/root_lv
```

---

## 3. Phase 2: Live Data Migration

Clone the active operating system into the new LVM-backed root volume.

### Mount the Target Root Volume

```bash
sudo mkdir -p /mnt/new_root
sudo mount /dev/system_vg/root_lv /mnt/new_root
```

---

### Synchronize the Root Filesystem

```bash
sudo rsync -axHAX --progress / /mnt/new_root/
```

**Flag rationale:**

* `-a` → archive mode
* `-x` → stay on one filesystem (avoids `/proc`, `/sys`)
* `-H` → preserve hard links
* `-A` → preserve ACLs
* `-X` → preserve extended attributes

---

## 4. Phase 3: Chroot and Bootloader Configuration

Prepare the cloned system to boot from LVM.

---

### Mount Required Virtual Filesystems

```bash
for i in /dev /dev/pts /proc /sys /run; do
  sudo mount -B $i /mnt/new_root$i
done
```

---

### Mount Boot Partitions (If Separate)

```bash
sudo mount /dev/sdX_boot /mnt/new_root/boot
sudo mount /dev/sdX_efi /mnt/new_root/boot/efi
```

> Adjust partition names according to your disk layout.

---

### Enter the New Root Environment

```bash
sudo chroot /mnt/new_root
```

---

### Update `/etc/fstab`

Edit the file:

```bash
nano /etc/fstab
```

Replace the `/` entry with:

```text
/dev/mapper/system_vg-root_lv  /  ext4  defaults  0 1
```

> UUIDs may be used instead for higher resilience.

---

### Remove Cloud-Specific Boot Constraints (AWS / cloud-init Only)

```bash
rm /etc/default/grub.d/40-force-partuuid.cfg
```

---

### Rebuild Boot Artifacts

```bash
update-initramfs -u
update-grub
```

---

### Install GRUB to the Boot Disk

```bash
grub-install /dev/sdX
```

> This must be the **actual boot disk**, not a partition.

---

### Exit Chroot and Reboot

```bash
exit
```

```bash
sudo reboot
```

---

## 5. Phase 4: Merge Original Disk Space into LVM

After confirming the system boots successfully from LVM, reclaim the original disk.

---

### Convert Old Root Partition into a Physical Volume

```bash
sudo pvcreate /dev/sdY_old
```

---

### Extend the Volume Group

```bash
sudo vgextend system_vg /dev/sdY_old
```

---

### Extend the Root Logical Volume

```bash
sudo lvextend -l +100%FREE /dev/system_vg/root_lv
```

---

### Resize the Filesystem

```bash
sudo resize2fs /dev/system_vg/root_lv
```

---

## 6. Verification & Validation

Use the following commands to confirm system state:

### Verify Root Is on LVM

```bash
lsblk
```

---

### Verify Physical Volumes

```bash
pvs
```

---

### Verify Volume Group Size

```bash
vgs
```

---

### Confirm Usable Root Space

```bash
df -h /
```

---

## Final Notes

* Always validate **before and after every reboot**
* LVM enables:

  * Online resizing
  * Disk aggregation
  * Snapshot-based recovery
* This playbook applies cleanly to:

  * Bare metal
  * Cloud VMs
  * NVMe and SATA devices

---


Just say the word.
