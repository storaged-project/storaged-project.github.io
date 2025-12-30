---
layout: post
title:  "ATA SMART in libblockdev and UDisks"
date:   2025-12-30 10:00:00 +0100
categories: udisks
author: Tomáš Bžatek
---

For a long time there was a need to modernize the UDisks' way of ATA SMART data retrieval. The ageing `libatasmart` project went unmaintained over time yet there was no other alternative available. There was the `smartmontools` project with its `smartctl` command whose console output was rather clumsy to parse. It became apparent we need to decouple the SMART functionality and create an abstraction.

`libblockdev-3.2.0` introduced a new `smart` plugin API tailored for UDisks needs, first used by the `udisks-2.10.90` public beta release. We haven't received much feedback for this beta release and so the code was released as the final 2.11.0 release about a year later.

While the `libblockdev-smart` plugin API is the single public interface, we created two plugin implementations right away - the existing `libatasmart`-based solution (plugin name `libbd_smart.so`) that was mostly a 1-1 mapping, and a new `libbd_smartmontools.so` plugin based around `smartctl` JSON output.

> Furthermore, there's a promising initiative going on: the [libsmartmon library](https://github.com/smartmontools/smartmontools/issues/409) and if that ever materializes we'd like to build a new plugin around it - likely deprecating the `smartctl` JSON-based implementation along with it. Contributions welcome, this effort deserves more public attention.

Whichever plugin gets actually used is controlled by the libblockdev plugin configuration - see `/etc/libblockdev/3/conf.d/00-default.cfg` for example or, if that file is absent, have a look at the builtin defaults: [https://github.com/storaged-project/libblockdev/blob/master/data/conf.d/00-default.cfg](https://github.com/storaged-project/libblockdev/blob/master/data/conf.d/00-default.cfg). Distributors and sysadmins are free to change the preference so be sure to check it out. Thus whenever you're about to submit a bugreport upstream, please specify which plugin you do use.



## Plugin differences

#### `libatasmart` plugin:

* small library, small runtime I/O footprint
*  preferred plugin, stable for decades
* `libatasmart` unmaintained upstream
* no internal drive/quirk database, possibly reporting false values for some attributes

#### `smartmontools` plugin:

* well-maintained upstream
* extensive `drivedb`, filtering out any false attribute interpretation
* experimental plugin, possibly to be dropped in the future
* heavy on runtime I/O due to additional device scanning and probing (ATA IDENTIFY)
* forking and calling `smartctl`

Naturally the available features do vary across plugin implementations and though we tried to abstract the differences as much as possible, there are still certain gaps.



## The `libblockdev-smart` API

Please refer to our extensive public documentation: [https://storaged.org/libblockdev/docs/libblockdev-SMART.html#libblockdev-SMART.description](https://storaged.org/libblockdev/docs/libblockdev-SMART.html#libblockdev-SMART.description)

Apart from ATA SMART, we also laid out foundation for SCSI/SAS(?) SMART, though currently unused in UDisks and essentially untested. Note that NVMe Health Information has been available through the `libblockdev-nvme` plugin for a while and is not subject to this API.


## Attribute names & validation

We spent great deal of effort to provide unified attribute naming, consistent data type interpretation and attribute validation. While `libatasmart` mostly provides raw values, `smartmontools` benefits from their `drivedb` and provide better interpretation of each attribute value.

For the public API we had to make a decision about attribute naming style. While `libatasmart` only provides single style with no variations, we've discovered lots of inconsistencies just by grepping the `drivedb.h`. For example attribute ID 171 translates to `program-fail-count` with `libatasmart` while `smartctl` may report variations of `Program_Fail_Cnt`, `Program_Fail_Count`, `Program_Fail_Ct`, etc. And with UDisks historically providing untranslated `libatasmart` attribute names, we had to create a translation table for `drivedb.h` -> `libatasmart` names. Check this atrocity out in [https://github.com/storaged-project/libblockdev/blob/master/src/plugins/smart/smart-private.h](https://github.com/storaged-project/libblockdev/blob/master/src/plugins/smart/smart-private.h). This table is by no means complete, just a bunch of commonly used attributes.

Unknown attributes or those that fail validation are reported as generic `attribute-171`. For this reason consumers of the new UDisks release (e.g. Gnome Disks) may spot some differences and perhaps more attributes reported as unknown comparing to previous UDisks releases. Feel free to submit fixes for the mapping table, we've only tested this on a limited set of drives.

Oh, and we also fixed the notoriously broken `libatasmart` drive temperature reporting, though the fix is not 100% bulletproof either.

We've also created an experimental `drivedb.h` validator on top of `libatasmart`, mixing the best of both worlds, with uncertain results. This feature can be turned on by the `--with-drivedb[=PATH]` configure option.



## Disabling ATA SMART functionality in UDisks

UDisks 2.10.90 release also brought a new configure option `--disable-smart` to disable ATA SMART completely. This was exceptionally possible without breaking public ABI due to the API providing the [Drive.Ata.SmartUpdated](https://storaged.org/udisks/docs/gdbus-org.freedesktop.UDisks2.Drive.Ata.html#gdbus-property-org-freedesktop-UDisks2-Drive-Ata.SmartUpdated) property indicating the timestamp the data were last refreshed. When disabled compile-time, this property remains always set to zero.

We also made SMART data retrieval work with `dm-multipath` to avoid accessing particular device paths directly and tested that on a particularly large system.


## Drive access methods

The `ID_ATA_SMART_ACCESS` udev property - see [man udisks(8)](https://storaged.org/udisks/docs/udisks.8.html). This property was a very well hidden secret, only found by accident while reading the `libatasmart` code. As such, this property was in place for over a decade. It controls the access method for the drive. Only `udisks-2.11.0` learned to respect this property in general no matter what `libblockdev-smart` plugin is actually used.

Those who prefer UDisks to avoid accessing their drives at all may want to set this `ID_ATA_SMART_ACCESS` udev property to `none`. The effect is similar to compiling UDisks with ATA SMART disabled, though this allows fine-grained control with the usual udev rule match constructions.


## Future plans, nice-to-haves

Apart from high hopes for the aforementioned [libsmartmon library](https://github.com/smartmontools/smartmontools/issues/409) effort there are some more rough edges in UDisks.

For example, housekeeping could use refactoring to allow arbitrary intervals for specific jobs or even particular drives other than the fixed 10 minutes interval that is used for SMART data polling as well. Furthermore some kind of throttling or a constrained worker pool should be put in place to avoid either spawning all jobs at once (think of spawning `smartctl` for your 100 of drives at the same time) or to avoid bottlenecks where one slow housekeeping job blocks the rest of the queue.

At last, make SMART data retrieval via USB passthrough work. If that happened to work in the past, it was a pure coincidence. After receiving dozen of bugreports citing spurious kernel failure messages that often led to a USB device being disconnected, we've disabled our ATA device probes for USB devices. As a result the `org.freedesktop.UDisks2.Drive.Ata` D-Bus interface gets never attached for USB devices.
