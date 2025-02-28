---
layout: post
title:  "Partitioning with Ansible Storage Role: VDO"
date:   2023-10-05 13:17:00 +0100
categories: storage-role
author: Jan Pokorny
---

This time we shall talk about Storage Role support of VDO. The abbreviation stands for Virtual Data Optimizer and that is exactly what it does. It reduces stored data size to save space. To be precise Storage Role utilizes LVM version of VDO called (what a surprise) [LVM VDO](https://man7.org/linux/man-pages/man7/lvmvdo.7.html).

### How Does VDO Do It

There are two main options VDO uses to reduce the data size:

- Data compression
- Data deduplication

Data compression works just like regular file compression. However VDO packs and unpacks blocks of data automatically and on lower level, so user does not even know about it happening.

The same goes for data deduplication. VDO identifies and removes duplicit blocks of data. Redundant blocks are removed and the last remaining copy of the block gets to do all their work.

During the VDO device creation compression and deduplication can be turned on or off.

### Using VDO in Storage Role

We should be already pretty confident about how to use the Storage Role and using VDO is not much different.

To use it the `storage_pool` has to be created. Then set `true` to one or both of the options `compression` and `deduplication` on one of the volumes. This will tell the role to use VDO. **Please note that currently the Storage Role supports only one VDO volume per `storage_pool`**.

You also want to set both the `vdo_pool_size` and `size` options.

Why two sizes?
The first size represented by `vdo_pool_size` option is actual physical space reserved for the compressed data.

The other option - `size` - tells the device how should it present itself on the outside. This value is virtual and can (and it is supposed to) be larger than reserved physical space. By how much is left to users discretion and should be based on estimation of data compressibility.

The playbook for VDO creation then should look like this:

        ---
        - hosts: all
        become: true
        vars:
        storage_safe_mode: false

        tasks:
        - name: Create LVM VDO volume under volume group 'vg1'
        include_role:
        name: linux-system-roles.storage
        vars:
        storage_pools:
                - name: vg1
                disks:
                - "dev/sda"
                - "dev/sdb"
                - "dev/sdc"
                volumes:
                - name: test1
                compression: true
                deduplication: true
                vdo_pool_size: "9 GiB" # space taken on disk
                size: "12 GiB"         # virtual space
                mount_point: "/opt/test1"
                state: present

### Things to Know Before Creating a VDO Device

As goes for all more advanced features that Storage Role provides, VDO is meant for specific use cases.

Some data such as logs are much easier to compress or deduplicate. This makes them much better candidates. On the other hand, using VDO with data that are often modified or already scrambled by encryption can result in just an additional strain on resources.

Data that cannot be easily deduplicated or compressed can also cause a situation when user runs out of physical storage space with VDO showing lots of free space left.

Since the system has no way of telling what kind of data are eventually going to be put on which device, the responsibility of choosing wisely falls upon the user.

### Couple of Tips at the End

And that's it. As I already mentioned, Storage Role VDO uses LVM VDO so its [manpages](https://man7.org/linux/man-pages/man7/lvmvdo.7.html) are a good point to start if you want to know more about it. And for more general information about VDO you can also check [VDO project on Github](https://github.com/dm-vdo/vdo).
