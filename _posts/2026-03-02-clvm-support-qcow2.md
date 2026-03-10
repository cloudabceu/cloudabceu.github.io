---
layout: post
title: Libvirt/CLVM - Add qcow2 support for logical volumes
toc: true
categories: [Libvirt, CLVM]
---

This post explains a small libvirt patch that adds `qcow2` (and other qemu image formats) support when creating volumes on `logical` (CLVM) storage pools. The patch lets libvirt use `qemu-img` to initialise a LV as a `qcow2` image and then expose that LV to a VM as a block source while preserving activation locking behavior for clustered LVM setups.

<!--more-->

## Patch summary

The patch makes three main changes:

- Extend `volOptions` for `logical` pool types so volumes can select formats other than raw.
- Only call `qemu-img` to create an image when the requested volume format is not `raw` (or when encryption is requested).
- Treat output block devices as `raw` only when qemu reports no format; this allows block LVs to carry a qcow2 image created by `qemu-img`.

Full patch (abridged):

```diff
From 97ccf411b950bc755f2b1ee426217ef4a1b09248 Mon Sep 17 00:00:00 2001
From: Wei Zhou <weizhou@apache.org>
Date: Sun, 1 Mar 2026 10:47:53 +0100
Subject: [PATCH] CLVM: Support qcow2 format

---
 src/conf/storage_conf.c                       |  6 +++++
 src/storage/storage_backend_logical.c         |  4 +++-
 src/storage/storage_util.c                    |  3 ++-
 .../poolcaps-fs.xml                           | 22 ++++++++++++++++++++
 .../poolcaps-full.xml                         | 22 ++++++++++++++++++++
 5 files changed, 55 insertions(+), 2 deletions(-)
@@
+    .volOptions = {
+        .defaultFormat = VIR_STORAGE_FILE_RAW,
+        .lastFormat = VIR_STORAGE_FILE_LAST,
+        .formatFromString = virStorageVolumeFormatFromString,
+        .formatToString = virStorageFileFormatTypeToString,
+    },
@@
-    if (vol->target.encryption &&
+    if ((vol->target.encryption ||
+         (vol->target.format != VIR_STORAGE_FILE_NONE &&
+          vol->target.format != VIR_STORAGE_FILE_RAW)) &&
         virStorageBackendCreateVolUsingQemuImg(pool, vol, NULL, 0) < 0)
         goto error;
@@
-    if (vol->type == VIR_STORAGE_VOL_BLOCK)
+    if (vol->type == VIR_STORAGE_VOL_BLOCK &&
+        info->format == VIR_STORAGE_FILE_NONE)
         info->format = VIR_STORAGE_FILE_RAW;
```

Refer to the original patch for full context and tests updates.

## Why this matters

Previously, logical pools were treated as block-only and libvirt assumed `raw` format for LVs. With this change, you can create a `qcow2` image directly on an LV (using `qemu-img`) and have libvirt handle it consistently during creation and activation. This is useful if you want copy-on-write features or thin-provisioning semantics provided by `qcow2` while still using clustered LVM for shared storage and locking.

## Example: create a qcow2 volume on a logical pool

You can create a volume with `virsh vol-create-as` and request `qcow2` format (replace `shared-lv-pool` and `vm1` with your names):

```bash
virsh vol-create-as shared-lv-pool vm1 10G --format qcow2
```

Equivalent `volume` XML for `virsh vol-create` or API use:

```xml
<volume>
  <name>vm1</name>
  <capacity unit='G'>10</capacity>
  <target>
    <format type='qcow2'/>
    <path>/dev/vg_shared_sdb/vm1</path>
  </target>
</volume>
```

Notes:
- The `path` is the LV device node (e.g. `/dev/vg_shared_sdb/vm1`).
- Libvirt will call `qemu-img create -f qcow2 /dev/vg_shared_sdb/vm1 10G` when applying the patch logic above.

## Example: domain disk XML (use LV as VM disk)

When you attach the LV that contains a `qcow2` image to a domain, declare the driver format and the source device. Example domain disk block device using a qcow2 image on an LV:

```xml
<disk type='block' device='disk'>
  <driver name='qemu' type='qcow2'/>
  <source dev='/dev/vg_shared_sdb/vm1'/>
  <target dev='vda' bus='virtio'/>
</disk>
```

Alternative using `file` type with the device path (both are accepted by libvirt/qemu in practice):

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2'/>
  <source file='/dev/vg_shared_sdb/vm1'/>
  <target dev='vda' bus='virtio'/>
</disk>
```

Important: ensure appropriate host-side locking and activation semantics (CLVM + sanlock/lvmlockd or equivalent) so the LV is exclusively activated on the host running the domain.

## Notes and caveats

- Storing qcow2 images on raw block LVs is supported but has different performance and recovery characteristics compared with storing qcow2 files on a filesystem.
- Backups and snapshots require careful coordination — consider qemu-img snapshot and libvirt snapshot tooling and ensure cluster-wide coordination where appropriate.
