---
layout: post
title: Ceph - Deploy Ceph 20 (tentacle) with cephadm-ansible
toc: true
categories: [Ceph]
---

Ceph 20 (tentacle) is deployed differently from the older releases I covered before: `ceph-ansible` is retired, and the [cephadm-ansible](https://github.com/ceph/cephadm-ansible) project only ships a handful of helper playbooks (preflight, ssh key distribution, purge, ...) instead of a single `site.yml` that installs the whole cluster. The actual cluster is built with `cephadm` itself (`cephadm bootstrap` + `ceph orch ...`), and Ansible just prepares the hosts and wires the pieces together. This article walks through deploying a 3-node Ceph 20 cluster on Ubuntu with `cephadm-ansible`, and the gotchas I hit along the way.

<!--more-->

This is a follow-up of [Ceph - Easy Deployment by ceph-ansible]({{ site.baseurl }}{% post_url 2024-02-21-Ceph-Deploy-By-Ansible %}) and [Ceph - Adopt Ceph-ansible Cluster by Cephadm]({{ site.baseurl }}{% post_url 2024-02-29-Adopt-Ceph-Ansible-Cluster-by-Cephadm %}).

Let's assume 3 Ubuntu hosts (`ceph-1`, `ceph-2`, `ceph-3`) have already been installed, and there's a server which can reach all of them over ssh.

## Step 1: Clone cephadm-ansible

```
$ git clone https://github.com/ceph/cephadm-ansible.git
$ cd cephadm-ansible
$ git checkout tentacle
```

## Step 2: Create the inventory

Unlike `ceph-ansible`, `cephadm-ansible` doesn't need `[mons]`/`[osds]`/`[mgrs]` groups: the cluster topology is driven entirely by `cephadm` (bootstrap + `ceph orch host add`), not by the inventory. The only group it actually needs is `[admin]`, for the host that will run the bootstrap (and later hold the admin keyring). `[clients]` is only for hosts that are pure Ceph clients -- don't put your mon/osd nodes in it, or the preflight playbook will treat them as clients and skip installing `cephadm`/`podman`/`lvm2` on them.

```
$ cat > ceph-hosts <<EOF
ceph-1  ansible_host=10.0.33.11
ceph-2  ansible_host=10.0.33.12
ceph-3  ansible_host=10.0.33.13

[admin]
ceph-1

[all:vars]
ansible_user=root
ansible_ssh_pass=password
EOF
```

## Step 3: Run the preflight playbook

The preflight playbook configures the Ceph repository and installs prerequisites (`podman`, `lvm2`, `chronyd`, `cephadm`, ...).

```
$ ansible-playbook -i ceph-hosts cephadm-preflight.yml --extra-vars "ceph_release=tentacle"
```

Note: `ceph_release` defaults to `quincy` in the role, not `tentacle`, so it must be passed explicitly -- otherwise preflight configures the wrong (much older) Ceph repository.

Two gotchas I hit running this on Ubuntu:

- **`ceph_stable_key is undefined`**: on Ubuntu, the "Configure ceph community repository stable key" task reads the raw `ceph_stable_key` variable, which has no default of its own (only `ceph_defaults_ceph_stable_key` does). This is a bug in `cephadm-preflight.yml` -- I fixed it locally by having that task use `ceph_defaults_ceph_stable_key` instead.
- **`Failed to find required executable "apt-key"`**: that same task used the `ansible.builtin.apt_key` module, which shells out to the `apt-key` binary. Ubuntu 24.04+ dropped `apt-key` entirely, so this fails outright on newer Ubuntu releases. The fix is to replace it with a `get_url` + `gpg --dearmor` step that writes a keyring under `/etc/apt/keyrings/`, and reference it via `signed-by=` in the repo line instead of relying on the old global trusted keyring.

Also worth knowing: `download.ceph.com/debian-tentacle` only publishes packages for `jammy` (22.04) and `noble` (24.04) at the time of writing -- not yet for newer Ubuntu codenames. If `apt update` on a node fails with a 404 on the repo's `Release` file, that's why; use 22.04 or 24.04 for now.

## Step 4: Bootstrap the cluster

Run `cephadm bootstrap` on the admin node to create the first mon/mgr and enable the dashboard.

```
$ ssh ceph-1 "cephadm bootstrap --mon-ip 10.0.33.11 \
    --initial-dashboard-user admin --initial-dashboard-password password \
    --dashboard-password-noupdate --allow-fqdn-hostname"
```

## Step 5: Distribute the ssh key and add the other hosts

`cephadm-distribute-ssh-key.yml` copies cephadm's own ssh public key to every host so the admin node (via the orchestrator) can manage them.

```
$ ansible-playbook -i ceph-hosts cephadm-distribute-ssh-key.yml -e admin_node=ceph-1
```

Then add the remaining hosts to the cluster:

```
$ ssh ceph-1 "cephadm shell -- ceph orch host add ceph-2 10.0.33.12"
$ ssh ceph-1 "cephadm shell -- ceph orch host add ceph-3 10.0.33.13"
```

## Step 6: Scale mon/mgr and deploy OSDs

```
$ ssh ceph-1 "cephadm shell -- ceph orch apply mon 3"
$ ssh ceph-1 "cephadm shell -- ceph orch apply mgr 3"
$ ssh ceph-1 "cephadm shell -- ceph orch apply osd --all-available-devices"
```

## Step 7: Wait for HEALTH_OK

The cluster takes a little while to settle after the steps above (OSDs coming up, PGs peering, ...), so it's worth polling `ceph health` rather than assuming success right away.

```
$ health=""
$ retries=60
$ until [ "$health" = "HEALTH_OK" ] || [ $retries -eq 0 ]; do
      health=$(ssh ceph-1 "cephadm shell -- ceph health" 2>/dev/null)
      [ "$health" = "HEALTH_OK" ] || { sleep 10; retries=$((retries-1)); }
  done
```

Note: `cephadm shell` writes its "Inferring fsid/config" messages to stderr, not stdout, so `2>/dev/null` is enough to capture a clean status string.

If everything goes well:

```
$ ssh ceph-1 "cephadm shell -- ceph -s"
  cluster:
    id:     a8c8a79c-aa0e-11f1-8885-1e003d000a1f
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-1,ceph-2,ceph-3 (age 86m) [leader: ceph-1]
    mgr: ceph-1.mmjitw(active, since 89m), standbys: ceph-2.lgdfkb, ceph-3.bagjbg
    osd: 3 osds: 3 up (since 85m), 3 in (since 86m)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 577 KiB
    usage:   83 MiB used, 300 GiB / 300 GiB avail
    pgs:     1 active+clean
```

## Step 8: Ceph Dashboard

`cephadm` enables the dashboard by default. Unlike `ceph-ansible` (where `dashboard_protocol` is a variable you set, and can be `http`), the cephadm-managed dashboard is TLS-only -- always `https`. You can confirm the URL it's actually bound to with:

```
$ ssh ceph-1 "cephadm shell -- ceph mgr services"
{
    "dashboard": "https://10.0.33.11:8443/",
    "prometheus": "http://10.0.33.11:9283/"
}
```

Open that URL in a browser and log in with the admin user/password from Step 4.

If it's not reachable, check the obvious things first: is `cephadm shell -- ceph orch ps --daemon-type mgr` showing the mgr daemons `running`, and is something actually listening on 8443 on that host (`ss -tlnp | grep 8443`)? If both look fine but the browser still can't reach it, look outside the cluster -- a security group/firewall rule that only allows the ports you've used before (e.g. 22 and Grafana's 3000) but not 8443, or, in my case, simply having two VPNs connected at once and the wrong one being used to route to the dashboard's network.
