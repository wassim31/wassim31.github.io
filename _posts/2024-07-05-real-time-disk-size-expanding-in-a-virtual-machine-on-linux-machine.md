---
layout: post
title: "Real-time Disk Size Expanding in a Linux virtual machine"
date: 2024-07-05 15:14:35 +0000
description: "I was compiling the Linux kernel in a VM with a full 20GB logical volume. Here is how to expand the disk without losing your data, step by step."
canonical_devto: https://dev.to/wassim31/real-time-disk-size-expanding-in-a-virtual-machine-on-linux-machine-307i
---

I was compiling the Linux kernel from source in a VM running AlmaLinux with a 20GB logical volume, and I reached the maximum size. I had to expand my disk size without losing my data, or the progress of a 12-hour compilation process (yes, heavy kernel and weak hardware).

This same technique is used in storage management by cloud service providers like AWS and Azure.

**Linux distribution:** AlmaLinux 9.4 on VMware Workstation 7

When you expand the disk size of your Linux virtual machine on VMware, you must adjust the partitions within the system to utilize the additional space. This guide will walk you through the steps required to resize the partitions and filesystems using bash.

## Prerequisites

Before you begin, ensure you have:

- Expanded the disk size of your VM in VMware.
- Root or sudo access to your Linux VM.

## Step 1: Identify the new disk size

First, verify the current disk and partition sizes using the `lsblk` command:

```bash
lsblk
```

The output will show all block devices and their partitions. Identify the disk you have expanded. For example:

```
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0                 11:0    1  988M  0 rom  
nvme0n1            259:0    0  100G  0 disk 
├─nvme0n1p1        259:1    0    1G  0 part /boot
└─nvme0n1p2        259:2    0   19G  0 part 
  ├─almalinux-root 253:0    0   17G  0 lvm  /
  └─almalinux-swap 253:1    0    2G  0 lvm  [SWAP]
```

## Step 2: Resize the partition

Use the `growpart` utility to resize the partition. If it's not already installed, you can install it with:

```bash
sudo dnf install cloud-utils-growpart
```

Then resize the partition `nvme0n1p2`:

```bash
sudo growpart /dev/nvme0n1 2
```

## Step 3: Resize the physical volume (PV)

Next, resize the physical volume to recognize the expanded partition:

```bash
sudo pvresize /dev/nvme0n1p2
```

## Step 4: Verify the physical volume size

Verify the changes using the `pvs` command:

```bash
sudo pvs
```

## Step 5: Resize the logical volume (LV)

Assuming you want to allocate all the new space to the root logical volume (`almalinux-root`):

```bash
sudo lvextend -l +100%FREE /dev/almalinux/root
```

## Step 6: Determine the filesystem type

Before resizing the filesystem, determine the filesystem type. You can do this with either of these commands:

```bash
df -Th
```

or

```bash
lsblk -f
```

## Step 7: Resize the filesystem

Depending on the filesystem type, use the appropriate command.

For ext4:

```bash
sudo resize2fs /dev/almalinux/root
```

For xfs:

```bash
sudo xfs_growfs /
```

## Step 8: Verify the changes

Finally, verify that the new space is available:

```bash
lsblk
df -h
```

## Conclusion

By following these steps, you can effectively utilize the additional disk space after expanding your Linux VM disk on VMware, without losing any data or any running work.
