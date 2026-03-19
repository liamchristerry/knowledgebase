
## LVM Hierarchy
Physical Volume > Volume Groups > Logical Volume

## Status Commands 
```
# Overview of everything
pvs          # quick PV summary
vgs          # quick VG summary
lvs          # quick LV summary

# Detailed info
pvdisplay
vgdisplay
lvdisplay

# Detailed with specific target
pvdisplay /dev/sdb
vgdisplay myvg
lvdisplay /dev/myvg/mylv

# See physical extents layout (great for visualizing)
pvs -v
lvs -a -o +devices

# Full scan for LVM on system
lvscan
vgscan
pvscan
```

## Creating PV, VG, LV
```
pvcreate /dev/sdb                 # whole disk (no partition needed - more on this below)
pvcreate /dev/sdb1                 # or a partition

vgcreate myvg /dev/sdb
vgcreate myvg /dev/sdb /dev/sdc    # multiple disks at once

lvcreate -L 50G -n mylv myvg       # gigabytes
lvcreate -L 500M -n mylv myvg      # megabytes
lvcreate -l 100%FREE -n mylv myvg  # presentage of free space to use 

mkfs.ext4 /dev/myvg/mylv           # ext4 format
mkfs.xfs /dev/myvg/mylv            # xfs format

mkdir /mnt/data
mount /dev/myvg/mylv /mnt/data

# Persistent mount in /etc/fstab:
/dev/myvg/mylv  /mnt/data  ext4  defaults  0  2
# Or use UUID (more reliable):
blkid /dev/myvg/mylv
UUID=xxxx-xxxx  /mnt/data  ext4  defaults  0  2
```

## Expanding a PV - Expanding current drive
```
# 1. Rescan to detect new disk size (no reboot needed usually)
echo 1 > /sys/class/block/sdb/device/rescan

# 2.0 If using a whole disk PV, just resize the PV directly:
pvresize /dev/sdb

# 2.1 If using a parition expand that first then follow step 2.0
growpart /dev/sda 3         # expands partition 3

# 3. Confirm the PV now shows more free space
pvs

# 4. Now extend your LV and filesystem (see below
# Extend LV by 20G
lvextend -L +20G /dev/myvg/mylv

# Extend LV to use all free space
lvextend -l +100%FREE /dev/myvg/mylv

# Resize filesystem (do this after, or use -r to do it in one shot)
lvextend -l +100%FREE -r /dev/myvg/mylv    # -r resizes filesystem automatically

# Manual filesystem resize if needed:
resize2fs /dev/myvg/mylv        # ext4 (can do live, no unmount needed)
xfs_growfs /mnt/data            # xfs (pass mountpoint, not device)
```

## Expanding a VG - Adding more disks/PVs
```
# 1. Prep the new disk as a PV
pvcreate /dev/sdc

# 2. Add it to existing VG
vgextend myvg /dev/sdc

# 3. Verify
vgs
pvs

# 4. Now you can expand the LV following above instructions
```

## LVM RAID?
You can use LVM to create a software RAID but not sure if its suggested over mdadm. Looks like LVM uses the same MD kernel packages so it might be a easier to use layer wrapped around mdadm 
```
# RAID 1 but you can also to 5/6 
lvcreate --type raid1 -m1 -L 50G -n mylv myvg

# check status of RAID
lvs -o name,copy_percent,raid_sync_action
```


## LVM Snapshots
Snapshots can fill up and when they do they become unusable. LVM snapshots use Copy-On-Write so its like the opposite of VMware snapshots. Active writes hit the original LV and not the snapshot
```
# Create a snapshot (give it space for changes - 5G is usually enough for short-term)
lvcreate -L 5G -s -n mylv_snap /dev/myvg/mylv

# Check snapshot status and how full it is
lvs -o name,lv_size,data_percent,snap_percent
lvdisplay /dev/myvg/mylv_snap

# Mount and browse the snapshot (read-only or read-write)
mount -o ro /dev/myvg/mylv_snap /mnt/snap
mount /dev/myvg/mylv_snap /mnt/snap    # read-write

# Restore (revert LV to snapshot state) - LV must be unmounted!
umount /mnt/data
lvconvert --merge /dev/myvg/mylv_snap
# Then remount - the snapshot is consumed/deleted automatically

# Remove snapshot without restoring
umount /mnt/snap
lvremove /dev/myvg/mylv_snap
```

## Noteworthy LVM Features
- Stripe data across disks
	- PVs have a concept called `Linear Layout` . This is the default and when you have multiple disks and PVs linear layout means data is written sequentially. (not stripped across disks like raid)
- Thin provisioning LVs and snapshots
	- Snapshots use Copy on Write (CoW) unlike VMware snapshots where all new data is written to the snapshot disk. All data is written to that snapshot disk
	- How CoW works
		- Original block is about to be overwritten
		- LVM intercepts the write
		- Copies the ORIGINAL block into the snapshot volume first
		- Allows the new write to land on the origin volume
		- Snapshot now points at the saved original block
- Move data between PVs for migrations and no downtime


## PV use a Partition or Entire Drive?
- /dev/sda will always be setup with partitions due to need this for booting
- For other drives its ok to use the entire drive for the PV
- Note- some utilities like fdisk will not work if you dont use partitions
-