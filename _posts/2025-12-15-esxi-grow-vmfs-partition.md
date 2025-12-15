---
layout: post
title: VMware - Extend VMFS partition on ESXi host
toc: true
categories: [VMware]
---

While working with a VMware ESXi 8.0 Update 3 testing environment, I encountered an issue where the vCenter Server Appliance (VCSA) repeatedly crashed without an obvious root cause. Attempts to power off the appliance failed with disk-related errors, and although the VM could be forcefully restarted, the problem reappeared after a short period of time.

At first glance, the ESXi host appeared to be running out of disk space, despite being provisioned with a 150 GB disk. This raised an important question: where had the space gone? As it turned out, the root cause was not excessive data growth, but rather unallocated disk space that had never been added to the VMFS datastore.

In this post, I walk through the investigation process, explain how to identify unallocated space on an ESXi host, and demonstrate how to safely extend a VMFS datastore to recover the missing capacity and prevent future outages.

<!--more-->

## Problem description

I was running a VMware ESXi 8.0 Update 3 testing environment when I encountered an issue where the vCenter Server Appliance (VCSA) unexpectedly crashed. When attempting to power off the VCSA virtual machine on the ESXi host, the operation failed with the following error:
```
[root@vc1] vim-cmd vmsvc/power.off 1
...
text = "msg.hbacommon.outofspace:There is no more space for virtual disk 'VC_12.vmdk'.
...
```

As a result, I had to manually terminate the VM process and then power the virtual machine back on:
```
[root@vc1] esxcli vm process list
VC
   World ID: 603996
   Process ID: 0
   VMX Cartel ID: 603994
   UUID: 56 4d be 1b 45 52 d6 bc-1b 4d 4b 5f 56 53 4d b6
   Display Name: VC
   Config File: /vmfs/volumes/66b0c33b-d2f3dff8-8f53-1e00f500013e/VC/VC.vmx

[root@vc1] esxcli vm process kill --type=force --world-id=603996

[root@vc1] vim-cmd vmsvc/power.on 1
Powering on VM:

[root@vc1] 
```

Although the appliance was successfully restarted, the same issue reoccurred one to two days later, indicating an underlying storage-related problem that required further investigation.

## Root cause analysis

To investigate the recurring VCSA crashes, I first checked the disk usage on the ESXi host and found that the primary VMFS datastore was almost completely full:

```
[root@vc1] df
Filesystem       Bytes        Used  Available Use% Mounted on
VMFS-6     96368328704 94978965504 1389363200  99% /vmfs/volumes/66b0c33b-d2f3dff8-8f53-1e00f500013e
VMFSOS      8321499136  2961178624 5360320512  36% /vmfs/volumes/OSDATA-66aa25b4-80eeca2a-d6dd-1e00f7000340
vfat        1073577984   268500992  805076992  25% /vmfs/volumes/BOOTBANK1
vfat        1073577984       32768 1073545216   0% /vmfs/volumes/BOOTBANK2

[root@vc1] esxcli storage filesystem list
Mount Point                                        Volume Name                                 UUID                                 Mounted  Type           Size         Free
-------------------------------------------------  ------------------------------------------  -----------------------------------  -------  ------  -----------  -----------
/vmfs/volumes/66b0c33b-d2f3dff8-8f53-1e00f500013e                                              66b0c33b-d2f3dff8-8f53-1e00f500013e     true  VMFS-6  96368328704  18654167040
/vmfs/volumes/66aa25b4-80eeca2a-d6dd-1e00f7000340  OSDATA-66aa25b4-80eeca2a-d6dd-1e00f7000340  66aa25b4-80eeca2a-d6dd-1e00f7000340     true  VMFSOS   8321499136   5360320512
/vmfs/volumes/fa2aa265-4242031f-848c-37c3316eebbb  BOOTBANK1                                   fa2aa265-4242031f-848c-37c3316eebbb     true  vfat     1073577984    805076992
/vmfs/volumes/0fa5a00a-354830aa-dca6-20a7b11416cd  BOOTBANK2                                   0fa5a00a-354830aa-dca6-20a7b11416cd     true  vfat     1073577984   1073545216
```

The output of `esxcli storage filesystem list` confirmed that the VMFS-6 datastore had very limited free space remaining, which explained the “out of space” errors observed when managing the VCSA virtual machine.

However, this result was unexpected, as the ESXi host had originally been provisioned with a 150 GB data disk. This suggested that not all available disk space had been allocated to the VMFS datastore.

To verify this, I examined the disk partition layout using `esxcli` and `partedUtil`:

```
[root@vc1] esxcli storage core device list
mpx.vmhba4:C0:T0:L0
   Display Name: Local NECVMWar CD-ROM (mpx.vmhba4:C0:T0:L0)
   Has Settable Display Name: false
   Size: 0
   Device Type: CD-ROM 
   Multipath Plugin: NMP
   Devfs Path: /vmfs/devices/cdrom/mpx.vmhba4:C0:T0:L0
   Vendor: NECVMWar
   Model: VMware IDE CDR00
   Revision: 1.00
   SCSI Level: 5
   Is Pseudo: false
   Status: on
   Is RDM Capable: false
   Is Local: true
   Is Removable: true
   Is SSD: false
   Is VVOL PE: false
   Is Offline: false
   Is Perennially Reserved: false
   Queue Full Sample Size: 0
   Queue Full Threshold: 0
   Thin Provisioning Status: unknown
   Attached Filters: 
   VAAI Status: unsupported
   Other UIDs: vml.0005000000766d686261343a303a30
   Is Shared Clusterwide: false
   Is SAS: false
   Is USB: false
   Is Boot Device: false
   Device Max Queue Depth: 1
   No of outstanding IOs with competing worlds: 1
   Drive Type: unknown
   RAID Level: unknown
   Number of Physical Drives: unknown
   Protection Enabled: false
   PI Activated: false
   PI Type: 0
   PI Protection Mask: NO PROTECTION
   Supported Guard Types: NO GUARD SUPPORT
   DIX Enabled: false
   DIX Guard Type: NO GUARD SUPPORT
   Emulated DIX/DIF Enabled: false

mpx.vmhba0:C0:T0:L0
   Display Name: Local VMware Disk (mpx.vmhba0:C0:T0:L0)
   Has Settable Display Name: false
   Size: 153600
   Device Type: Direct-Access 
   Multipath Plugin: HPP
   Devfs Path: /vmfs/devices/disks/mpx.vmhba0:C0:T0:L0
   Vendor: VMware  
   Model: Virtual disk    
   Revision: 2.0 
   SCSI Level: 6
   Is Pseudo: false
   Status: on
   Is RDM Capable: false
   Is Local: true
   Is Removable: false
   Is SSD: false
   Is VVOL PE: false
   Is Offline: false
   Is Perennially Reserved: false
   Queue Full Sample Size: 0
   Queue Full Threshold: 0
   Thin Provisioning Status: unknown
   Attached Filters: 
   VAAI Status: unsupported
   Other UIDs: vml.0000000000766d686261303a303a30
   Is Shared Clusterwide: false
   Is SAS: false
   Is USB: false
   Is Boot Device: true
   Device Max Queue Depth: 1024
   No of outstanding IOs with competing worlds: 32
   Drive Type: unknown
   RAID Level: unknown
   Number of Physical Drives: unknown
   Protection Enabled: false
   PI Activated: false
   PI Type: 0
   PI Protection Mask: NO PROTECTION
   Supported Guard Types: NO GUARD SUPPORT
   DIX Enabled: false
   DIX Guard Type: NO GUARD SUPPORT
   Emulated DIX/DIF Enabled: false

[root@vc1] partedUtil getptbl /vmfs/devices/disks/mpx.vmhba0:C0:T0:L0
gpt
19581 255 63 314572800
1 64 204863 C12A7328F81F11D2BA4B00A0C93EC93B systemPartition 128
5 208896 2306047 EBD0A0A2B9E5443387C068B6B72699C7 linuxNative 0
6 2308096 4405247 EBD0A0A2B9E5443387C068B6B72699C7 linuxNative 0
7 4407296 20971486 4EB2EA3978554790A79EFAE495E21F8D vmfsl 0
8 20973535 209715166 AA31E02A400F11DB9590000C2911D1B8 vmfs 0
```

The partition table showed that the VMFS partition (partition 8) ended well before the physical end of the disk. A closer analysis of the disk geometry confirmed the discrepancy:

```
Total sectors: 314,572,800
Sector size: 512 bytes (standard for ESXi)
Total disk size: 314,572,800 × 512 bytes ≈ 161,061,273,600 bytes ≈ 150 GB

Last used sector: 209,715,166
Last disk sector: 314,572,800
Free space (in sectors): 314,572,800 − 209,715,166 = 104,857,634 sectors
Free space (in GB): 104,857,634 × 512 bytes ≈ 53.7 GB
```

In other words, although the disk capacity was 150 GB, only about 96 GB had been allocated to the VMFS datastore. Approximately 54 GB of disk space remained unallocated at the end of the disk, leading to the datastore filling up and ultimately causing the VCSA to crash.

## Extend the VMFS partition

Extending the VMFS datastore involves two steps: first resizing the underlying partition, and then growing the VMFS filesystem itself.

### Step 1: Resize the VMFS Partition

The existing VMFS partition (partition 8) was expanded to consume the remaining unallocated disk space:

```
[root@vc1] partedUtil resize /vmfs/devices/disks/mpx.vmhba0:C0:T0:L0 \
          8 20973535 314572766
```

Note: 314572766 represents the last usable sector on the disk, leaving sufficient space for the GPT backup partition.

### Step 2: Grow the VMFS Filesystem

After resizing the partition, the VMFS filesystem was expanded to use the newly available space:

```
[root@vc1] vmkfstools --growfs \
          /vmfs/devices/disks/mpx.vmhba0:C0:T0:L0:8 \
          /vmfs/devices/disks/mpx.vmhba0:C0:T0:L0:8
```

### Verification

Finally, the datastore size and available free space were verified:

```
[root@vc1] esxcli storage filesystem list
Mount Point                                        Volume Name                                 UUID                                 Mounted  Type            Size         Free
-------------------------------------------------  ------------------------------------------  -----------------------------------  -------  ------  ------------  -----------
/vmfs/volumes/66b0c33b-d2f3dff8-8f53-1e00f500013e                                              66b0c33b-d2f3dff8-8f53-1e00f500013e     true  VMFS-6  150055419904  54943285248
/vmfs/volumes/66aa25b4-80eeca2a-d6dd-1e00f7000340  OSDATA-66aa25b4-80eeca2a-d6dd-1e00f7000340  66aa25b4-80eeca2a-d6dd-1e00f7000340     true  VMFSOS    8321499136   5360320512
/vmfs/volumes/fa2aa265-4242031f-848c-37c3316eebbb  BOOTBANK1                                   fa2aa265-4242031f-848c-37c3316eebbb     true  vfat      1073577984    805076992
/vmfs/volumes/0fa5a00a-354830aa-dca6-20a7b11416cd  BOOTBANK2                                   0fa5a00a-354830aa-dca6-20a7b11416cd     true  vfat      1073577984   1073545216

[root@vc1] df
Filesystem        Bytes        Used   Available Use% Mounted on
VMFS-6     150055419904 94990499840 55064920064  63% /vmfs/volumes/66b0c33b-d2f3dff8-8f53-1e00f500013e
VMFSOS       8321499136  2961178624  5360320512  36% /vmfs/volumes/OSDATA-66aa25b4-80eeca2a-d6dd-1e00f7000340
vfat         1073577984   268500992   805076992  25% /vmfs/volumes/BOOTBANK1
vfat         1073577984       32768  1073545216   0% /vmfs/volumes/BOOTBANK2
```

The VMFS datastore now correctly reflects the full 150 GB disk capacity, with sufficient free space restored. At this point, the issue is fully resolved.
