---
layout: post
title:  "Partitioning with Ansible Storage Role: Partitions"
date:   2025-12-15 10:13:00 +0100
categories: storage-role
author: Vojtech Trefny
---

The [https://linux-system-roles.github.io/storage/](storage role) always allowed creating and managing different storage technologies like LVM, LUKS encryption or MD RAID, but one technology seemed to be missing for a long time, and surprisingly, it was the most basic one, the actual partitioning. Support for partition management was always something that was planned for the storage role, but it was never a high priority. From the start, the role could create partitions. When creating a more complex storage setup on an empty disk, for example creating a new LVM volume group or adding a new physical volume to an existing LVM setup, the role would always automatically create a single partition on the disk. But that was all the role could do, just one single partition spanning the entire disk.

The reason for this limitation was simple: creating multiple partitions is something usually reserved for the OS installation process, where users need to have separate partitions required by the bootloader, like `/boot` and `/boot/efi`. The more advanced "partitioning" is then delegated to a more complex storage technologies like LVM, which is where most of the changes are done in an existing system and where users will usually employ Ansible to make changes later.

But the requirement for more advanced partition management was always there, and since the [1.19 release](https://github.com/linux-system-roles/storage/releases/tag/1.19.0), the role can now create and manage partitions in the Ansible way.

### Partition Management with Storage Role

The usage of the role for partition management is simple and follows the same logic as the other storage technologies, with the management divided into two parts: managing the `storage_pools`, which in the case of partitions is the underlying disk (or to be more precise, the partition table), and the `volumes`, which are the partitions themselves. A simple playbook to create two partitions on a disk can look like this:

```
  roles:
    - name: linux-system-roles.storage
      storage_pools:
        - name: sdb
          type: partition
          disks: sdb
          volumes:
            - name: sdb1
              type: partition
              size: 1 GiB
              fs_type: ext4
            - name: sdb2
              type: partition
              size: 10 GiB
              fs_type: ext4
```

and the partitions it creates will look like this

```
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS FSTYPE
sdb      8:16   0  20G  0 disk
├─sdb1   8:17   0   1G  0 part             ext4
└─sdb2   8:18   0  10G  0 part             ext4
```

Other filesystem-related properties (like `mount_point` or `fs_label`) can be specified, and these work in the same way as for any other volume type.

The only property that is specific to partitions is `part_type`, which allows you to choose a partition type when using the MBR/MSDOS partition table. Supported types are `primary`, `logical` and `extended`. If you don't specify the partition type, the role will create the first three partitions as primary and for the fourth one, add an extended partition and create it as a logical partition inside it. On GPT, which is used as the default partition table, the partition type is ignored.

Encrypted partitions can be created by adding the `encryption: true` option for the partition and setting the passphrase:

```
  roles:
    - name: linux-system-roles.storage
      storage_pools:
        - name: sdb
          type: partition
          disks: sdb
          volumes:
            - name: sdb1
              type: partition
              size: 1 GiB
              fs_type: ext4
              encryption: true
              encryption_password: "aaaaaaaaa"
            - name: sdb2
              type: partition
              size: 10 GiB
              fs_type: ext4
              encryption: true
              encryption_password: "aaaaaaaaa"
```

Don't forget that adding the encryption layer is a destructive operation -- if you run the two playbooks above one after another, the filesystems created by the first one will be removed, and all data on them will be lost. Adding the LUKS encryption layer (so-called re-encryption) is currently not supported by the role.

### Idempotency and Partition Numbers

One of the core principles of Ansible is idempotency, or the ability to re-run the same playbook, and if the system is in the state specified by the playbook, no changes will be made.

This is true for partitioning with the storage role as well. When running the playbook from our example above for the second time, the role will check the `sdb` disk and look for the two specified partitions. And if there are two partitions 1 and 10 GiB large, it won't do anything. This is how the role works in general, but with partitions, there is a new challenge: partitions don't have unique names and using partition numbers for idempotency can be tricky.

> Did you know that partition numbers for logical partitions are not stable? If you have two logical partitions `sdb5` and `sdb6`, removing the `sdb5` partition will automatically re-number the `sdb6` partition to `sdb5`.

Predicting the partition name is not always straightforward. For example, disks that end in a number (common with NVMe drives) require adding a `p` separator before the partition number (`nvme0n1` becomes `nvme0n1p1`).

For these reasons, the role requires explicitly using the `state: absent` option to remove a partition, and partitions can be referred to by their numbers in the playbooks as well as their full names. So, for example, the following playbook will resize the `sdb2` partition from our first example

```
  roles:
    - name: linux-system-roles.storage
      storage_pools:
        - name: sdb
          type: partition
          disks: sdb
          volumes:
            - name: 2
              type: partition
              size: 15 GiB
              fs_type: ext4
```

and the first partition won't be removed, because it is not explicitly mentioned as `absent`, only omitted in the playbook:

```
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS FSTYPE
sdb      8:16   0  20G  0 disk
├─sdb1   8:17   0   1G  0 part             ext4
└─sdb2   8:18   0  15G  0 part             ext4
```

### Feedback and Future Features

With this change, the storage role can now manage all basic storage technologies. We are of course not yet covering all the potential features, but we are always looking for more ideas from our users. If you have any features you'd like to see in the role, please don't hesitate and [let us know](https://github.com/linux-system-roles/storage/issues).
