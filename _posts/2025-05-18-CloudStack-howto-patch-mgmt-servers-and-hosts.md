---
layout: post
title: CloudStack Howto - Patch management servers and hosts
toc: true
categories: [CloudStack]
---

Recently I was asked by CloudStack users how to apply a patch (or patches) to their environments. It is very useful when the patch fixes a critical issue for them. So, this article describes the steps how to compile Apache CloudStack project using Ubuntu 22.04 docker container, and how to apply the patch to management servers and hosts.

<!--more-->

## Create Ubuntu 22.04 container

Apache CloudStack can be compiled in multiple OS distributions. In this example I use Ubuntu 22.04 as it is easy to install the dependencies (Ubuntu 24.04 does not work).
```
docker run -d -it --name ubuntu22 ubuntu:22.04
```

Note: if you want to mount local directories, for example ~/.m2 repository or local source code directory,
```
docker run -d -it \
    --name ubuntu22 \
    -v ~/.m2:/root/.m2/ \
    -v ${CLOUDSTACK_SOURCE_DIR}:/cloudstack/ \    
    ubuntu:22.04
```

Then log into it via
```
docker exec -it ubuntu22 bash
```

## Clone the CloudStack repository

Install git and clone the repository using git commands.
```
apt update
apt install -y git wget patch
git clone https://github.com/apache/cloudstack.git -b <tag or branch name>

cd cloudstack
```

Please note, the `tag or branch name` should be same to what you are using.

## Apply the patch

Get the patch files and apply the patches
```
wget https://github.com/apache/cloudstack/pull/{PR_ID}.patch

git config --global user.email "you@example.com"
git config --global user.name "Your Name"

git am {PR_ID}.patch
```

Sometimes it does not work, for example, there are some conflicts, you need to fix the conflicts manually.

```
wget https://github.com/apache/cloudstack/pull/{PR_ID}.diff
patch -p1 < {PR_ID}.diff

## fix the conflicts manually

git status
git add <files are changed>
git commit
```

## Install build dependencies

Please note, this does not work in Ubuntu 24.04 docker container
```
echo "y" | apt build-dep .
```

If you build projects with VMware, please install other JARs as well
```
git clone https://github.com/shapeblue/cloudstack-nonoss.git nonoss && \
    cd nonoss && \
    bash -x install-non-oss.sh && \
    cd ..

rm -fr nonoss
```

## Build the CloudStack project

Please note, `-DskiptTests` will skip the unit tests, hence speed up the build.
```
mvn -P developer,systemvm -Dnoredist clean install -DskipTests
```

## Apply to management server

In many cases, It is good enough to apply the patch to management server by 
- Backup the `/usr/share/cloudstack-management/lib/cloudstack-<Version>.jar` on management server
- Copy the `client/target/cloud-client-ui-<Version>.jar` in the docker container to your management server (overwrite the file above)
- Restart cloudstack-management service

Please note, 
- the JAR needs to copy is NOT `server/target/cloud-server-<Version>.jar`
- If there are some changes with usage server, copy the JARs to `/usr/share/cloudstack-usage` (`cloud-usage-*.jar`) or `/usr/share/cloudstack-usage/lib/` (other `cloud-*.jar`)
- If there are some changes in scripts/, copy new scripts to `/usr/share/cloudstack-common/scripts/` on the management server
- If there are some changes for System VMs and VRs, copy files in `systemvm/dist/` to `/usr/share/cloudstack-common/vms/` on the management server

## Apply to hosts

To Apply the patches to KVM hosts
- Copy the JARs in the projects with changes to `/usr/share/cloudstack-agent/lib/` on the KVM host
- Restart cloudstack-agent service
- If there are some changes in scripts/, copy new scripts to `/usr/share/cloudstack-common/scripts/` on the KVM host
- If there are some changes for System VMs and VRs, copy files in `systemvm/dist/` to `/usr/share/cloudstack-common/vms/` on the KVM host

If you use XCP-ng or Xenserver, copy files in `systemvm/dist/` to `/opt/xensource/packages/resources/` on the XCP-ng or Xenserver host.

