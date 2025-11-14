---
layout: page
title: "libblockdev"
permalink: /libblockdev/
---

[Source code](https://github.com/storaged-project/libblockdev) | [API Documentation](/libblockdev/docs) | [Developer Guide](https://github.com/storaged-project/libblockdev/blob/master/README.DEVEL.md)

libblockdev is a C library supporting GObject introspection for manipulation of
block devices. It has a plugin-based architecture where each technology (like
LVM, Btrfs, MD RAID, Swap,...) is implemented in a separate plugin, possibly
with multiple implementations (e.g. using LVM CLI or the new LVM DBus API).

#### Features

Following storage technologies are supported by libblockdev

 - partitions
   - MSDOS, GPT
 - filesystem operations
   - ext2, ext3, ext4, xfs, vfat, ntfs, exfat, btrfs, f2fs, nilfs2, udf
   - mounting
 - LVM
   - thin provisioning, LVM RAID, cache, LVM VDO
 - BTRFS
   - multi-device volumes, subvolumes, snapshots
 - swap
 - encryption
   - LUKS, TrueCrypt/VeraCrypt, BitLocker, FileVault2
   - integrity
   - SED OPAL
 - DM (device mapper)
 - loop devices
 - MD RAID
 - multipath
 - s390
   - DASD, zFCP
 - NVDIMM namespaces (deprecated)
 - NVMe
 - SMART
