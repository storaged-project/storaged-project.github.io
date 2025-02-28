---
layout: page
title: "blivet"
permalink: /blivet/
---

[Source code](https://github.com/storaged-project/blivet) | [API Documentation](/blivet/docs) | [Developer Guide](https://github.com/storaged-project/blivet/blob/main/CONTRIBUTING.md)

Blivet is a free, open-source python module for configuring storage on Linux. It was originally written as the storage backend for Anaconda — the OS installer used by Fedora and Red Hat Enterprise Linux, among others.

The main thing that makes Blivet stand apart from other storage configuration tools is the ability to model an arbitrarily large set of changes in memory without changing anything on disk. This makes it possible to specify your full configuration before ever changing anything on disk instead of applying changes as you define each layer.

Another powerful feature of Blivet is its DeviceFactory class hierarchy, which allows creation of full device stacks with a single method call using a top-down specification.

Blivet also features a powerful partitioning engine that can allocate new partitions based on flexible specifications.
