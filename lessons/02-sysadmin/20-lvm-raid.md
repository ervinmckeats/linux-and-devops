# Lesson 20. LVM, RAID, snapshots

## Goal
Resize storage without downtime, survive a dead disk, and take consistent snapshots.

## LVM: flexible volumes

```
 /dev/sdb   /dev/sdc         PV  physical volumes (disks or partitions)
     \        /
      vg_data                VG  volume group (a pool of space)
     /       \
 lv_db     lv_logs           LV  logical volumes (used like partitions)
   |          |
  xfs       ext4             filesystems
```

You can add disks to a VG and grow LVs while mounted. Ubuntu Server and RHEL use LVM by default.

```bash
sudo pvcreate /dev/sdb /dev/sdc
sudo vgcreate vg_data /dev/sdb /dev/sdc
sudo lvcreate -n lv_db -L 20G vg_data
sudo lvcreate -n lv_logs -l 100%FREE vg_data
sudo mkfs.xfs /dev/vg_data/lv_db

# short status
pvs; vgs; lvs

sudo lvdisplay
```

### Grow a volume (online)

```bash
# -r resizes the filesystem too
sudo lvextend -r -L +10G /dev/vg_data/lv_db

# use all free space
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

Ubuntu installs often leave half the VG unused. `vgs` shows it under `VFree`.

### Add a disk to a full VG

```bash
sudo pvcreate /dev/sdd
sudo vgextend vg_data /dev/sdd
sudo lvextend -r -L +50G /dev/vg_data/lv_db
```

### Snapshots

A copy-on-write point-in-time view. Use it for backups of a consistent state or before a risky upgrade.

```bash
# 5G for changes
sudo lvcreate -s -n db_snap -L 5G /dev/vg_data/lv_db

# nouuid needed for xfs
sudo mount -o ro,nouuid /dev/vg_data/db_snap /mnt/snap

# ... back up from /mnt/snap ...
sudo umount /mnt/snap && sudo lvremove /dev/vg_data/db_snap
# or roll back the origin to the snapshot:
sudo lvconvert --merge /dev/vg_data/db_snap
```

If the snapshot's space fills up, the snapshot becomes invalid. Snapshots also slow writes. Remove them after use.

## RAID: redundancy

| Level | Disks | Survives | Usable space | Use |
|---|---|---|---|---|
| RAID 0 | 2+ | nothing | 100% | speed only, scratch data |
| RAID 1 | 2 | 1 disk | 50% | boot, small servers |
| RAID 5 | 3+ | 1 disk | (n-1)/n | read-heavy, rebuild is slow and risky on big disks |
| RAID 6 | 4+ | 2 disks | (n-2)/n | large arrays |
| RAID 10 | 4+ | 1 per mirror | 50% | databases |

**RAID is not a backup.** It does *not* protect from `rm -rf`, ransomware, or a bad deploy.

### Software RAID with mdadm

```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# status and rebuild progress
cat /proc/mdstat

sudo mdadm --detail /dev/md0
sudo mkfs.ext4 /dev/md0
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

Replace a failed disk:

```bash
sudo mdadm /dev/md0 --fail /dev/sdb --remove /dev/sdb
# swap the disk physically / attach a new virtual disk
sudo mdadm /dev/md0 --add /dev/sdd
watch cat /proc/mdstat
```

Monitor arrays: `mdadm --monitor` sends mail, or export `/proc/mdstat` to your monitoring (lesson 56).

## Practice

1. Add 3 small disks to your VM. Create a VG from two of them, one LV of 50%, ext4, mount it.
2. Fill the LV with data, then grow it online with `lvextend -r` and the third disk.
3. Take a snapshot, delete files in the origin, restore them with `lvconvert --merge`.
4. Build a RAID 1 from two disks, fail one, replace it and watch the rebuild.

## Check yourself

- What are PV, VG and LV?
- Why is RAID not a backup?
- What happens when an LVM snapshot runs out of space?

## Next
[Lesson 21. Archives and backups](21-archives-backups.md)
