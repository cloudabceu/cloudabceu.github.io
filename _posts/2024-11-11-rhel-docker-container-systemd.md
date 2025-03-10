---
layout: post
title: Support systemd services in CentOS/Rocky/Alma/OracleLinux Docker container
toc: true
categories: [Linux, Container]
---

Last post has introduced how to support systemd services in Ubuntu and Debian containers. The same issue also exists in CentOS/Redhat and the variants, including Rocky Linux, AlmaLinux and OracleLinux.
This article explains how to support systemd services in the CentOS/Rocky/Alma/OracleLinux docker containers.

<!--more-->

## Problem description

Please refer to the "Problem description" section in [Support systemd services in Ubuntu/Debian Docker container]({{ "/linux/container/2024/10/27/ubuntu-docker-container-systemd/#problem-description" | absolute_url }})

## Solve the problem

To solve the problem, create a Dockerfile like below
```
FROM oraclelinux:9

ENV container docker

RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == \
systemd-tmpfiles-setup.service ] || rm -f $i; done); \
rm -f /lib/systemd/system/multi-user.target.wants/*;\
rm -f /etc/systemd/system/*.wants/*;\
rm -f /lib/systemd/system/local-fs.target.wants/*; \
rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
rm -f /lib/systemd/system/basic.target.wants/*;\
rm -f /lib/systemd/system/anaconda.target.wants/*;

RUN dnf install -y openssh-server

VOLUME [ "/sys/fs/cgroup" ]

CMD ["/usr/sbin/init"]
```

You may notice that the Dockerfile has much differences with the dockerfile for Ubuntu/Debian.
Please refer to [rockylinux/rockylinux - Docker Image | Docker Hub](https://hub.docker.com/r/rockylinux/rockylinux)

## Verification

Create a docker image from the Dockerfile.
```
# docker build -t oracle9-systemd -f Dockerfile.oracle9.systemd .
```

Create a docker container from the image
```
docker run -it -d \
    --privileged \
    --volume=/sys/fs/cgroup:/sys/fs/cgroup:rw \
    --cgroupns=host \
    --name oracle9-test \
    --hostname=oracle9-test \
    oracle9-systemd
```

Now go into the bash shell and verify it.
```
# docker exec -it oracle9-test bash
[root@oracle9-test /]# 
[root@oracle9-test /]# systemctl restart sshd
[root@oracle9-test /]# 
[root@oracle9-test /]# systemctl status sshd
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; disabled; vendor preset: enabled)
   Active: active (running) since Mon 2024-11-11 09:44:02 UTC; 1s ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 289824 (sshd)
    Tasks: 1 (limit: 49840)
   Memory: 3.2M
   CGroup: /system.slice/docker-3bf2e1d51d03f9ab419f0f71c5221bbd482af4efad26ad2a2e62803d9a98a81f.scope/system.slice/sshd.service
           └─289824 /usr/sbin/sshd -D -oCiphers=aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr,aes256-cbc,aes128-gcm@openssh.com,aes128-ctr,aes128-cbc -oMACs=hmac-sha2-256-etm@openssh.com,hmac-sha1-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha2-256,hmac-sha1,umac-128@openssh.com,hmac-sha2-512 -oGSSAPIKexAlgorithms=gss-curve25519-sha256-,gss-nistp256-sha256-,gss-group14-sha256-,gss-group16-sha512-,gss-gex-sha1-,gss-group14-sha1- -oKexAlgorithms=curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256,diffie-hellman-group14-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group-exchange-sha1,diffie-hellman-group14-sha1 -oHostKeyAlgorithms=ecdsa-sha2-nistp256,ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp384,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp521,ecdsa-sha2-nistp521-cert-v01@openssh.com,ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-256,rsa-sha2-256-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-512-cert-v01@openssh.com,ssh-rsa,ssh-rsa-cert-v01@openssh.com -oPubkeyAcceptedKeyTypes=ecdsa-sha2-nistp256,ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp384,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp521,ecdsa-sha2-nistp521-cert-v01@openssh.com,ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-256,rsa-sha2-256-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-512-cert-v01@openssh.com,ssh-rsa,ssh-rsa-cert-v01@openssh.com -oCASignatureAlgorithms=ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,ssh-ed25519,rsa-sha2-256,rsa-sha2-512,ssh-rsa

Nov 11 09:44:02 oracle9-test systemd[1]: Starting OpenSSH server daemon...
Nov 11 09:44:02 oracle9-test sshd[289824]: Server listening on 0.0.0.0 port 22.
Nov 11 09:44:02 oracle9-test sshd[289824]: Server listening on :: port 22.
Nov 11 09:44:02 oracle9-test systemd[1]: Started OpenSSH server daemon.
```

It looks good.

## Notes

1. `CMD ["/usr/sbin/init"]` in the Dockerfile makes the system PID to 1.

2. Same as Ubuntu/Debian containers, we must specify `--privileged` when create the docker container.

3. Two extra arguments are required when create the docker container
```
    --volume=/sys/fs/cgroup:/sys/fs/cgroup:rw \
    --cgroupns=host \
```

4. The name of sshd service in the docker container is `sshd`, not the same as Ubuntu/Debian container (which uses `ssh`). To set the default root password and enable ssh,
```
echo "root:password" |chpasswd
echo "PermitRootLogin yes" >>/etc/ssh/sshd_config
systemctl enable ssh || systemctl enable sshd
systemctl restart ssh || systemctl restart sshd
```

5. This only works for RHEL/RockyLinux/AlmaLinux/OracleLinux, not for Ubuntu/Debian.
To support it, please refer to [Support systemd services in Ubuntu/Debian Docker container]({{ "/linux/container/2024/10/27/ubuntu-docker-container-systemd/" | absolute_url }})

