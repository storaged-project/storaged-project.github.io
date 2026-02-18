---
layout: post
title:  "Filtering Devices with LVM Devices File"
date:   2026-02-18 8:13:00 +0100
categories: lvm
author: Vojtech Trefny
---

To control which devices LVM can work with, it was always possible to configure filtering in the `devices` section of the `/etc/lvm/lvm.conf` configuration file. But filtering devices this way was not very simple and could lead to problems when using paths like `/dev/sda` which are not stable. Many users also didn't know this possibility exists and while using this type of filtering is possible for a single command with the `--config` option, it is not very user friendly. This all changed recently with the introduction of the new configuration file `/etc/lvm/devices/system.devices` and the corresponding `lvmdevices` command in LVM 2.03.12. A new option `--devices` was also added to the existing LVM commands for a quick way to limit which devices one specific command can use.

### LVM Devices File

As was said above, there is a new `/etc/lvm/devices/system.devices` configuration file. When this file exists, it controls which devices LVM is allowed to scan. Instead of relying on matching the device path, the devices file uses stable identifiers like WWID, serial number or UUID.

A device file on a simple system with a single physical volume on a partition would look like this:

```
# LVM uses devices listed in this file.
# Created by LVM command vgimportdevices pid 187757 at Fri Feb 13 16:44:45 2026
# HASH=1524312511
PRODUCT_UUID=4d58d0c1-8b67-4fa6-a937-035d2bfbb220
VERSION=1.1.1
IDTYPE=devname IDNAME=/dev/sda2 DEVNAME=/dev/sda2 PVID=rYeMgwy0mO0THDagB6k8mZkoOSqAWfte PART=2
```

When the devices file is enabled, LVM will only scan and operate on devices listed in it. Any device not present in the file is invisible to LVM, even if it has a valid PV header.

This is the biggest change brought in with this feature. The old `lvm.conf` based filters were always optional and LVM always scanned all devices in the system, unless told otherwise. This could cause problems on systems with many disks, where LVM (especially during boot) could take a long time scanning devices that did not even "belong" to it.

By default, the LVM devices file is enabled with the latest versions of LVM and on systems without preexisting volume groups, creating new LVM setups with commands like `pvcreate` or `vgcreate` will automatically add the new physical volumes to the devices file. If desired, this feature can be disabled by setting `use_devicesfile=0` in `lvm.conf` or by simply removing the existing devices file. On systems without the devices file, LVM will simply scan all devices in the system the same way it did before introduction of this configuration file.

### Managing Devices with `lvmdevices` and `vgimportdevices`

On most newly installed systems with LVM, the devices file should be already present and populated, but you might want to either create it later on systems installed with an older version of LVM, or manage some devices manually. It is possible to modify the `system.devices` manually, but a new command `lvmdevices` was added for simple management of the file.

To simply import all devices in an existing volume group, `vgimportdevices <vgname>` can be used and for all volume groups in the system, `vgimportdevices -a` can be used.

A single physical volume can be added to the file with `lvmdevices --adddev` and removed with `lvmdevices --deldev`.

To check all entries in the devices file, `lvmdevices --check` can be used and any issues found by the check command can be fixed with `lvmdevices --update`.

#### Backups

In the sample devices file above, you might have noticed the `VERSION` field. This is the current version of the file. LVM automatically makes a backup of the file with every change and old versions of the file can be found in the `/etc/lvm/devices/backup` directory. So if you make some mistakes when changing the file with `lvmdevices`, you can simply restore to a previous version of the file.

### Overriding the Devices File and Filtering with Commands

Together with the devices file feature, a new option `--devices` was added to all LVM commands. This option allows specifying devices which are visible to the command. This overrides the existing devices file so it can be used either to restrict the command to work only on a subset of devices specified in the devices file or even to allow it to run on devices not specified in the file at all.

This option is also very useful when dealing with multiple volume groups with the same name. This is a known limitation of LVM -- two volume groups with the same name cannot coexist in one system and LVM will refuse to work without renaming one of them. This can be a problem when dealing with cloned disks or backups. With `--devices`, commands like `vgs` can be restricted to "see" only one of the volume groups.

### Issue: Missing Volume Group

As mentioned above, when installing a new system with LVM, for the newly created volume groups, the used devices will be added to the devices file. Fedora (and RHEL) installer, Anaconda, will also add all other volume groups present during installation to the devices file so these will also be visible in the installed system. The problems start when a device with a volume group is added to the system after installation. The volume group (and any logical volumes in it) is suddenly invisible. Even commands like `vgs` will simply ignore it, because its physical volumes are not listed in the devices file.

This can be a problem on dual boot systems with encryption. Because the second system's volume group is "hidden" by the encryption layer, [it is not visible during installation and not added to the devices file](https://discussion.fedoraproject.org/t/luks-group-doesnt-show-up/97164). When the user unlocks the LUKS device in their newly installed system, they can't access their second system. Unfortunately in this situation, the only solution is to manually add the second system's volume group with `vgimportdevices` as described above.

### Conclusion

The LVM devices file provides a cleaner and more reliable way to control which devices LVM uses, replacing the old `lvm.conf` based filtering with stable device identifiers and simple management through the `lvmdevices` command. Overall, for most users the devices file should work transparently without any manual configuration needed.
