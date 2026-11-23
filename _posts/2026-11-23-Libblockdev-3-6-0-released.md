---
layout: post
title:  "Libblockdev 3.6.0 released"
date:   2026-11-23 14:15:00 +0100
categories: libblockdev
author: Vojtech Trefny
---

A new upstream version of *Libblockdev* was released on November 11th -- [3.6.0](https://github.com/storaged-project/libblockdev/releases/tag/3.6.0). This release adds support for pervasive encryption on IBM Z, new loop and device-mapper functions, and fixes a security issue in the filesystem plugin.

### S390 and crypto plugins: pervasive encryption

The s390 plugin can now use `zkey` to generate, list and remove secure keys in the IBM Z secure key repository. It can also set a key's verification pattern in the LUKS2 metadata and check whether a device is correctly set up for pervasive encryption.

To use one of these secure keys when formatting a LUKS2 device, the crypto plugin now provides `bd_crypto_luks_format_volume_key` which lets the caller supply the volume key instead of having cryptsetup generate one.

### Loop plugin: operating on open file descriptors

The loop plugin already had `bd_loop_setup_from_fd` for setting up a loop device from an open backing file. This release adds `_from_fd` variants for getting loop device information, tearing down a loop device, setting autoclear and updating its capacity. These new functions take an open file descriptor for the loop device itself, so callers can keep working with the device they opened without having to resolve its path again.

### Device-mapper plugin: thin pool statistics

`bd_dm_get_thin_pool_stats` reports used and total block counts for both data and metadata in a device-mapper thin pool, along with its status flags. Unlike the existing LVM-specific functions, it also works with thin pools that aren't managed by LVM.

### Filesystem plugin: security and other fixes

This release fixes [CVE-2026-111581](https://www.cve.org/CVERecord?id=CVE-2026-111581), where the NTFS and VFAT utilities could interpret filesystem labels as command-line options. It also improves compatibility with newer exFAT tools and makes F2FS resizing non-interactive.

### NVMe plugin: switching to libnvme 3.x in 3.7.0

Libblockdev 3.6 is the last release series to support libnvme 1.x. Starting with 3.7.0, the NVMe plugin will require libnvme 3.2 or later. Existing libblockdev function signatures are expected to stay the same, but there will be a few changes to NVMe over Fabrics behavior to watch for.

At the time of writing, libblockdev 3.6.0 is available in [Fedora 45 and Rawhide](https://packages.fedoraproject.org/pkgs/libblockdev/libblockdev/) and in [Debian Testing and Unstable](https://tracker.debian.org/pkg/libblockdev). The latest test builds can be found in our [Copr repository](https://copr.fedorainfracloud.org/coprs/g/storage/udisks-daily/).
