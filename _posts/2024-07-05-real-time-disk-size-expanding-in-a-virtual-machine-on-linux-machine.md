---
layout: post
title: "Real-time Disk Size Expanding in a Linux virtual machine"
date: 2024-07-05 15:14:35 +0000
description: "I was compiling the Linux kernel from source and i had a vm running AlmaLinux with 20GB Logical..."
canonical_devto: https://dev.to/wassim31/real-time-disk-size-expanding-in-a-virtual-machine-on-linux-machine-307i
---

I was compiling the Linux kernel from source and i had a vm running AlmaLinux with 20GB Logical volume, so i attained the maximum size, i had to expend my disk size without losing my data, the progress of a 12 hours compilation process(yes heavy kernel, and a weak hardware)

this solution is used in storage management in cloud services providers AWS, Azure ..

**_Linux Distribution_** : AlmaLinux 9.4 on Vmware Workstation 7

When you expand the disk size of your Linux virtual machine (VM) on VMware, you must adjust the partitions within the system to utilize the additional space. 

This guide will walk you through the steps required to resize the partitions and filesystems using bash.

**Prerequisites**
Before you begin, ensure you have:

Expanded the disk size of your VM in VMware.

Root or sudo access to your Linux VM.

**Step 1:** Identify the New Disk Size
First, verify the current disk and partition sizes using the lsblk command:

```
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
**Step 2:** Resize the Partition
Use the **growpart** utility to resize the partition. If it's not already installed, you can install it with:


```
sudo dnf install cloud-utils-growpart

```
Then, resize the partition nvme0n1p2:


```
sudo growpart /dev/nvme0n1 2

```
**Step 3:** Resize the Physical Volume (PV)
Next, resize the physical volume to recognize the expanded partition:

```
sudo pvresize /dev/nvme0n1p2

```
**Step 4:** Verify the Physical Volume Size
Verify the changes using the pvs command:

```
sudo pvs

```
**Step 5:** Resize the Logical Volume (LV)

Assuming you want to allocate all the new space to the root logical volume (almalinux-root):

```
sudo lvextend -l +100%FREE /dev/almalinux/root
```
**Step 6:** Determine the Filesystem Type
Before resizing the filesystem, determine the filesystem type. You can do this with the **df -Th** or **lsblk -f** commands:


```
df -Th

```
or

```
lsblk -f

```
**Step 7:** Resize the Filesystem
Depending on the filesystem type, use the appropriate command to resize the filesystem.

**For ext4:**

```
sudo resize2fs /dev/almalinux/root

```
**For xfs:**


```
sudo xfs_growfs /

```
**Step 8:** Verify the Changes
Finally, verify that the new space is available using the lsblk and df -h commands:


```
lsblk
df -h
```
**Conclusion**

By following these steps, you can effectively utilize the additional disk space after expanding your Linux VM disk on VMware.
