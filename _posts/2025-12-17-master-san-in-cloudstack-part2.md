---
layout: post
title: Mastering SAN in CloudStack (Part 2 - Setup OCFS2 Cluster File System)
toc: true
categories: [CloudStack, Storage]
---

When mounting an iSCSI target on two or more servers simultaneously using conventional filesystems (ext4, XFS, etc.), data corruption is inevitable. Each server maintains independent caches and metadata in memory, creating a critical coordination gap: neither server is aware of the other's modifications. This lack of synchronization leads to: Metadata corruption, Lost or duplicated files, Filesystem structural damage, Potential complete data loss.

To safely share storage across multiple servers, you must implement a cluster-aware filesystem specifically designed for concurrent access. Currently, the two most prominent options are: OCFS2 (Oracle Cluster File System) - Oracle's robust, feature-rich solution; GFS2 (Global File System) - Red Hat's scalable cluster filesystem.

In this part, we'll focus on implementing OCFS2 across Oracle Linux servers, providing step-by-step configuration for reliable shared storage access.


<!--more-->

## Step 1: Install dependencies (on all servers)

```
# Install OCFS2 packages (on both kvm1 and kvm2)
sudo dnf install ocfs2-tools

# Check if modules are loaded
lsmod | grep ocfs2
sudo modprobe ocfs2
sudo modprobe ocfs2_dlm

# Add firewall rules if firewalld is enabled
sudo firewall-cmd --permanent --add-port=7777/tcp
sudo firewall-cmd --permanent --add-port=7000-7100/tcp
sudo firewall-cmd --reload
```

## Step 2: Create OCFS2 partition (on a server)

Please ensure the iSCSI targets are discovered and user has logged in.

```
wipefs -a /dev/sdb
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart primary 0% 100%
parted /dev/sdb print
mkfs.ocfs2 -L "cloudstack1" -N 4 /dev/sdb1
fsck.ocfs2 -n /dev/sdb1
```

See the output as below

```
[root@kvm1 ~]# wipefs -a /dev/sdb

[root@kvm1 ~]# parted /dev/sdb mklabel gpt
Information: You may need to update /etc/fstab.

[root@kvm1 ~]# parted /dev/sdb mkpart primary 0% 100%                     
Information: You may need to update /etc/fstab.

[root@kvm1 ~]# parted /dev/sdb print                                      
Model: LIO-ORG disk01 (scsi)
Disk /dev/sdb: 42.9GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
 1      8389kB  42.9GB  42.9GB               primary

[root@kvm1 ~]# mkfs.ocfs2 -L "cloudstack1" -N 4 /dev/sdb1
mkfs.ocfs2 1.8.6
Cluster stack: o2cb
Cluster name: cloudabc
Stack Flags: 0x1
NOTE: Feature extended slot map may be enabled
Label: cloudstack1
Features: sparse extended-slotmap backup-super unwritten inline-data strict-journal-super xattr indexed-dirs refcount discontig-bg
Block size: 4096 (12 bits)
Cluster size: 4096 (12 bits)
Volume size: 42932895744 (10481664 clusters) (10481664 blocks)
Cluster groups: 325 (tail covers 30720 clusters, rest cover 32256 clusters)
Extent allocator size: 12582912 (3 groups)
Journal size: 268328960
Node slots: 4
Creating bitmaps: done
Initializing superblock: done
Writing system files: done
Writing superblock: done
Writing backup superblock: 3 block(s)
Formatting Journals: done
Growing extent allocator: done
Formatting slot map: done
Formatting quota files: done
Writing lost+found: done
mkfs.ocfs2 successful

[root@kvm1 ~]# fsck.ocfs2 -n /dev/sdb1
fsck.ocfs2 1.8.6
Checking OCFS2 filesystem in /dev/sdb1:
  Label:              cloudstack1
  UUID:               F4533D5A02D3441BAE948A7006BB27C8
  Number of blocks:   10481664
  Block size:         4096
  Number of clusters: 10481664
  Cluster size:       4096
  Number of slots:    4

** Skipping slot recovery because -n was given. **
/dev/sdb1 is clean.  It will be checked after 20 additional mounts.

[root@kvm1 ~]# 
```

## Step 3: create OCFS2 cluster (on a server)

```
[root@kvm1 ~]# create OCFS2 cluster
mkdir -p /etc/ocfs2/
cat > /etc/ocfs2/cluster.conf <<EOF
cluster:
    name = cloudabc
    heartbeat_mode = global
    node_count = 2

node:
    cluster = cloudabc
    number = 1
    ip_port = 7777
    ip_address = 10.1.34.125
    name = kvm1

node:
    cluster = cloudabc
    number = 2
    ip_port = 7777
    ip_address = 10.1.35.76
    name = kvm2
EOF

[root@kvm1 ~]# mv /etc/sysconfig/o2cb /etc/sysconfig/o2cb.orig

[root@kvm1 ~]# cat > /etc/sysconfig/o2cb <<EOF
O2CB_ENABLED=true
O2CB_BOOTCLUSTER=cloudabc
O2CB_HEARTBEAT_THRESHOLD=31
O2CB_STACK=o2cb
EOF

[root@kvm1 ~]# systemctl stop o2cb

[root@kvm1 ~]# systemctl start o2cb
```

Now check the status of the OCFS2 cluster:

```
[root@kvm1 ~]# o2cb list-cluster cloudabc
node:
	number = 1
	name = kvm1
	ip_address = 10.1.34.125
	ip_port = 7777
	cluster = cloudabc

node:
	number = 2
	name = kvm2
	ip_address = 10.1.35.76
	ip_port = 7777
	cluster = cloudabc

cluster:
	node_count = 2
	heartbeat_mode = global
	name = cloudabc

[root@kvm1 ~]# o2cb cluster-status cloudabc
Cluster 'cloudabc' is offline
```

Unfortunately, the cluster is created but offline.

## Step 4: configure heartbeat (on all servers)

```
[root@kvm1 ~]# o2cb add-heartbeat cloudabc /dev/sdb1
[root@kvm1 ~]# o2cb start-heartbeat cloudabc --heartbeat-device=/dev/sdb1
Global heartbeat started

[root@kvm2 ~]# o2cb add-heartbeat cloudabc /dev/sdb1
[root@kvm2 ~]# o2cb start-heartbeat cloudabc --heartbeat-device=/dev/sdb1
Global heartbeat started

[root@kvm1 ~]# o2cb cluster-status cloudabc
Cluster 'cloudabc' is online
```

## Step 5: Verfication

```
[root@kvm1 ~]# o2cb.init status
Driver for "configfs": Loaded
Filesystem "configfs": Mounted
Stack glue driver: Loaded
Stack plugin "o2cb": Loaded
Driver for "ocfs2_dlmfs": Not loaded
Checking O2CB cluster "cloudabc": Online
  Heartbeat dead threshold: 31
  Network idle timeout: 30000
  Network keepalive delay: 2000
  Network reconnect delay: 2000
  Heartbeat mode: Global
Checking O2CB heartbeat: Active
  0CF2E83DE01F416591FEDC9C4F29A05A /dev/sdc1
  F4533D5A02D3441BAE948A7006BB27C8 /dev/sdb1
Nodes in O2CB cluster: 1 2 
Debug file system at /sys/kernel/debug: mounted

[root@kvm1 ~]# 
```

## Common issues

### 1. Cluster is not registered

```
[root@kvm1 ~]# o2cb start-heartbeat cloudabc --heartbeat-device=/dev/sdc1
o2cb: Cluster 'cloudabc' not registered

[root@kvm1 ~]# o2cb register-cluster cloudabc
```

### 2. Heartbeat region could not be found

remove and create the heartbeat

```
[root@kvm1 ~]# o2cb list-cluster cloudabc

[root@kvm1 ~]# o2cb start-heartbeat cloudabc --heartbeat-device=/dev/sdb1
o2cb: Heartbeat region could not be found 8BDD5C5FC98B4F61B456F6E31F2EB8A7

[root@kvm1 ~]# o2cb remove-heartbeat cloudabc 8BDD5C5FC98B4F61B456F6E31F2EB8A7

[root@kvm1 ~]# o2cb start-heartbeat cloudabc --heartbeat-device=/dev/sdb1
Global heartbeat started
```

### 3. OCFS2 partition is not found on server

Rescan iSCSI device
```
[root@kvm2 ~]# sudo iscsiadm -m session --rescan
[root@kvm2 ~]# sudo partprobe /dev/sdb
[root@kvm2 ~]# sudo blockdev --rereadpt /dev/sdb
```

### 4. Devices are found on different order on servers

For example, `cloudstack1` is mounted as `/dev/sdb1` on kvm1, but `/dev/sdc1` on kvm2.

Please note: this is also found when server is rebooted.

```
[root@kvm1 ~]# ls -l /dev/disk/by-label/
total 0
lrwxrwxrwx. 1 root root 10 Dec 13 07:47 boot -> ../../sda1
lrwxrwxrwx. 1 root root 10 Dec 13 08:33 cloudstack1 -> ../../sdb1
lrwxrwxrwx. 1 root root 10 Dec 13 08:33 cloudstack2 -> ../../sdc1
lrwxrwxrwx. 1 root root 10 Dec 13 07:47 root -> ../../dm-0

[root@kvm2 ~]# ls -l /dev/disk/by-label/
total 0
lrwxrwxrwx. 1 root root 10 Dec 13 07:47 boot -> ../../sda1
lrwxrwxrwx. 1 root root 10 Dec 13 08:32 cloudstack1 -> ../../sdc1
lrwxrwxrwx. 1 root root 10 Dec 13 08:32 cloudstack2 -> ../../sdb1
lrwxrwxrwx. 1 root root 10 Dec 13 07:47 root -> ../../dm-0
```

In this case, it would be good to mount the devices by label or uuid, not by name.

```
[root@kvm1 ~]# mkdir /mnt/cloudstack1
[root@kvm1 ~]# mkdir /mnt/cloudstack2
[root@kvm1 ~]# mount /dev/disk/by-label/cloudstack1 /mnt/cloudstack1
[root@kvm1 ~]# mount /dev/disk/by-label/cloudstack2 /mnt/cloudstack2
```

OR by uuid:

```
[root@kvm2 ~]# blkid
/dev/sdb1: LABEL="cloudstack2" UUID="0cf2e83d-e01f-4165-91fe-dc9c4f29a05a" BLOCK_SIZE="4096" TYPE="ocfs2" PARTLABEL="primary" PARTUUID="22baa158-8849-46d1-807c-0c4679c32894"
/dev/sdc1: LABEL="cloudstack1" UUID="f4533d5a-02d3-441b-ae94-8a7006bb27c8" BLOCK_SIZE="4096" TYPE="ocfs2" PARTLABEL="primary" PARTUUID="a7381b0f-3fe1-4721-bc58-dc35118f03c5"
...

[root@kvm2 ~]# mount /dev/disk/by-uuid/f4533d5a-02d3-441b-ae94-8a7006bb27c8 /mnt/cloudstack1/
[root@kvm2 ~]# 
```

### 5. Automatic mount after reboot

```
# add these lines to /etc/fstab
/dev/disk/by-label/cloudstack1  /mnt/cloudstack1     ocfs2  _netdev,noatime,nodiratime,inode64,data=ordered     0   0
/dev/disk/by-label/cloudstack2  /mnt/cloudstack2     ocfs2  _netdev,noatime,nodiratime,inode64,data=ordered     0   0
```
