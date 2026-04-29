---
layout: post
title:  "Libblockdev 3.5.0 released"
date:   2026-04-29 10:25:00 +0100
categories: libblockdev
author: Vojtech Trefny
---

A new upstream version of *Libblockdev* was released on Monday (April 27th) -- [3.5.0](https://github.com/storaged-project/libblockdev/releases/tag/3.5.0). This release brings both new functions and a large number of bug fixes.

### Btrfs plugin: recursive deletion of subvolumes and device stats

Two new functions have been added to the btrfs plugin.

`bd_btrfs_delete_subvolume_recursive` can be used to remove a btrfs subvolume and all its children in one call. This was prompted by [an issue reported for blivet-gui](https://github.com/storaged-project/blivet-gui/issues/493) requesting this feature. While we could do this manually, using the new btrfs `--recursive` option can noticeably speed up this operation.

`bd_btrfs_device_stats` can be used to get device statistics for a btrfs volume. This is equivalent to the `btrfs device stats` command. In the future we want to bring this functionality to UDisks and ultimately to Cockpit, which [requested this feature](https://github.com/storaged-project/libblockdev/issues/1167).

### Crypto plugin: open with flags

All "open" functions in the crypto plugin were extended to allow passing the activation flags to libcryptsetup. This for example allows enabling discard when opening encrypted devices. These new functions will be used in the next version of [UDisks](https://github.com/storaged-project/udisks/issues/1409) to correctly support options specified in `/etc/crypttab`.

### NVMe plugin: listing namespaces for a controller

`bd_nvme_find_namespaces_for_ctrl` is a new utility function to find all namespaces associated with a given controller. This mirrors the existing `bd_nvme_find_ctrls_for_ns` function but performs the reverse operation.

### Bug fixes

The biggest chunk of the changes in this release are various bug fixes. As described in my [previous post](https://storaged.org/other/2026/04/16/Using-LLM-for-code-review-in-storaged.html), we started experimenting with using LLM for code review and for libblockdev the result is over a hundred issues fixed, mostly memory and resource leaks, logical issues in error paths and various issues in tests and documentation.

At the time of writing, libblockdev 3.5.0 is already available in Fedora Rawhide and Debian Unstable and on its way to Fedora 44. The latest builds for testing and bleeding-edge enthusiasts can be always found in our [Copr repository](https://copr.fedorainfracloud.org/coprs/g/storage/udisks-daily/).
