# Lesson 22. Boot process and recovery

## Goal
Know what happens from power-on to login, and bring back a system that does not boot.

## Theory: the boot chain

```
Firmware (UEFI/BIOS)  →  Bootloader (GRUB)  →  Kernel + initramfs  →  systemd (PID 1)  →  targets  →  login
   POST, find disk         menu, kernel args      drivers, find root fs     start services
```

1. **Firmware** tests hardware and starts the bootloader from the EFI partition (`/boot/efi`).
2. **GRUB** loads the kernel (`/boot/vmlinuz-*`) and the **initramfs**: a tiny temporary root filesystem with drivers needed to mount the real root (LVM, RAID, encryption, NVMe).
3. **Kernel** initializes hardware, mounts the real root, starts `/sbin/init` = systemd.
4. **systemd** reaches the default **target**: `multi-user.target` (server) or `graphical.target`.

Knowing the stage tells you where to look: no GRUB menu → disk/firmware; kernel panic → initramfs/root fs; services fail → systemd and logs.

## Commands

```bash
systemctl get-default
sudo systemctl set-default multi-user.target

# time per stage
systemd-analyze

systemd-analyze critical-chain

# errors during this boot
journalctl -b -p err

# kernel arguments used
cat /proc/cmdline

ls /boot

# Debian/Ubuntu, after editing /etc/default/grub
sudo update-grub

# RHEL
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Debian/Ubuntu; RHEL: dracut -f
sudo update-initramfs -u
```

## Rescue modes

In the GRUB menu press `e` on an entry, find the line starting with `linux`, add a parameter at the end, boot with `Ctrl+X`:

| Parameter | Result |
|---|---|
| `systemd.unit=rescue.target` | single-user, basic services, needs root password |
| `systemd.unit=emergency.target` | minimal, root fs read-only |
| `init=/bin/bash` | no systemd at all; root shell immediately |
| `rd.break` (RHEL) | stop inside initramfs |

On cloud VMs you use the provider's **serial console** for this.

## Recovery recipes

### Forgotten root password

1. Boot with `init=/bin/bash`.
2. `mount -o remount,rw /`
3. `passwd root`
4. RHEL with SELinux: `touch /.autorelabel`
5. `exec /sbin/init` (or `sync; reboot -f`)

Anyone with console access can do this. Physical/console access = root. Protect it (GRUB password, disk encryption, restricted console).

### Broken fstab → emergency mode

```bash
# in emergency shell
mount -o remount,rw /

# fix or comment out the bad line, add nofail
vim /etc/fstab

mount -a && systemctl daemon-reload && reboot
```

### System does not boot at all: chroot from a live ISO

Boot a live Ubuntu ISO (or the cloud provider's rescue image), then:

```bash
# find the root partition
lsblk -f

# root (for LVM: vgchange -ay first)
sudo mount /dev/sda2 /mnt

# EFI partition if separate
sudo mount /dev/sda1 /mnt/boot/efi

for d in dev proc sys run; do sudo mount --rbind /$d /mnt/$d; done
sudo chroot /mnt
# now you are "inside" the broken system:
passwd root
apt install --reinstall linux-image-generic
update-initramfs -u -k all
grub-install /dev/sda && update-grub
exit
```

### Disk full, cannot log in

Log in via console as root (root has 5% reserved space on ext4), then free space with `journalctl --vacuum-size`, removing old logs, `apt clean`.

## Practice

Do each in a snapshot:
1. Reset the root password with `init=/bin/bash`.
2. Break `/etc/fstab`, recover from emergency mode.
3. Delete `/boot/grub/grub.cfg`, boot a live ISO, repair with chroot + `update-grub`.
4. Change default target to `rescue.target`, reboot, revert.

## Check yourself

- What is initramfs for?
- Which boot stage failed if you see the GRUB menu but then a kernel panic "unable to mount root fs"?
- Why must you bind-mount `/dev`, `/proc`, `/sys` before chroot?

## Next
[Lesson 23. Network basics](../03-networking/23-network-basics.md)
