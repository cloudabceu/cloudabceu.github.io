---
layout: post
title: CloudStack - Create Windows 11 VM instance on Ubuntu (Update 2025.03)
toc: true
categories: [CloudStack]
---

In the article [CloudStack - Create Windows 11 VM instance on Ubuntu 22.04]({{ "/cloudstack/2024/02/18/CloudStack-Create-Windows-11-on-Ubuntu/" | absolute_url }}) I have introduced how to create a Windows 11 VM instance on Ubuntu 22.04.

In the past year, there are some related changes. This article introduces what has been changed and what we need to update. Here we go !

<!--more-->

## Apache CloudStack no longer supports VM settings for extraconfig

Due to critical vulnerability [CloudStack - Create Windows 11 VM instance on Ubuntu (Update 2025.03)](https://www.cve.org/CVERecord?id=CVE-2024-29008), it is no longer possible to add VM setting for VM extraconfig via GUI.

Users will get an error message `It is not allowed to add setting for extraconfig. Please update VirtualMachine with extraconfig parameter.`

![Add VM setting for extra config]({{ "/images/2025-03-05_17-11-add-vm-setting-extra-config.png" | absolute_url }}){:width="100%"}{:.glightbox}

Therefore, users have to use [cloudmonkey](https://github.com/apache/cloudstack-cloudmonkey/releases) to update virtual machine with extra config

```
(localcloud) 🐱 > update virtualmachine id=5aae3c99-5188-4731-824a-cc779ed85343 
extraconfig="<devices><tpm model='tpm-crb'> <backend type='emulator' version='2.0'/> </tpm></devices>"
```


## Options for vTPM

There are two vTPM models on KVM
- `tpm-tis`: TPM Interface Specification (TIS)
- `tpm-crb`: Command-Response Buffer (CRB)

There are two TPM versions
- `1.2`: supported with TIS
- `2.0`: supported with both TIS and CRB

## Windows 11 24H2 support

Windows 24H2 requires support for the Population Count (POPCNT) and Streaming SIMD Extensions 4.2 (SSE 4.2) instruction sets.

Therefore, The Windows 11 24H2 virtual machine cannot run with the default CPU model `qemu64`. The supported Intel processors can be found at [Windows 11 version 24H2 supported Intel processors](https://learn.microsoft.com/en-us/windows-hardware/design/minimum/supported/windows-11-24h2-supported-intel-processors)

To create Windows 11 24H2 virtual machines, CloudStack users need to set the CPU mode in the cloudstack agent configuration file (agent.properties).

```
guest.cpu.mode=host-passthrough
```

Please refer to [Configure CPU model for KVM guest](https://docs.cloudstack.apache.org/en/latest/installguide/hypervisor/kvm.html#configure-cpu-model-for-kvm-guest-optional).

