Generalized Playbook: Root Partition Migration to LVM
1. Prerequisites
A Linux system with a standard partition (non-LVM) as root (/).
A second available disk or partition (e.g., /dev/sdb or /dev/nvme1n1).
Root or sudo privileges.
Backup: Ensure a full system snapshot or backup exists before proceeding.
2. Phase 1: Initialize LVM Structure
Prepare the target disk to host the new system.
Initialize the Physical Volume (PV):
sudo pvcreate /dev/sdX (Replace sdX with your new disk)
Create a Volume Group (VG):
```
sudo vgcreate system_vg /dev/sdX
```
Create the Logical Volume (LV):
```
sudo lvcreate -n root_lv -l 100%FREE system_vg
```
Format the Volume:
```
sudo mkfs.ext4 /dev/system_vg/root_lv
```
3. Phase 2: Live Data Migration
Clone the current operating system to the LVM volume.
Mount the target:
```
sudo mkdir -p /mnt/new_root
sudo mount /dev/system_vg/root_lv /mnt/new_root
```
Synchronize files:
```
sudo rsync -axHAX --progress / /mnt/new_root/
```
(Note: -x ensures rsync stays on the current filesystem, avoiding /proc, /sys, etc.)
4. Phase 3: Chroot and Bootloader Configuration
Update the cloned system to recognize the LVM path.
Mount system API filesystems:
```
for i in /dev /dev/pts /proc /sys /run; do sudo mount -B $i /mnt/new_root$i; done
```
Mount boot partitions (if separate):
```
sudo mount /dev/sdX_boot /mnt/new_root/boot
sudo mount /dev/sdX_efi /mnt/new_root/boot/efi
```
Enter the environment:
```
sudo chroot /mnt/new_root
```
Update /etc/fstab:
Edit /etc/fstab and replace the / entry with:
```
/dev/mapper/system_vg-root_lv / ext4 defaults 0 1
```
Remove Cloud-Specific Constraints (AWS/Cloud-init only):
```
rm /etc/default/grub.d/40-force-partuuid.cfg
```
Rebuild Boot Images:
```
update-initramfs -u
update-grub
```
Install Bootloader:
```
grub-install /dev/sdX
```
Exit and Reboot:
```
exit
```
```
sudo reboot
```
5. Phase 4: Merging Original Space
Once the system has successfully booted from the LVM volume, reclaim the original disk space.
Convert old partition to PV:
```
sudo pvcreate /dev/sdY_old
```
Extend the VG:
```
sudo vgextend system_vg /dev/sdY_old
```
Extend the LV:
```
sudo lvextend -l +100%FREE /dev/system_vg/root_lv
```
Resize the Filesystem:
```
sudo resize2fs /dev/system_vg/root_lv
```
6. Verification Commands
Use these to confirm the final state:
lsblk: Verify / is mounted on the LVM mapper.
pvs: Check that both disks are listed as Physical Volumes.
vgs: Verify the Volume Group size reflects the sum of both disks.
df -h /: Confirm the total usable space.



