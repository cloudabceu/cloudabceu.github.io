---
layout: post
title: CloudStack - How to add Ceph 20 (Tentacle) as Primary Storage on KVM
toc: true
categories: [CloudStack, KVM, Ceph]
---

Ceph 20 ("Tentacle") is the newest Ceph major release, and if you try to add it as RBD primary storage to a CloudStack KVM cluster today, you will very likely hit two separate failures before you get a working pool - neither of which is a CloudStack bug. This post walks through both, and how to get a Tentacle cluster working as CloudStack primary storage today, without downgrading Ceph.

This is a follow-up to [Ceph - Deploy Ceph 20 (tentacle) with cephadm-ansible]({{ site.baseurl }}{% post_url 2026-09-06-Ceph-Deploy-Tentacle-By-Cephadm-Ansible %}) - if you don't have a Tentacle cluster yet, start there.

<!--more-->

## Symptom

Adding the RBD pool as primary storage in CloudStack fails, and the KVM agent log on the host shows:

```
DEBUG [kvm.storage.LibvirtStorageAdaptor] Attempting to create storage pool 5754fb0c-853e-3097-97b1-95a51e1fb080
DEBUG [kvm.storage.LibvirtStorageAdaptor] <pool type='rbd'>
<name>5754fb0c-853e-3097-97b1-95a51e1fb080</name>
<uuid>5754fb0c-853e-3097-97b1-95a51e1fb080</uuid>
<source>
<host name='10.1.33.97'/>
<name>cloudstack</name>
<auth username='cloudstack' type='ceph'>
<secret uuid='5754fb0c-853e-3097-97b1-95a51e1fb080'/>
</auth>
</source>
</pool>

ERROR [kvm.storage.LibvirtStorageAdaptor] Failed to create RBD storage pool: org.libvirt.LibvirtException: failed to connect to the RADOS monitor on: 10.1.33.97,: No such file or directory
```

or, after fixing the first problem, this one instead:

```
ERROR [kvm.storage.LibvirtStorageAdaptor] Failed to create RBD storage pool: org.libvirt.LibvirtException: internal error: failed to set RADOS option: auth_supported
```

Both are environment/library issues below CloudStack, not something fixed by CloudStack config. Here's what's going on and how to fix each one.

## Problem 1: the KVM host's Ceph client is too old for Tentacle

CloudStack's KVM agent doesn't talk RBD itself - it hands the mon host, pool name, and cephx secret to libvirt, which opens the connection using whatever `librados`/`librbd` is installed on the hypervisor. On stock Rocky Linux 9 / RHEL 9, that's the AppStream package:

```
librados2-16.2.4-5.el9.x86_64    # Ceph Pacific (2021)
librbd1-16.2.4-5.el9.x86_64
```

Ceph Pacific talking to a Tentacle (20.2.4) monitor is a 4-major-release gap, well outside Ceph's supported client/server compatibility window, and shows up as the generic `failed to connect to the RADOS monitor ... No such file or directory` - not a helpful version-mismatch error.

**Fix:** install a current `librados2`/`librbd1` from `download.ceph.com` on every KVM host. On Rocky/RHEL 9 you also need the CRB repo enabled, for `lttng-ust` and `libpmemobj`:

```bash
dnf config-manager --set-enabled crb

rpm --import https://download.ceph.com/keys/release.asc
cat <<'EOF' > /etc/yum.repos.d/ceph.repo
[ceph]
name=Ceph packages
baseurl=https://download.ceph.com/rpm-tentacle/el9/x86_64/
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc
EOF

dnf install -y librados2 librbd1
systemctl restart libvirtd
```

(Reef or Squid client packages also work fine against a Tentacle mon - Ceph supports N-2 - but matching the mon's release is the safest choice, and it's what I used here.)

With just this fix applied, `virsh pool-create` against the RBD pool XML gets further, but now fails with a *different* error - which is problem 2.

## Problem 2: libvirt still sets a config option Ceph removed

Ceph Tentacle (`librados` ≥ 19.2.6/20.2.4) fully removed the long-deprecated `auth_supported` config option. libvirt's RBD storage backend (`src/storage/storage_backend_rbd.c`, function `virStorageBackendRBDOpenRADOSConn`) still sets it unconditionally when opening the pool connection, and treats a failed `rados_conf_set()` as fatal:

```c
if (virStorageBackendRBDRADOSConfSet(ptr->cluster,
                                     "auth_supported", "cephx") < 0)
    goto cleanup;
```

I checked this against upstream libvirt `master` and the latest release (v12.7.0, newer than what ships on Rocky 9 today) - both still have this exact code, unfixed. It's tracked upstream as [libvirt#918](https://gitlab.com/libvirt/libvirt/-/work_items/918), open at the time of writing. Proxmox hit the identical wall from their own RBD storage plugin ([bugzilla #7961](https://bugzilla.proxmox.com/show_bug.cgi?id=7961)) and worked around it by restoring the option in their own downstream Ceph build - not something available if you're running upstream Ceph packages.

There's also an [apache/cloudstack#13991](https://github.com/apache/cloudstack/pull/13991) PR that swaps `auth_supported` for `auth_client_required` in CloudStack's own RBD connection string - but that string is only used for `qemu-img`-driven operations (template seeding, volume/snapshot copies), in `KVMPhysicalDisk.RBDStringBuilder()`. It never touches `LibvirtStorageAdaptor.createRBDStoragePool()`, which is the code path that actually fails above - that one goes straight to libvirt's XML API, and the failing `auth_supported` call happens entirely inside libvirt's compiled C code. No string CloudStack builds can reach it. See the discussion on [apache/cloudstack#13989](https://github.com/apache/cloudstack/issues/13989) for the full thread - the PR is still worth having (it fixes the qemu-img side), but it doesn't fix storage pool creation on its own.

### Workaround: an `LD_PRELOAD` shim for `libvirtd`

Until libvirt#918 lands and gets packaged, the option I'd reach for on the hypervisors themselves is a tiny `LD_PRELOAD` shim that intercepts the one deprecated `rados_conf_set()` call libvirt makes and rewrites it to `auth_client_required` (the same replacement suggested in libvirt#918). It's scoped to the `libvirtd` service only via a systemd drop-in, so nothing else on the host is affected.

`rados_auth_shim.c`:

```c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <string.h>
#include <stdio.h>

typedef int (*rados_conf_set_fn)(void *cluster, const char *option, const char *value);

int rados_conf_set(void *cluster, const char *option, const char *value)
{
    static rados_conf_set_fn real_rados_conf_set = NULL;
    if (!real_rados_conf_set) {
        real_rados_conf_set = (rados_conf_set_fn)dlsym(RTLD_NEXT, "rados_conf_set");
    }

    if (option != NULL && strcmp(option, "auth_supported") == 0) {
        fprintf(stderr, "[rados_auth_shim] rewriting deprecated option "
                "'auth_supported=%s' -> 'auth_client_required=%s'\n",
                value ? value : "(null)", value ? value : "(null)");
        option = "auth_client_required";
    }

    return real_rados_conf_set(cluster, option, value);
}
```

Build and install it on each KVM host (needs a C compiler - `gcc`/`glibc-devel`):

```bash
gcc -Wall -fPIC -shared -o /usr/local/lib/rados_auth_shim.so rados_auth_shim.c -ldl

mkdir -p /etc/systemd/system/libvirtd.service.d
cat > /etc/systemd/system/libvirtd.service.d/rados-auth-shim.conf <<'EOF'
[Service]
Environment=LD_PRELOAD=/usr/local/lib/rados_auth_shim.so
EOF

systemctl daemon-reload
systemctl restart libvirtd
```

The `Environment=` line only applies to the `libvirtd` unit because of the drop-in, not system-wide - it won't affect `qemu`/other processes on the host. You can confirm it's active by watching the journal while a pool is (re)created:

```bash
journalctl -u libvirtd -f
# [rados_auth_shim] rewriting deprecated option 'auth_supported=cephx' -> 'auth_client_required=cephx'
```

With both fixes in place - current `librados2`/`librbd1`, and the shim loaded into `libvirtd` - the RBD storage pool comes up correctly:

```
$ virsh pool-info 5754fb0c-853e-3097-97b1-95a51e1fb080
Name:           5754fb0c-853e-3097-97b1-95a51e1fb080
UUID:           5754fb0c-853e-3097-97b1-95a51e1fb080
State:          running
Persistent:     no
Autostart:      no
Capacity:       299.99 GiB
Allocation:     0.00 B
Available:      299.91 GiB
```

...and CloudStack's own "Add primary storage" retries the same pool creation successfully once the fixes are applied on all KVM hosts.

## Adding the storage in CloudStack

Once both host-side fixes are applied, adding the pool is completely ordinary - no CloudStack-side workaround needed:

```bash
cmk create storagepool \
  name=ceph-tentacle-primary \
  zoneid=<zone-id> \
  podid=<pod-id> \
  clusterid=<cluster-id> \
  url="rbd://cloudstack:<cephx-key-base64>@10.1.33.97/cloudstack" \
  scope=cluster \
  provider=DefaultPrimary
```

or via the UI: **Infrastructure > Primary Storage > Add Primary Storage**, protocol `RBD`, server = a mon IP, path = the pool name, and the `cloudstack` cephx user's key.

## Caveats

- The `LD_PRELOAD` shim is a stopgap, not a real fix. It should come out once libvirt actually ships a fix for [#918](https://gitlab.com/libvirt/libvirt/-/work_items/918) and your distro packages it: `rm /etc/systemd/system/libvirtd.service.d/rados-auth-shim.conf /usr/local/lib/rados_auth_shim.so && systemctl daemon-reload && systemctl restart libvirtd`.
- Both fixes (client library upgrade + shim) need to be applied on **every** KVM host in the cluster, not just one.
- If you rebuild/reimage a KVM host later, remember to reapply both - they aren't part of the standard CloudStack KVM agent install.
- If you don't specifically need Ceph 20 for testing, pinning to Squid (19.2.x) or Reef (18.2.x) avoids all of this entirely - both are well-supported today and don't have the `auth_supported` problem.

## References

- [apache/cloudstack#13989](https://github.com/apache/cloudstack/issues/13989) - the original bug report and discussion thread
- [apache/cloudstack#13991](https://github.com/apache/cloudstack/pull/13991) - fixes the qemu-img side of the `auth_supported` removal
- [libvirt#918](https://gitlab.com/libvirt/libvirt/-/work_items/918) - upstream libvirt tracking issue
- [Proxmox bugzilla #7961](https://bugzilla.proxmox.com/show_bug.cgi?id=7961) - Proxmox hitting the same removal from their RBD plugin
