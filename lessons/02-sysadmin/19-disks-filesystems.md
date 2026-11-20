# Lesson 19. Disks, partitions, mount, fstab, swap

## Goal
Add a disk, format it, mount it permanently, and find what is eating space.

## Theory

```
/dev/sdb                 block device (whole disk)
 └─ /dev/sdb1            partition (GPT table)
     └─ ext4 / xfs       filesystem: inodes, directories, journal
         └─ /data        mount point in the single tree
```

- Device names: `sda` (SATA/SCSI/virtio-scsi), `vda` (virtio), `nvme0n1` + `p1` for partitions.
- **Names can change between boots.** Always refer to disks by **UUID** in configs.
- **Partition table**: GPT (modern, use it) or MBR (legacy, 2 TB limit).
- **Filesystems**: `ext4` (default on Debian/Ubuntu, can shrink), `xfs` (default on RHEL, great for big files, cannot shrink), `btrfs` (snapshots, checksums).
- A **journal** lets the filesystem recover quickly after a crash without a full check.

## Look before you touch

```bash
lsblk -f
blkid
df -hT
df -i
findmnt
sudo fdisk -l
```

- `lsblk -f`: tree of disks, partitions, filesystems, mount points
- `blkid`: UUIDs and types
- `df -hT`: free space per mounted filesystem
- `df -i`: free inodes
- `findmnt`: mount tree

## Add a new disk

```bash
sudo parted /dev/sdb --script mklabel gpt mkpart data ext4 0% 100%
sudo mkfs.ext4 -L data /dev/sdb1
sudo mkdir -p /data
sudo blkid /dev/sdb1
```

- `sudo blkid /dev/sdb1`: copy the UUID

`/etc/fstab`:

```
# device                                  mount  type  options          dump pass
UUID=3f1c...-...                          /data  ext4  defaults,noatime 0    2
```

```bash
sudo findmnt --verify
sudo mount -a
df -h /data
```

- `sudo findmnt --verify`: check fstab syntax
- `sudo mount -a`: mount everything in fstab - test BEFORE reboot

A broken fstab line can stop the boot in emergency mode. Always run `mount -a` after editing. For non-critical disks add `nofail` so the system still boots without them.

Useful options: `noatime` (fewer writes), `ro`, `noexec`, `nosuid`, `nodev` (security for `/tmp`, uploads), `nofail`.

## Where did the space go?

```bash
df -h
sudo du -xh --max-depth=1 / | sort -h
sudo du -sh /var/log/* | sort -h | tail
sudo find / -xdev -type f -size +500M -exec ls -lh {} + 2>/dev/null
sudo ncdu -x /
sudo lsof +L1
sudo journalctl --vacuum-size=200M
sudo apt clean; docker system df
```

- `df -h`: which filesystem is full
- `sudo du -xh --max-depth=1 / | sort -h`: biggest dirs, stay on one fs (`-x`)
- `sudo ncdu -x /`: interactive
- `sudo lsof +L1`: deleted but still open files (lesson 8)

Also: ext4 reserves 5% for root by default: that is why a "full" disk still lets root write.

## Grow a disk in the cloud

After increasing the disk size in the provider console:

```bash
lsblk
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
sudo xfs_growfs /
```

- `lsblk`: disk bigger, partition not yet
- `sudo growpart /dev/sda 1`: grow partition 1 (package cloud-guest-utils)
- `sudo resize2fs /dev/sda1`: ext4
- `sudo xfs_growfs /`: xfs (takes the mount point)

## Swap

Swap is disk space used as overflow memory. A little swap lets the kernel move idle pages out; heavy swapping means you need more RAM.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
swapon --show; free -h
```

## Check and repair

```bash
sudo umount /data && sudo fsck -f /dev/sdb1
sudo xfs_repair /dev/sdb1
sudo smartctl -a /dev/sda
```

- `sudo umount /data && sudo fsck -f /dev/sdb1`: ext4: only on UNMOUNTED fs
- `sudo smartctl -a /dev/sda`: disk health (smartmontools)

## Practice

1. Add a second virtual disk to your VM. Partition, format ext4, mount at `/data` via UUID in fstab. Reboot and check.
2. Break fstab on purpose (typo in UUID) in a snapshot, reboot, recover from emergency mode.
3. Fill `/data` with `fallocate -l` until full, then find the big file with `du`.
4. Create a 1 GB swap file and enable it permanently.
5. Create 200 000 empty files on a small filesystem and watch `df -i`.

## Check yourself

- Why use UUID instead of `/dev/sdb1` in fstab?
- What does `nofail` do, and when do you need it?
- `df` says 100%, `du` disagrees. Two possible reasons?

## Next
[Lesson 20. LVM, RAID, snapshots](20-lvm-raid.md)
