---
layout: post
title: CloudStack HowTo - Create Ubuntu Cloud images for VMware and XenServer testing
toc: true
categories: [CloudStack]
---

Ubuntu Minimal Cloud Images are official Ubuntu images and are pre-installed disk images that have been customized by Ubuntu engineering to have a small runtime footprint in order to increase workload density in environments where humans are not expected to log in.

This article introduces how to create Ubuntu cloud images for testing on VMware and XenServer, other than KVM.
<!--more-->

## Ubuntu Minimal Cloud Images

There are two types of cloud images built by Ubuntu

- Releases

For Ubuntu 24.04 (noble), the images are located at https://cloud-images.ubuntu.com/minimal/releases/noble/release/

You can find images for all releases on https://cloud-images.ubuntu.com/minimal/releases/

- Daily Builds

For Ubuntu 24.04 (noble), the images are located at https://cloud-images.ubuntu.com/minimal/daily/noble/current/

You can find all daily builds images on https://cloud-images.ubuntu.com/minimal/daily/

You can download the qcow2 cloud image for latest release of Ubuntu 24.04 by
```
# amd64 (x86_64)
wget https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img

# arm64
wget https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-arm64.img
```

## Create Ubuntu Cloud images for XenServer

This can be done by qemu-img

```
# Convert to vhd
qemu-img convert -f qcow2 -O vpc ubuntu-24.04-minimal-cloudimg-amd64.img ubuntu-24.04-minimal-cloudimg-amd64.vhd
```

## Create Ubuntu Cloud images for VMware

For Ubuntu 22.04 (jammy), Ubuntu has built cloud images in ova format which can be used on VMware. However, there is no such images for Ubuntu 24.04 (noble).

```
# Convert to vmdk
qemu-img convert -f qcow2 -O vmdk ubuntu-24.04-minimal-cloudimg-amd64.img ubuntu-24.04-minimal-cloudimg-amd64.vmdk
```

However, `vmdk` is not ready for use in Apache CloudStack, so we need to convert to `ova` format. A configuration file is required for the convension. Create a file `ubuntu-24.04-minimal-cloudimg-amd64.vmx` with content below.

```
.encoding = "UTF-8"
displayname = "ubuntu-24.04-minimal-cloudimg-amd64"
guestos = "ubuntu-64"
virtualhw.version = "10"
config.version = "8"
numvcpus = "2"
cpuid.coresPerSocket = "1"
memsize = "1024"
pciBridge0.present = "TRUE"
pciBridge4.present = "TRUE"
pciBridge4.virtualDev = "pcieRootPort"
pciBridge4.functions = "8"
pciBridge5.present = "TRUE"
pciBridge5.virtualDev = "pcieRootPort"
pciBridge5.functions = "8"
pciBridge6.present = "TRUE"
pciBridge6.virtualDev = "pcieRootPort"
pciBridge6.functions = "8"
pciBridge7.present = "TRUE"
pciBridge7.virtualDev = "pcieRootPort"
pciBridge7.functions = "8"
vmci0.present = "TRUE"
floppy0.present = "TRUE"
floppy0.fileType = "device"
floppy0.autodetect = "FALSE"
floppy0.startConnected = "FALSE"
floppy0.clientDevice = "TRUE"
floppy0.allowguestconnectioncontrol = "true"
ide1:0.clientDevice = "TRUE"
ide1:0.present = "TRUE"
ide1:0.deviceType = "cdrom-raw"
ide1:0.autodetect = "TRUE"
ide1:0.startConnected = "FALSE"
ide1:0.exclusive = "false"
ide1:0.allowguestconnectioncontrol = "true"
svga.autodetect = "false"
mks.enable3d = "false"
mks.use3dRenderer = "automatic"
svga.vramSize = "4194304"
scsi0:0.present = "TRUE"
scsi0:0.deviceType = "disk"
scsi0:0.fileName = "ubuntu-24.04-minimal-cloudimg-amd64.vmdk"
scsi0:0.allowguestconnectioncontrol = "false"
scsi0:0.writeThrough = "false"
scsi0:0.mode = "persistent"
scsi0.virtualDev = "pvscsi"
scsi0.present = "TRUE"
vmci0.unrestricted = "false"
serial0.startConnected = "FALSE"
serial0.present = "TRUE"
serial0.autodetect = "FALSE"
serial0.fileName = "Serial Port 3"
serial0.yieldOnMsrRead = "false"
serial0.allowguestconnectioncontrol = "true"
ethernet0.present = "TRUE"
ethernet0.virtualDev = "vmxnet3"
ethernet0.connectionType = "bridged"
ethernet0.startConnected = "TRUE"
ethernet0.addressType = "generated"
ethernet0.wakeonpcktrcv = "true"
ethernet0.allowguestconnectioncontrol = "true"
vcpu.hotadd = "false"
vcpu.hotremove = "false"
mem.hotadd = "false"
firmware = "bios"
tools.syncTime = "false"
toolscripts.afterpoweron = "true"
toolscripts.afterresume = "true"
toolscripts.beforepoweroff = "true"
toolscripts.beforesuspend = "true"
tools.upgrade.policy = "manual"
powerType.powerOff = "preset"
powerType.reset = "preset"
powerType.suspend = "preset"
vhv.enable = "false"
chipset.onlineStandby = "FALSE"
```

(The vmx is extracted from the .ova cloud image for Ubuntu 22.04)

We need a program `ovftool` for the next convension. We can download it from VMware website for free on https://developer.broadcom.com/tools/open-virtualization-format-ovf-tool/latest .

Now run the command below
```
ovftool ubuntu-24.04-minimal-cloudimg-amd64.vmx ubuntu-24.04-minimal-cloudimg-amd64.ova
```

Now we can register a VMware template with the generated `ubuntu-24.04-minimal-cloudimg-amd64.ova` file.

