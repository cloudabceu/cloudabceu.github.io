---
layout: post
title: CloudStack HowTo - Patch System VMs and Virtual Routers
toc: true
categories: [CloudStack]
---

System VMs (Secondary Storage VM and Console Proxy VM) and Virtual Routers play key role in Apache CloudStack. If user want to apply a change on the system vms and VRs, they do not need to wait for next release with the fix, they can patch the VMs. This article describes the steps how to apply a patch to a system vm or a VR.

<!--more-->

## Get the files to configure System VMs and Virtual Routers

When System VMs and Virtual Routers start,  CloudStack copies some files to them via SCP. The files can be found in the folder `/usr/share/cloudstack-common/vms/` in CloudStack management server or agent (KVM hosts).

```
# ls -l /usr/share/cloudstack-common/vms/
total 103992
-rw-rw-r-- 1 debian debian 106330553 May 20 17:31 agent.zip
-rw-rw-r-- 1 debian debian    144656 May 20 17:31 cloud-scripts.tgz
-rw-rw-r-- 1 debian debian      5347 May 20 17:31 patch-sysvms.sh
```

- agent.zip: contains lots of JAR files for System VMs, few shell scripts, noVNC console files, etc.
- cloud-scripts.tgz: contains lots of shell scripts, python scripts, configuration files, and other settings.

## Update agent.zip

Unzip the agent.zip file, update the JAR file, then re-zip the file.
Please refer to the article [CloudStack HowTo - Patch management servers and hosts]({{ "/cloudstack/2025/05/18/CloudStack-howto-patch-mgmt-servers-and-hosts/" | absolute_url }}) on how to compile the CloudStack project.

```
mkdir /tmp/agent

# Extract agent.zip
unzip agent.zip -d /tmp/agent/

# Replace JAR files or other files

# Create new zip file
cd /tmp/agent
zip /root/agent.zip.new *
```

Then copy the file and overwrite the existing `agent.zip` file on the management servers, and KVM hosts if applicable.
If you use XCP-ng or Xenserver, copy the file to /opt/xensource/packages/resources/ on the XCP-ng or Xenserver hosts.

## Update cloud-scripts.tgz

Similar as above,

```
# Extract
mkdir /tmp/cloud-scripts/
tar xvzf cloud-scripts.tgz -C /tmp/cloud-scripts/

# Apply changes

# Create new tarball
cd /tmp/cloud-scripts
tar cvzf /tmp/cloud-scripts-new.tgz *
```

Then copy the file and overwrite the existing `cloud-scripts.tgz` file on the management servers, and KVM hosts if applicable.
If you use XCP-ng or Xenserver, copy the file to /opt/xensource/packages/resources/ on the XCP-ng or Xenserver hosts.

## Patch System VMs

There are three ways to have a System VM with the patch

- Live-patch the System VM
- Stop and start the System VM
- Destroy the System VM, CloudStack will create a new one

## Patch Virtual Routers 

Similar to System VMs, there are multiple options

- Live-patch the Virtual Router
- Stop and start the virtual router
- Restart network or VPC with cleanup
