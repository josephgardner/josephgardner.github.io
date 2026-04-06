---
layout: post
title: Recovering Data from a Dead Synology NAS on Apple Silicon
comments: true
---

My Synology DS712+ decided to give up the ghost. The good news: I have a new Synology arriving tomorrow and can just pop the drives in. The bad news: I wanted to back up some data *before* trusting the migration process. Here's how I mounted the drive read-only on an M1 Mac using UTM and Ubuntu.

### The Challenge

Synology uses a layered storage stack:
1. **mdadm** - Linux software RAID (even for single-disk setups, it uses RAID1 with one member)
2. **LVM** - Logical Volume Manager on top of the RAID
3. **ext4** - The actual filesystem

macOS can't read any of this natively, so we need a Linux VM.

### Requirements

- Mac with Apple Silicon (M1/M2/M3)
- [UTM](https://mac.getutm.app/) (free, QEMU-based virtualization)
- Ubuntu 20.04 or similar (newer kernels can have USB passthrough issues)
- SATA to USB adapter (powered is better; avoid UAS-only adapters)

### Step 1: Release the Disk from macOS

When you plug in the Synology drive, macOS will complain that it can't read the disk. **Click Eject, not Ignore.** This releases the USB interface so UTM can grab it.

Quit Disk Utility if it's open.

### Step 2: USB Passthrough in UTM (The Tricky Part)

This is where I wasted the most time. On Apple Silicon with UTM, you need to attach **both** the disk and its USB hub/adapter.

1. Open UTM → select your Ubuntu VM → Edit → USB
2. Attach both devices:
   - External Disk 3.0 (the SATA drive)
   - USB-C Digital AV Multiport Adapter (or whatever hub you're using)
3. Start the VM
4. Don't hot-plug or unplug after boot

If you only attach the disk, it silently fails. Both are needed.

### Step 3: Verify Linux Sees the Drive

```bash
lsblk -o NAME,SIZE,FSTYPE
```

You should see something like:

```
sdb     7.3T
├─sdb1  linux_raid_member
├─sdb2  linux_raid_member
├─sdb5  linux_raid_member
└─sdb6  linux_raid_member
```

If you don't see the partitions, USB passthrough failed. Go back to step 2.

### Step 4: Assemble the RAID Arrays

Here's where I hit my first real snag. My Synology had **two** data partitions that needed to be assembled:

```bash
# Stop any auto-assembled arrays first
sudo mdadm --stop /dev/md126
sudo mdadm --stop /dev/md127

# Assemble both partitions as read-only RAID arrays
sudo mdadm --assemble --run --readonly /dev/md126 /dev/sdb5
sudo mdadm --assemble --run --readonly /dev/md127 /dev/sdb6
```

Verify they're running:

```bash
cat /proc/mdstat
```

You should see both arrays listed as `active (read-only)`.

### Step 5: Activate LVM

```bash
sudo vgscan
```

This should find `vg1000` (Synology's default volume group name).

```bash
sudo pvscan
```

**This is critical.** You should see both `/dev/md126` and `/dev/md127` listed as physical volumes in `vg1000`. If you only see one, the LVM won't mount correctly because half the logical volume is missing.

Now activate the volume group:

```bash
sudo vgchange -ay vg1000
```

Verify the logical volume is complete:

```bash
sudo dmsetup table
```

You should see `vg1000-lv` with segments pointing to real devices (like `9:126` and `9:127`), **not** any `error` segments. If you see a `vg1000-lv-missing_0_0` with `error`, you haven't assembled all the RAID arrays.

### Step 6: Mount Read-Only

```bash
sudo mkdir -p /mnt/syno
sudo mount -o ro,noload /dev/vg1000/lv /mnt/syno
```

The `-o ro,noload` flags are important:
- `ro` - read-only mount
- `noload` - don't replay the ext4 journal (prevents any writes)

### Step 7: Browse Your Data

```bash
ls /mnt/syno
```

You'll see the typical Synology layout:

```
@appstore
@docker
Data
docker
homes
photo
Plex
Public
surveillance
Time Machine
video
```

The `@` prefixed directories are Synology system folders. Your data is in the others.

### Copying Data Off

I used rsync to copy what I needed:

```bash
rsync -avh --progress /mnt/syno/photo/ /path/to/backup/
```

Everything stays read-only on the source.

### Clean Shutdown

```bash
sudo umount /mnt/syno
sudo vgchange -an vg1000
sudo mdadm --stop /dev/md126
sudo mdadm --stop /dev/md127
sudo poweroff
```

Only unplug the disk after the VM has fully powered down.

### Lessons Learned

1. **macOS Eject ≠ Ignore** - You must eject the disk so UTM can claim the USB interface
2. **UTM USB passthrough requires the hub too** - Attaching only the disk fails silently on Apple Silicon
3. **Synology may use multiple RAID arrays for one volume** - My 7.3TB volume was split across sdb5 (2.7TB) and sdb6 (4.6TB), both as RAID1 members that LVM combined into one logical volume
4. **The `noload` mount option is your friend** - It prevents the filesystem from replaying its journal, which would write to the disk
5. **Loop devices and offset tricks are red herrings** - You can't bypass the mdadm/LVM layers

The whole process took about an hour of trial and error, but once I understood the storage stack, it made sense. Now I can confidently migrate to the new NAS knowing my data is backed up.
