---
layout: post
title: Rebuilding libvirt on Oracle Linux 9 with CLVM + sanlock patch
toc: true
categories: [Linux, Libvirt]
---

In this guide, I walk through rebuilding `libvirt-daemon-driver-storage-logical` on Oracle Linux 9 and applying a custom CLVM + sanlock patch.

The goal is to produce a rebuilt RPM that can be installed cleanly on Oracle Linux 9 systems while preserving the distro packaging workflow.

In clustered environments, CLVM with `lvmlockd` and `sanlock` is a common approach for shared storage. In practice, libvirt’s `logical` pool behavior can conflict with that model when one host encounters an LV lock held by another host. This patch targets a practical operating model: shared VG visibility with exclusive LV activation (except migration workflows coordinated by an external control plane).

<!--more-->

## Background

`lvmlockd` + `sanlock` is widely used to let multiple hosts access the same VG safely.

The issue appears when libvirt creates or activates a `logical` storage pool. If any LV in the shared VG is already active and locked by another host, libvirt may fail the pool operation instead of tolerating that lock state.

Typical failure example:

```text
error: Failed to create pool from pool.xml
error: internal error: Child process (/usr/sbin/vgchange -aly vg_shared_sdb) unexpected exit status 5:   LV locked by other host: vg_shared_sdb/vm2
  Failed to lock logical volume vg_shared_sdb/vm2.

```

The patch in this post is designed around this goal:

- Keep the VG shared and visible to all cluster hosts.
- Activate VM LVs exclusively on the host currently using them.
- Leave VM migration coordination to an external control plane.

## Prerequisites and repository setup

```bash
dnf install rpm-build
dnf config-manager --enable ol9_codeready_builder
dnf makecache
```

- Installs `rpm-build` so `rpmbuild` and the RPM toolchain are available for compiling the SRPM.
- Enables `ol9_codeready_builder` so required `-devel` and other build dependencies can be resolved.
- Refreshes DNF metadata so dependency lookups use current repository data and avoid stale package errors.

## Download and install source RPM

```bash
dnf download --source libvirt-daemon-driver-storage-logical-9.0.0-10.3.0.1.el9_2.x86_64
rpm -ivh libvirt-9.0.0-10.3.0.1.el9_2.src.rpm
```

- Downloads the matching libvirt source RPM so you rebuild from the official packaging baseline.
- Installs the SRPM into `~/rpmbuild/` (including `SPECS/` and `SOURCES/`) so the standard `rpmbuild` workflow can be used.

## Enter build tree and install build dependencies

```bash
cd rpmbuild
rpmbuild -bb SPECS/libvirt.spec 2>&1 | grep "is needed" | awk '{print $1}' | tr "\n" " " | xargs dnf install -y
```

- Triggers a build attempt, extracts missing package names from errors, and installs them automatically.
- This quickly closes libvirt’s large dependency chain so the next real build can proceed with fewer manual iterations.

> Tip: This is a practical bootstrap trick. In stricter build environments, you may prefer explicit `dnf builddep` workflows.

## Add your custom patch to SOURCES

Copy your patch file into `~/rpmbuild/SOURCES/`.

Important: the patch header must match proper `git format-patch` style, for example:

```text
From b747ecf4a0bf14fa00bf3afc375fc3b2e61c523c Mon Sep 17 00:00:00 2001
From: Cloud ABC <blog@cloudabc.eu>
Date: Sat, 28 Feb 2026 14:55:22 +0100
Subject: [PATCH] CLVM: shared VG and exclusive LVs

---
 src/storage/storage_backend_logical.c | 72 ++++++++++++++++++++++++++-
 1 file changed, 71 insertions(+), 1 deletion(-)

diff --git a/src/storage/storage_backend_logical.c b/src/storage/storage_backend_logical.c
index 6acbc37f18..2de54b4574 100644
--- a/src/storage/storage_backend_logical.c
+++ b/src/storage/storage_backend_logical.c
@@ -41,15 +41,74 @@
 VIR_LOG_INIT("storage.storage_backend_logical");
 
 
+/* Add a cached cluster flag to pool struct */
+typedef struct _LogicalPoolPrivate {
+    bool vg_clustered;
+    bool vg_clustered_valid;
+} LogicalPoolPrivate;
+
+/* Helper: detect if VG uses cluster lock (sanlock/dlm) */
+static bool
+virStorageBackendLogicalVGIsClusteredCached(LogicalPoolPrivate *priv,
+                                            const char *vgname)
+{
+    g_autoptr(virCommand) cmd = NULL;
+    g_autofree char *output = NULL;
+
+    if (priv->vg_clustered_valid)
+        return priv->vg_clustered;
+
+    cmd = virCommandNewArgList("vgs",
+                               "--noheadings",
+                               "--readonly",
+                               "-o", "locktype",
+                               vgname,
+                               NULL);
+    virCommandSetOutputBuffer(cmd, &output);
+
+    if (virCommandRun(cmd, NULL) < 0 || !output) {
+        priv->vg_clustered = false;
+        priv->vg_clustered_valid = true;
+        return false;
+    }
+
+    g_strstrip(output);
+    priv->vg_clustered = (STREQ(output, "sanlock") || STREQ(output, "dlm"));
+    priv->vg_clustered_valid = true;
+    return priv->vg_clustered;
+}
+
+/* LV activation flag for VM disks */
+static const char *
+virStorageBackendLogicalGetLVActivateFlag(LogicalPoolPrivate *priv,
+                                          const char *vgname)
+{
+    if (virStorageBackendLogicalVGIsClusteredCached(priv, vgname))
+        return "-aey";   /* exclusive activation for VM disks */
+    else
+        return "-ay";    /* default for non-clustered VGs */
+}
+
 static int
 virStorageBackendLogicalSetActive(virStoragePoolObj *pool,
                                   bool on)
 {
     virStoragePoolDef *def = virStoragePoolObjGetDef(pool);
     g_autoptr(virCommand) cmd = NULL;
+    LogicalPoolPrivate priv = { 0 };
+    const char *activateFlag;
     int ret;
 
-    cmd = virStorageBackendLogicalChangeCmd(VGCHANGE, def, on);
+    if (on) {
+        if (virStorageBackendLogicalVGIsClusteredCached(&priv, def->source.name))
+            activateFlag = "-asy";
+        else
+            activateFlag = "-aly";
+    } else {
+        activateFlag = "-aln";
+    }
+
+    cmd = virCommandNewArgList(VGCHANGE, activateFlag, def->source.name, NULL);
 
     virObjectUnlock(pool);
     ret = virCommandRun(cmd, NULL);
@@ -860,7 +919,10 @@ virStorageBackendLogicalCreateVol(virStoragePoolObj *pool,
                                   virStorageVolDef *vol)
 {
     virStoragePoolDef *def = virStoragePoolObjGetDef(pool);
+    g_autoptr(virCommand) lvchange_cmd = NULL;
     virErrorPtr err;
+    LogicalPoolPrivate priv = { 0 };
+    const char *lvActivateFlag;
     struct stat sb;
     VIR_AUTOCLOSE fd = -1;
 
@@ -872,6 +934,14 @@ virStorageBackendLogicalCreateVol(virStoragePoolObj *pool,
     if (virStorageBackendLogicalLVCreate(vol, def) < 0)
         return -1;
 
+    lvActivateFlag = virStorageBackendLogicalGetLVActivateFlag(&priv, def->source.name);
+    lvchange_cmd = virCommandNewArgList(LVCHANGE,
+                                        lvActivateFlag,
+                                        vol->target.path,
+                                        NULL);
+    if (virCommandRun(lvchange_cmd, NULL) < 0)
+        goto error;
+
     if (vol->target.encryption &&
         virStorageBackendCreateVolUsingQemuImg(pool, vol, NULL, 0) < 0)
         goto error;
-- 
2.43.0


```

- Places the patch in `SOURCES/` where RPM spec processing expects it.
- Keeps `git format-patch` style metadata so `%patch` application is reliable and the change remains traceable.

## Reference patch in the spec file

Edit `~/rpmbuild/SPECS/libvirt.spec` and add a patch declaration, for example:

```spec
Patch1003: clvm.patch
```

Then ensure the patch is applied in the `%prep` section (pattern depends on the existing spec style).

- Declares and applies the patch in `libvirt.spec` so it becomes part of the RPM build lifecycle.
- This is required because files in `SOURCES/` are ignored unless referenced, and it keeps future rebuilds reproducible.

## Optional: adjust Release field

Example change:

```spec
Release: 10.3.0.1%{?dist}%{?extra_release}
# to
Release: 10.3.0.1.el9_2
```

- Changes the RPM `Release` tag to match your internal naming/versioning convention.
- This helps distinguish custom rebuilds and reduces ambiguity when multiple variants exist.

## Build the RPM

```bash
rpmbuild -bb SPECS/libvirt.spec
```

- Runs a binary RPM build (`-bb`) from the spec and sources.
- This produces the installable package artifact under `~/rpmbuild/RPMS/x86_64/`.

Build options reference: https://linux.die.net/man/8/rpmbuild

If tests fail during build,

```bash
rpmbuild -bb SPECS/libvirt.spec --nocheck
```

- Uses `--nocheck` to skip the `%check` phase when tests block the build.
- This is useful for environment-specific failures unrelated to your patch so you can still create a package for controlled validation.

> Caution: skipping tests reduces build-time validation confidence. Prefer running checks when possible.

## Install rebuilt package

```bash
rpm -Uvh RPMS/x86_64/libvirt-daemon-driver-storage-logical-9.0.0-10.3.0.1.el9_2.x86_64.rpm --reinstall
```

Example output:

```text
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:libvirt-daemon-driver-storage-log################################# [ 50%]
Cleaning up / removing...
   2:libvirt-daemon-driver-storage-log################################# [100%]
```

- Installs your rebuilt RPM with upgrade/reinstall behavior.
- This deploys the patched logical storage driver, and `--reinstall` supports intentional same-NEVRA replacement.

## Restart libvirt service

```bash
systemctl restart libvirtd
```

- Restarts `libvirtd` so the runtime loads updated binaries and libraries.
- This is required for your CLVM + sanlock package changes to actually take effect.

## Quick verification checklist

- Confirm installed package version with `rpm -qa | grep libvirt-daemon-driver-storage-logical`.
- Inspect libvirt logs after restart: `journalctl -u libvirtd -b`.
- Validate the logical storage workflow that depends on CLVM/sanlock behavior.
