---
layout: post
title: Ceph - Adopt Ceph-ansible Cluster by Cephadm
toc: true
categories: [Ceph]
---

Cephadm is is the new official Ceph installer, which is more recommended. If you have a ceph cluster deployed by ceph-ansible, and wish to convert it to Cephadm-managed cluster, ceph-ansible provides to a playbook `cephadm-adopt.yml` to adopt the cluster by cephadm.

<!--more-->

This is a follow-up of [Ceph - Easy Deployment by ceph-ansible]({{ site.baseurl }}{% post_url 2024-02-21-Ceph-Deploy-By-Ansible %})

[Comparison between Ceph Ansible and Cephadm](https://www.ibm.com/docs/en/storage-ceph/7?topic=installing-comparison-between-ceph-ansible-cephadm) explains the difference between Ceph-ansible and Cephadm. Cephadm is is the new official Ceph installer, which is more recommended. 

[Ceph Storage with CloudStack on Ubuntu/KVM](https://rohityadav.cloud/blog/ceph/) introduces how to deploy ceph cluster using cephadm step by step. 

## Run playbook cephadm-adopt.yml

The following command can easily convert a ceph-ansible cluster to cephadm. However, before running the command, please read the section `Fix Issue: [Errno -2] Name or service not known`.

```
$ ansible-playbook infrastructure-playbooks/cephadm-adopt.yml -i ceph-hosts 
Are you sure you want to adopt the cluster by cephadm ? [no]: yes

...

PLAY RECAP ******************************************************************************************************************************************************************
ceph-1                     : ok=88   changed=23   unreachable=0    failed=1    skipped=30   rescued=0    ignored=0   
ceph-2                     : ok=43   changed=6    unreachable=0    failed=0    skipped=25   rescued=0    ignored=0   
ceph-3                     : ok=43   changed=6    unreachable=0    failed=0    skipped=25   rescued=0    ignored=0   
ceph-4                     : ok=37   changed=4    unreachable=0    failed=0    skipped=24   rescued=0    ignored=0   
localhost                  : ok=0    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
```

## Fix Issue: [Errno -2] Name or service not known

It stopped at the task with error
```
TASK [Adopt prometheus daemon] **********************************************************************************************************************************************
fatal: [ceph-1]: FAILED! => changed=true 
  cmd:
  - cephadm
  - --docker
  - --image
  - docker.io/prom/prometheus:v2.7.2
  - adopt
  - --cluster
  - ceph
  - --name
  - prometheus.ceph-1
  - --style
  - legacy
  - --skip-pull
  - --skip-firewalld
  delta: '0:00:01.745507'
  end: '2024-02-21 19:32:41.714614'
  rc: 1
  start: '2024-02-21 19:32:39.969107'
  stderr: |-
    Traceback (most recent call last):
      File "/usr/lib/python3.10/runpy.py", line 196, in _run_module_as_main
        return _run_code(code, main_globals, None,
      File "/usr/lib/python3.10/runpy.py", line 86, in _run_code
        exec(code, run_globals)
      File "/tmp/tmpm6j3p97n.cephadm.build/__main__.py", line 10700, in <module>
      File "/tmp/tmpm6j3p97n.cephadm.build/__main__.py", line 10688, in main
      File "/tmp/tmpm6j3p97n.cephadm.build/__main__.py", line 2495, in _default_image
      File "/tmp/tmpm6j3p97n.cephadm.build/__main__.py", line 7464, in command_adopt
      File "/tmp/tmpm6j3p97n.cephadm.build/__main__.py", line 7732, in command_adopt_prometheus
      File "/tmp/tmpm6j3p97n.cephadm.build/__main__.py", line 3689, in get_container
      File "/tmp/tmpm6j3p97n.cephadm.build/__main__.py", line 3013, in get_daemon_args
      File "/tmp/tmpm6j3p97n.cephadm.build/__main__.py", line 2291, in get_ip_addresses
      File "/usr/lib/python3.10/socket.py", line 955, in getaddrinfo
        for res in _socket.getaddrinfo(host, port, family, type, proto, flags):
    socket.gaierror: [Errno -2] Name or service not known
  stderr_lines: <omitted>
  stdout: ''
  stdout_lines: <omitted>
```

It is because the hostname is not resolved
```
root@ceph-1:~# ping ceph-1
ping: ceph-1: Name or service not known
```

Fixed the issue by
```
$ ansible-playbook update_etc_hosts.yaml -i ceph-hosts
```

## Result and verification

The command finally succeeded with retuls below.
```
TASK [Inform users about cephadm] ***************************************************
ok: [ceph-1] => 
  msg: |-
    This Ceph cluster is now managed by cephadm. Any new changes to the
    cluster need to be achieved by using the cephadm CLI and you don't
    need to use ceph-ansible playbooks anymore.

PLAY RECAP ******************************************************************************************************************************************************************
ceph-1                     : ok=109  changed=33   unreachable=0    failed=0    skipped=30   rescued=0    ignored=0   
ceph-2                     : ok=67   changed=27   unreachable=0    failed=0    skipped=26   rescued=0    ignored=0   
ceph-3                     : ok=67   changed=27   unreachable=0    failed=0    skipped=26   rescued=0    ignored=0   
ceph-4                     : ok=39   changed=6    unreachable=0    failed=0    skipped=24   rescued=0    ignored=0   
localhost                  : ok=0    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
```

Everything looks ok when check the status of ceph cluster on one of the ceph monitors.
```
root@ceph-1:~# cephadm shell ceph -s
Inferring fsid 121f9c10-d958-4de4-b74c-14aa3a68cad8
Inferring config /var/lib/ceph/121f9c10-d958-4de4-b74c-14aa3a68cad8/mon.ceph-1/config
Using ceph image with id '624f4607a91f' and tag '<none>' created on 2024-02-14 00:38:28 +0000 UTC
quay.io/ceph/daemon-base@sha256:15d0ee67691c1f65b7ae81fd20c672d1e800c5572b8bc768ebb905fc53cbdefa
  cluster:
    id:     121f9c10-d958-4de4-b74c-14aa3a68cad8
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            mons ceph-1,ceph-2,ceph-3 are low on available space
            all OSDs are running squid or later but require_osd_release < squid
 
  services:
    mon: 3 daemons, quorum ceph-3,ceph-2,ceph-1 (age 16m)
    mgr: ceph-3(active, since 15m), standbys: ceph-1, ceph-2
    osd: 4 osds: 4 up (since 13m), 4 in (since 9h)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   110 MiB used, 400 GiB / 400 GiB avail
    pgs:     1 active+clean
```

## Fix issue: "mons are allowing insecure global_id reclaim"

```
ceph config set mon auth_allow_insecure_global_id_reclaim false
```

## Fix issue: "all OSDs are running squid or later but require_osd_release < squid"

```
ceph osd require-osd-release squid
```
