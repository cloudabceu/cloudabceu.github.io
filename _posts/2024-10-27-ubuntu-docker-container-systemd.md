---
layout: post
title: Support systemd services in Ubuntu/Debian Docker container
toc: true
categories: [Linux, Container]
---

As of now, many system and application services in linux distributions are managed by systemd. However, systemd is not supported by default in docker containers. This article explains how to support systemd services in docker containers.

<!--more-->

## Problem description

As the first step, let's create a ubuntu 24.04 container by the following command
```
# docker run --name ubuntu24-test -it -d ubuntu:24.04
f3ff816528f4128615a1cb696f4f7e1e1e4e08d10ee2e8a6ac8d3b8d801fcdfd
```
Then go into the bash shell of the docker container, install `ssh` and `systemd` packages.
```
# docker exec -it ubuntu24-test bash
root@f3ff816528f4:/# apt update && apt install -y ssh systemd
...
```

Same as normal ubuntu 24.04 system, let's start the sshd service by systemctl
```
root@f3ff816528f4:/# systemctl restart ssh 
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
```

As displayed, the ssh service cannot be started by `systemctl` as the system PID is not 1.

## Solve the problem

To solve the problem, create a Dockerfile like below
```
FROM ubuntu:24.04

ARG DEBIAN_FRONTEND=noninteractive

RUN apt update && apt install -y ssh systemd

RUN echo exit 0 > /usr/sbin/policy-rc.d

CMD ["/sbin/init"] 
```

then create a docker image from the Dockerfile, and create a docker container from the image.
```
# docker build -t ubuntu24-systemd -f Dockerfile.ubuntu24.systemd .
# docker run --name ubuntu24-test-2 -it -d  --privileged ubuntu24-systemd
```

Now go into the bash shell and verify it.
```
# docker exec -it ubuntu24-test-2 bash
root@f19944edc064:/# 
root@f19944edc064:/# 
root@f19944edc064:/# systemctl restart ssh
root@f19944edc064:/# 
root@f19944edc064:/# systemctl status ssh
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; disabled; preset: enabled)
     Active: active (running) since Sun 2024-10-27 15:38:13 UTC; 4s ago
TriggeredBy: ● ssh.socket
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 123 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 125 (sshd)
      Tasks: 1 (limit: 1401)
     Memory: 1.2M ()
        CPU: 38ms
     CGroup: /system.slice/ssh.service
             └─125 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

Oct 27 15:38:13 f19944edc064 systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
Oct 27 15:38:13 f19944edc064 sshd[125]: Server listening on :: port 22.
Oct 27 15:38:13 f19944edc064 systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
```

Brilliant, it works !

## Notes

1. `CMD ["/sbin/init"]` in the Dockerfile makes the system PID to 1.

2. Meanwhile, we must specify `--privileged` when create the docker container, otherwise the docker container will not be started successfully due to error
```
# docker logs ubuntu24-test-3
Failed to mount tmpfs (type tmpfs) on /run (MS_NOSUID|MS_NODEV|MS_STRICTATIME "mode=0755,size=20%,nr_inodes=800k"): Operation not permitted 
Failed to mount tmpfs (type tmpfs) on /run/lock (MS_NOSUID|MS_NODEV|MS_NOEXEC "mode=1777,size=5242880"): Operation not permitted
[!!!!!!] Failed to mount API filesystems.
Exiting PID 1...
```

3. `RUN echo exit 0 > /usr/sbin/policy-rc.d` overwites the file `/usr/sbin/policy-rc.d`, it exits with code 101 by default which prevents some daemons from starting or restarting during package upgrade or installation.

4. This only works for Ubuntu/Debian, not for RHEL/RockyLinux/AlmaLinux/OracleLinux.
