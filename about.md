---
layout: page
title: "About"
permalink: /about/
---

Storaged Project is a collection of open-source storage management tools, libraries and APIs used for creating, managing and monitoring storage devices across the GNU/Linux ecosystem.

### Projects

- **[blivet](/blivet/)** — A Python library for storage management, originally written as the storage backend for the Anaconda installer.
- **[blivet-gui](/blivet-gui/)** — A graphical frontend for blivet, providing a user-friendly interface for managing storage devices.
- **[libblockdev](/libblockdev/)** — A C library for working with block devices, offering a plugin-based approach to storage technologies.
- **[libbytesize](/libbytesize/)** — A library for working with sizes in bytes.
- **[UDisks](/udisks/)** — A D-Bus service for accessing and manipulating storage devices.

The diagram below shows how our projects relate to each other and to some notable external projects that use our APIs. Arrows indicate a "uses" relationship.

{% include projects-graph.svg %}

All our projects are hosted on [GitHub](https://github.com/storaged-project).
