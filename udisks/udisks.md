---
layout: page
title: "udisks"
permalink: /udisks/
---

[Source code](https://github.com/storaged-project/udisks) | [API Documentation](/udisks/docs/latest) | [Developer Guide](https://github.com/storaged-project/udisks/blob/master/HACKING)

UDisks provides multiple tools and libraries for storage configuration and manipulation:

 * **udisksd**

    Daemon providing DBus API for storage configuration and monitoring.
    UDisks DBus API is used for example by Cockpit for storage configuration or by Anaconda installer for iSCSI devices manipulation.

 * **libudisks**

    A wrapper around UDisks DBus API allowing to use UDisks functionality directly from C or Python.
    libudisks is used for example by GNOME Disks.

 * **udiskctl**

    A command line tool for storage manipulation. Allows for example (un)mounting of filesystems, (un)locking of encrypted devices, monitoring of storage events etc.

UDisks has a modular architecture and provides modules for working with some “advanced” storage technologies like LVM, Btrfs, iSCSI and NVMe.
