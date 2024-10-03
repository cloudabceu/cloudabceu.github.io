---
layout: post
title: Configure Linux Server as Router with Dynamic BGP (part 2)
toc: true
categories: [Linux, Networking]
---

This part will introduce how to configure a Linux router to run with Dynamic BGP. BGP (Border Gateway Protocol) is a gateway protocol which is widely used to exchange routing information between routers.

<!--more-->

## Step 5: Install and configure FRR

[FRR](https://frrouting.org/) is an open-source software which supports a few routing protocols, for example BGP, OSPF. It is available in most popular Linux distributions, it can be installed via apt/yum/dnf.

By default, no routing protocol is enabled. We can enable bgp by the following steps
```
sed -i "s/bgpd=no/bgpd=yes/g" /etc/frr/daemons
systemctl restart frr
```

## Step 6: Setup FRR with Dynamic BGP

Now create a frr configuration file `/etc/frr/frr.conf` with content below.

```
log syslog informational

frr version 6.0
frr defaults traditional
hostname bgp-router
service integrated-vtysh-config
!
router bgp 1003
 no bgp default ipv4-unicast
 bgp router-id 10.200.0.1
 neighbor ACS-V4 peer-group
 neighbor ACS-V4 remote-as external
 neighbor ACS-V4 ebgp-multihop 10
 neighbor ACS-V4 password password3
 bgp listen range 10.200.0.1/24 peer-group ACS-V4
 !
 address-family ipv4 unicast
  neighbor ACS-V4 activate
  neighbor ACS-V4 route-reflector-client
 exit-address-family
 !
 bgp default ipv6-unicast
 neighbor ACS-V6 peer-group
 neighbor ACS-V6 remote-as external
 neighbor ACS-V6 ebgp-multihop 10
 bgp listen range fc00:2024:9:7::/64 peer-group ACS-V6
 !
 address-family ipv6 unicast
  redistribute connected
  neighbor ACS-V6 activate
  neighbor ACS-V6 route-reflector-client
 exit-address-family
 !
line vty
!
```

Restarting FRR, then the linux router is ready for BGP connections.

## Explanation of FRR configuration

- `router bgp 1003`: The Linux router runs with AS number 1003.

- `no bgp default ipv4-unicast`: It accepts both IPv4 and Ipv6 unicast.
- `bgp router-id 10.200.0.1`: the router ID is 10.200.0.1 (also router IP).
- `neighbor ACS-V4 peer-group`: defines a new peer group `ACS-V4`
- `neighbor ACS-V4 remote-as external`: allows all ASNs except the router ASN 1003.
- `neighbor ACS-V4 ebgp-multihop 10`: allows eBGP neighbors with multiple hops away.
- `neighbor ACS-V4 password password3`: the password of peer group is `password3`.
- `bgp listen range 10.200.0.1/24 peer-group ACS-V4`: allows eBGP neighbors from IP range `10.200.0.1/24`.

- `neighbor ACS-V4 activate`: enables IPv4 unicast.
- `neighbor ACS-V4 route-reflector-client`: To avoid single points of failure, multiple route reflectors can be configured.

For more details, refer to [BGP Router Configuration](https://docs.frrouting.org/en/latest/bgp.html#bgp-router-configuration)

## Step 7: Verify BGP sessions: Create another Linux router

To verify the BGP sessions, create another Linux router in the same subnet. It has aditional NIC with different IP range.

```
...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:01:03:9f:00:02 brd ff:ff:ff:ff:ff:ff
    inet 192.168.20.1/28 brd 192.168.20.15 scope global eth0
    inet6 2024:6:18:c::1/64 scope global 
    inet6 fe80::1:3ff:fe9f:2/64 scope link 
...
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 1e:00:ba:00:03:6a brd ff:ff:ff:ff:ff:ff
    inet 10.200.0.19/24 brd 10.200.0.255 scope global eth2
    inet6 fc00:2024:9:7:1c00:baff:fe00:36a/64 scope global 
    inet6 fe80::1c00:baff:fe00:36a/64 scope link 
```

It also has FRR installed and configured as below

```
frr version 6.0
frr defaults traditional
hostname r-1011-VM
service integrated-vtysh-config
ip nht resolve-via-default
router bgp 5051
 bgp router-id 10.200.0.19
 bgp default ipv6-unicast
 neighbor 10.200.0.1 remote-as 1003
 neighbor 10.200.0.1 password password3
 neighbor fc00:2024:9:7::1 remote-as 1003
 neighbor fc00:2024:9:7::1 password password3
 address-family ipv4 unicast
  network 192.168.20.0/28
 exit-address-family
 address-family ipv6 unicast
  network 2024:6:18:c::/64
 exit-address-family
line vty
```


## Step 7: Verify BGP sessions: Check in Linux router (1)

Now let's check the frr status on both servers.

On Linux router (10.200.0.1), run `vtysh` as root user.
```
bgp-router# show bgp summary

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.200.0.1, local AS number 1003 vrf-id 0
BGP table version 2
RIB entries 3, using 576 bytes of memory
Peers 2, using 1448 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
*10.200.0.11    4       5033     34917     34916        0    0    0 01w3d05h            1        2 N/A
*10.200.0.19    4       5051     36523     36522        0    0    0 01w4d08h            1        2 N/A

Total number of neighbors 2
* - dynamic neighbor
2 dynamic neighbor(s), limit 100

IPv6 Unicast Summary (VRF default):
BGP router identifier 10.200.0.1, local AS number 1003 vrf-id 0
BGP table version 3
RIB entries 5, using 960 bytes of memory
Peers 2, using 1448 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
*10.200.0.11    4       5033     34917     34916        0    0    0 01w3d05h            1        3 N/A
*10.200.0.19    4       5051     36523     36522        0    0    0 01w4d08h            1        3 N/A

Total number of neighbors 2
* - dynamic neighbor
2 dynamic neighbor(s), limit 100
```

The BGP session is working on both Ipv4 and Ipv6.

```
bgp-router# show ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

...
C>* 10.200.0.0/24 is directly connected, eth1, 01w4d08h
B>* 192.168.20.0/28 [20/0] via 10.200.0.19, eth1, weight 1, 01w4d08h
B>* 192.168.20.16/28 [20/0] via 10.200.0.11, eth1, weight 1, 01w3d05h


bgp-router# show ipv6 route
Codes: K - kernel route, C - connected, S - static, R - RIPng,
       O - OSPFv3, I - IS-IS, B - BGP, N - NHRP, T - Table,
       v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

B>* 2024:6:18:7::/64 [20/0] via fe80::1c00:a5ff:fe00:362, eth1, weight 1, 01w3d05h
B>* 2024:6:18:c::/64 [20/0] via fe80::1c00:baff:fe00:36a, eth1, weight 1, 01w4d08h
C>* fc00:2024:9:7::/64 is directly connected, eth1, 01w4d08h
...
```
Both Ipv4 and Ipv6 routes are advertised to children routers.

## Step 8: Verify BGP sessions: Check in Linux router (2)

Now let's check the childen router (10.200.0.19).

```
r-1011-VM# show bgp summary

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.200.0.19, local AS number 5051 vrf-id 0
BGP table version 2
RIB entries 3, using 576 bytes of memory
Peers 6, using 4345 KiB of memory

Neighbor           V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.200.0.1         4       1003     36525     36526        0    0    0 01w4d08h            1        2 N/A
fc00:2024:9:7::1   4       1003         0         0        0    0    0    never      Connect        0 N/A

Total number of neighbors 2

IPv6 Unicast Summary (VRF default):
BGP router identifier 10.200.0.19, local AS number 5051 vrf-id 0
BGP table version 3
RIB entries 5, using 960 bytes of memory
Peers 6, using 4345 KiB of memory

Neighbor           V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.200.0.1         4       1003     36525     36526        0    0    0 01w4d08h            2        3 N/A
fc00:2024:9:7::1   4       1003         0         0        0    0    0    never      Connect        0 N/A

Total number of neighbors 2
```

Now check the routes
```
r-1011-VM# show ip route
...
C>* 192.168.20.0/28 is directly connected, eth0, 01w4d08h
B>* 192.168.20.16/28 [20/0] via 10.200.0.11, eth2, weight 1, 01w3d05h

r-1011-VM# show ipv6 route
B>* 2024:6:18:7::/64 [20/0] via fe80::3eff:fe61:1, eth2, weight 1, 01w3d05h
C>* 2024:6:18:c::/64 is directly connected, eth0, 01w4d08h
B   fc00:2024:9:7::/64 [20/0] via fe80::3eff:fe61:1, eth2, weight 1, 01w4d08h
C>* fc00:2024:9:7::/64 is directly connected, eth2, 01w4d08h
```

The children router has successfully configured the IPv4 and IPv6 routes advertised by its parent router.


## Summary

These two parts introduce how to configure a Linux router with Dynamic BGP from scratch. Dynamic BGP is useful in large scale environments to setup multiple networks without manual intervention.
