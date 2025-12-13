---
layout: post
title: Mastering SAN in CloudStack (Part 1 - Setup iSCSI SAN)
toc: true
categories: [CloudStack, Storage]
---

Most CloudStack users settle for local storage or NFS in production. Local storage locks VMs to specific hosts, while NFS often becomes a performance bottleneck. There's a better way: SAN storage. 

In Part 1, we'll build an iSCSI SAN that combines the best of both worlds—shared access without the single-point bottlenecks, giving you true cloud flexibility without compromise.

<!--more-->

## Why Shared Storage Matters in CloudStack & Why SAN?

Most CloudStack users use NFS or local storage. CloudStack also support SAN as shared storage.

### Local Storage: The Illusion of Simplicity

What you gain:
- Maximum I/O performance (no network hops)
- Lowest latency (direct disk access)
- Simple setup and troubleshooting

What you lose:
- live migration takes long time
- Resource fragmentation: idle capacity can't be shared or balanced
- Maintenance nightmares: Host failure = VM downtime (not easy to recovery if host is corrupted)

Reality: You're not running a cloud—you're running separate virtualization hosts.

### NFS: The Smart Choice for Many Deployments

Simplicity & Reliability:
- Mature protocol: 30+ years of refinement
- Predictable performance: Easy to monitor and troubleshoot
- Native CloudStack support: First-class citizen in the UI
- File-level operations: Easy snapshots, cloning, backups

Limitations:
- NFS server becomes SPOF: Hardware failure = entire cluster down
- One server handles all I/O
- Performance bottleneck
- Scalability limited by one machine

### SAN: The Distributed Architecture

The concurrency advantage:
- Multiple hosts access storage simultaneously
- No single network path bottleneck
- Storage handles coordination, not hosts
- Multipath I/O: Active-active load balancing

Why this matters for CloudStack:
- Live Migration: VM disk on SAN → move compute between hosts freely
- Storage Overcommit: Thin provisioning across entire cluster
- Maintenance: Update hosts without VM downtime
- High Availability: Host fails → restart VM on any other host
- Performance Scaling: Add storage controllers independently of compute

## iSCSI vs Fibre Channel: Choosing Your Storage Backbone

### iSCSI SAN: Ethernet-Based Storage

iSCSI (Internet Small Computer System Interface) runs over your existing Ethernet network:

Pros:
- Uses existing network infrastructure
- Much cheaper than FC (no special HBAs or switches)
- Easier to configure and maintain
- Perfect for proof-of-concept and small-to-medium deployments

Cons:
- Performance depends on network quality
- Shared bandwidth with other traffic
- Higher CPU overhead (TCP/IP processing)

### FC SAN: Dedicated Storage Network

Fibre Channel uses dedicated hardware and protocols:

Pros:
- Dedicated, lossless network
- Extremely low latency
- Highest performance for demanding workloads
- Ideal for enterprise databases and high-IOPS applications

Cons:
- Very expensive (HBAs, switches, cables)
- Complex to configure and manage
- Requires specialized skills
- Vendor lock-in common

## Hands-On: Building iSCSI SAN for CloudStack

### Deploy a iSCSI server with DATA disk

```
[root@ol90 ~]# fdisk -l /dev/sdb
Disk /dev/sdb: 100 GiB, 107374182400 bytes, 209715200 sectors
Disk model: Virtual disk    
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

Create partition
```
[root@ol90 ~]# printf "n\np\n1\n\n\nw\n" | fdisk /dev/sdb

Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x741dd5ab.

Command (m for help): Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): Partition number (1-4, default 1): First sector (2048-209715199, default 2048): Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-209715199, default 209715199): 
Created a new partition 1 of type 'Linux' and of size 100 GiB.

Command (m for help): The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

then format with ext4 filesystem

```
[root@ol90 ~]# mkfs.ext4 /dev/sdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 26214144 4k blocks and 6553600 inodes
Filesystem UUID: 29492435-5e68-480c-b8c0-f84e4d6e0c32
Superblock backups stored on blocks: 
    32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
    4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (131072 blocks): done
Writing superblocks and filesystem accounting information: done   
```

Mount it locally
```
[root@ol90 ~]# mkdir -p /scsi
[root@ol90 ~]# mount /dev/sdb1 /scsi/
[root@ol90 ~]# echo "/dev/sdb1     /scsi    ext4  defaults  0 2" >>/etc/fstab
[root@ol90 ~]# ls -l /scsi/
total 16
drwx------. 2 root root 16384 Dec 12 19:36 lost+found
[root@ol90 ~]# 
```

### Setup as iSCSI server

```
[root@ol90 ~]# sudo dnf install targetcli -y
Oracle Linux 9 EPEL Packages for Development (x86_64)                                                                                         31 MB/s |  30 MB     00:00    
Oracle Linux 9 BaseOS Latest  (x86_64)                                                                                                        25 MB/s | 102 MB     00:04    
Oracle Linux 9 Application Stream Packages (x86_64)                                                                                           39 MB/s |  79 MB     00:02    
Oracle Linux 9 UEK Release 7 (x86_64)                                                                                                         44 MB/s |  89 MB     00:02    
Last metadata expiration check: 0:00:06 ago on Fri 12 Dec 2025 07:41:22 PM GMT.
Dependencies resolved.
=============================================================================================================================================================================
 Package                                   Architecture                      Version                                      Repository                                    Size
=============================================================================================================================================================================
Installing:
 targetcli                                 noarch                            2.1.57-2.el9                                 ol9_appstream                                101 k
Installing dependencies:
 python3-pyudev                            noarch                            0.22.0-6.el9                                 ol9_baseos_latest                            143 k
 python3-rtslib                            noarch                            2.1.76-1.0.1.el9                             ol9_appstream                                147 k
 target-restore                            noarch                            2.1.76-1.0.1.el9                             ol9_appstream                                 18 k

Transaction Summary
=============================================================================================================================================================================
Install  4 Packages

Total download size: 409 k
Installed size: 1.2 M
Downloading Packages:
(1/4): python3-pyudev-0.22.0-6.el9.noarch.rpm                                                                                                600 kB/s | 143 kB     00:00    
(2/4): python3-rtslib-2.1.76-1.0.1.el9.noarch.rpm                                                                                            613 kB/s | 147 kB     00:00    
(3/4): target-restore-2.1.76-1.0.1.el9.noarch.rpm                                                                                             76 kB/s |  18 kB     00:00    
(4/4): targetcli-2.1.57-2.el9.noarch.rpm                                                                                                     1.6 MB/s | 101 kB     00:00    
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                        1.3 MB/s | 409 kB     00:00     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                     1/1 
  Installing       : python3-pyudev-0.22.0-6.el9.noarch                                                                                                                  1/4 
  Installing       : python3-rtslib-2.1.76-1.0.1.el9.noarch                                                                                                              2/4 
  Installing       : target-restore-2.1.76-1.0.1.el9.noarch                                                                                                              3/4 
  Running scriptlet: target-restore-2.1.76-1.0.1.el9.noarch                                                                                                              3/4 
  Installing       : targetcli-2.1.57-2.el9.noarch                                                                                                                       4/4 
  Running scriptlet: targetcli-2.1.57-2.el9.noarch                                                                                                                       4/4 
  Verifying        : python3-pyudev-0.22.0-6.el9.noarch                                                                                                                  1/4 
  Verifying        : python3-rtslib-2.1.76-1.0.1.el9.noarch                                                                                                              2/4 
  Verifying        : target-restore-2.1.76-1.0.1.el9.noarch                                                                                                              3/4 
  Verifying        : targetcli-2.1.57-2.el9.noarch                                                                                                                       4/4 

Installed:
  python3-pyudev-0.22.0-6.el9.noarch        python3-rtslib-2.1.76-1.0.1.el9.noarch        target-restore-2.1.76-1.0.1.el9.noarch        targetcli-2.1.57-2.el9.noarch       

Complete!
```

Enable it so it will start automatically after reboot
```
[root@ol90 ~]# sudo systemctl enable --now target
Created symlink /etc/systemd/system/multi-user.target.wants/target.service → /usr/lib/systemd/system/target.service.
```

Below is the original state
```
[root@ol90 ~]# sudo targetcli ls
Warning: Could not load preferences file /root/.targetcli/prefs.bin.
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 0]
  o- loopback ......................................................................................................... [Targets: 0]
  o- vhost ............................................................................................................ [Targets: 0]
[root@ol90 ~]# 
```

### Create iSCSI target

Run below commands to create a image and expose it as iSCSI target
```
disk=disk01
size=40G
dir=/scsi
iscsi_prefix=iqn.2025-12.local
iscsi_target_iqn=$iscsi_prefix.server:$disk

targetcli backstores/fileio create $disk $dir/$disk.img $size &&
targetcli /iscsi create $iscsi_target_iqn &&
targetcli /iscsi/$iscsi_target_iqn/tpg1/luns/ create /backstores/fileio/$disk &&
targetcli /iscsi/$iscsi_target_iqn/tpg1/acls/ create $iscsi_prefix.client:node01 &&
targetcli /iscsi/$iscsi_target_iqn/tpg1 set attribute authentication=0 &&
targetcli /iscsi/$iscsi_target_iqn/tpg1 set attribute demo_mode_write_protect=0 &&
targetcli /iscsi/$iscsi_target_iqn/tpg1 set attribute generate_node_acls=1 &&
targetcli saveconfig
```

The output as below
```
Created fileio disk01 with size 42949672960
Created target iqn.2025-12.local.server:disk01.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
Created LUN 0.
Created Node ACL for iqn.2025-12.local.client:node01
Created mapped LUN 0.
Parameter authentication is now '0'.
Parameter demo_mode_write_protect is now '0'.
Parameter generate_node_acls is now '1'.
Configuration saved to /etc/target/saveconfig.json
```

New state
```

[root@ol90 ~]# targetcli ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 1]
  | | o- disk01 .................................................................. [/scsi/disk01.img (40.0GiB) write-back activated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2025-12.local.server:disk01 ................................................................................... [TPGs: 1]
  |   o- tpg1 .................................................................................................. [gen-acls, no-auth]
  |     o- acls .......................................................................................................... [ACLs: 1]
  |     | o- iqn.2025-12.local.client:node01 ...................................................................... [Mapped LUNs: 1]
  |     |   o- mapped_lun0 ............................................................................... [lun0 fileio/disk01 (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 ............................................................ [fileio/disk01 (/scsi/disk01.img) (default_tg_pt_gp)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................................................................................... [OK]
  o- loopback ......................................................................................................... [Targets: 0]
  o- vhost ............................................................................................................ [Targets: 0]

```

### Create another iSCSI target (Optionally)

```
[root@ol90 ~]# disk=disk02
size=42G
dir=/scsi
iscsi_prefix=iqn.2025-12.local
iscsi_target_iqn=$iscsi_prefix.server:$disk

[root@ol90 ~]#  targetcli backstores/fileio create $disk $dir/$disk.img $size &&
targetcli /iscsi create $iscsi_target_iqn &&
targetcli /iscsi/$iscsi_target_iqn/tpg1/luns/ create /backstores/fileio/$disk &&
targetcli /iscsi/$iscsi_target_iqn/tpg1/acls/ create $iscsi_prefix.client:node01 &&
targetcli /iscsi/$iscsi_target_iqn/tpg1 set attribute authentication=0 &&
targetcli /iscsi/$iscsi_target_iqn/tpg1 set attribute demo_mode_write_protect=0 &&
targetcli /iscsi/$iscsi_target_iqn/tpg1 set attribute generate_node_acls=1 &&
targetcli saveconfig

Created fileio disk02 with size 45097156608
Created target iqn.2025-12.local.server:disk02.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
Created LUN 0.
Created Node ACL for iqn.2025-12.local.client:node01
Created mapped LUN 0.
Parameter authentication is now '0'.
Parameter demo_mode_write_protect is now '0'.
Parameter generate_node_acls is now '1'.
Last 10 configs saved in /etc/target/backup/.
Configuration saved to /etc/target/saveconfig.json
[root@ol90 ~]# 
```

New state
```
[root@ol90 ~]# targetcli ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 2]
  | | o- disk01 .................................................................. [/scsi/disk01.img (40.0GiB) write-back activated]
  | | | o- alua ................................................................................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | | o- disk02 .................................................................. [/scsi/disk02.img (42.0GiB) write-back activated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 2]
  | o- iqn.2025-12.local.server:disk01 ................................................................................... [TPGs: 1]
  | | o- tpg1 .................................................................................................. [gen-acls, no-auth]
  | |   o- acls .......................................................................................................... [ACLs: 1]
  | |   | o- iqn.2025-12.local.client:node01 ...................................................................... [Mapped LUNs: 1]
  | |   |   o- mapped_lun0 ............................................................................... [lun0 fileio/disk01 (rw)]
  | |   o- luns .......................................................................................................... [LUNs: 1]
  | |   | o- lun0 ............................................................ [fileio/disk01 (/scsi/disk01.img) (default_tg_pt_gp)]
  | |   o- portals .................................................................................................... [Portals: 1]
  | |     o- 0.0.0.0:3260 ..................................................................................................... [OK]
  | o- iqn.2025-12.local.server:disk02 ................................................................................... [TPGs: 1]
  |   o- tpg1 .................................................................................................. [gen-acls, no-auth]
  |     o- acls .......................................................................................................... [ACLs: 1]
  |     | o- iqn.2025-12.local.client:node01 ...................................................................... [Mapped LUNs: 1]
  |     |   o- mapped_lun0 ............................................................................... [lun0 fileio/disk02 (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 ............................................................ [fileio/disk02 (/scsi/disk02.img) (default_tg_pt_gp)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................................................................................... [OK]
  o- loopback ......................................................................................................... [Targets: 0]
  o- vhost ............................................................................................................ [Targets: 0]
```

### Enable firewall

Add a firewall rule to allow port 3260
```
sudo firewall-cmd --add-port=3260/tcp --permanent
sudo firewall-cmd --reload
```

Alternatively, stop and disable the firewall totally
```
systemctl stop firewalld && systemctl disable firewalld
```

## Hands-On: Find iSCSI Targets on KVM Hosts

### Discover iSCSI targets

```
[root@kvm1 ~]# iscsiadm -m discovery -t sendtargets -p 10.1.32.137
10.1.32.137:3260,1 iqn.2025-12.local.server:disk01
10.1.32.137:3260,1 iqn.2025-12.local.server:disk02
```
Login
```
[root@kvm1 ~]# iscsiadm -m node --login
Logging in to [iface: default, target: iqn.2025-12.local.server:disk01, portal: 10.1.32.137,3260]
Logging in to [iface: default, target: iqn.2025-12.local.server:disk02, portal: 10.1.32.137,3260]
Login to [iface: default, target: iqn.2025-12.local.server:disk01, portal: 10.1.32.137,3260] successful.
Login to [iface: default, target: iqn.2025-12.local.server:disk02, portal: 10.1.32.137,3260] successful.
```

### Identify Connected Targets
```
[root@kvm1 ~]# iscsiadm -m session
tcp: [1] 10.1.32.137:3260,1 iqn.2025-12.local.server:disk01 (non-flash)
tcp: [2] 10.1.32.137:3260,1 iqn.2025-12.local.server:disk02 (non-flash)
```

### Map Targets to Local Devices
```
[root@kvm1 ~]# iscsiadm -m session -P 3 | grep -E "(Target:|Attached scsi disk)"
Target: iqn.2025-12.local.server:disk01 (non-flash)
            Attached scsi disk sdb        State: running
Target: iqn.2025-12.local.server:disk02 (non-flash)
            Attached scsi disk sdc        State: running

[root@kvm1 ~]# fdisk -l /dev/sdb
Disk /dev/sdb: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 8388608 bytes

[root@kvm1 ~]# fdisk -l /dev/sdc
Disk /dev/sdc: 42 GiB, 45097156608 bytes, 88080384 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 8388608 bytes
```

Note: <b>The iSCSI targets might be mapped to local devices in different order: (1) on different severs; (2) After rebooting.</b>

### FAQ: How to remove iSCSI targets

```
[root@kvm1 ~]# ls -l /var/lib/iscsi/send_targets
total 0
drwx------. 2 root root 155 Dec 12 20:35 10.1.32.137,3260

[root@kvm1 ~]# ls -l /var/lib/iscsi/send_targets/10.1.32.137,3260/
total 4
lrwxrwxrwx. 1 root root  71 Dec 12 20:35 iqn.2025-12.local.server:disk01,10.1.32.137,3260,1,default -> /var/lib/iscsi/nodes/iqn.2025-12.local.server:disk01/10.1.32.137,3260,1
lrwxrwxrwx. 1 root root  71 Dec 12 20:35 iqn.2025-12.local.server:disk02,10.1.32.137,3260,1,default -> /var/lib/iscsi/nodes/iqn.2025-12.local.server:disk02/10.1.32.137,3260,1
-rw-------. 1 root root 581 Dec 12 20:35 st_config

// Logout and remove persistent login
[root@kvm1 ~]# iscsiadm -m node
[root@kvm1 ~]# iscsiadm -m node -T iqn.2025-12.local.server:disk01 -p 10.1.32.137 --logout
[root@kvm1 ~]# iscsiadm -m node -T iqn.2025-12.local.server:disk01 -p 10.1.32.137 -o delete 
[root@kvm1 ~]# iscsiadm -m node -T iqn.2025-12.local.server:disk02 -p 10.1.32.137 --logout
[root@kvm1 ~]# iscsiadm -m node -T iqn.2025-12.local.server:disk02 -p 10.1.32.137 -o delete 

// Remove iSCSI server
[root@kvm1 ~]# rm -rf /var/lib/iscsi/send_targets/10.1.32.137,3260/
```