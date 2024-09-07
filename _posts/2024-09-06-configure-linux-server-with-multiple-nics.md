---
layout: post
title: Configure Linux Server with Multiple NICs
toc: true
categories: [Linux, Networking]
---

For linux server with multiple nics, it is a common issue that only one of the nic IP is reachable. The issue is due to missing routes of other nics. This article will show how to fix the issue.

<!--more-->

## Problem description

At first I created a VM from [Ubuntu 24.04 cloud image](https://cloud-images.ubuntu.com/minimal/releases/noble/release/) on two networks. The VM has two nics: `ens3` and `ens4`. By default, only `ens3` is enabled in netplan. After enabling `ens4` in netplan config and applying netplan configuration, the VM has two assigned IPs.

![IPs of test vm]({{ "/images/2024-09-07_10-24-test-vm-ips.png" | absolute_url }}){:width="100%"}{:.glightbox}

Now ping the two IPs from another server which can access both networks.

![Ping result from another server]({{ "/images/2024-09-07_10-25-ping-before-fix.png" | absolute_url }}){:width="100%"}{:.glightbox}

Only ping to the first IP worked. The IP on `ens4` is not pingable.

## Root analysis

Use `tcpdump` to monitor the packets, it shows the VM has received the ICMP request and sent the ICMP reply. However, it does not reach the source host.

![Tcpdump result in test vm]({{ "/images/2024-09-07_10-27-tcpdump-icmp-before-fix.png" | absolute_url }}){:width="100%"}{:.glightbox}

The test vm has the default route via `ens3` but no default route for `ens4`.

![IP route in test vm]({{ "/images/2024-09-07_10-28-ip-route-before-fix.png" | absolute_url }}){:width="100%"}{:.glightbox}


## Solve the problem

`iproute2` provides `ip-rule` tool to manage complicated routing policies.

```
echo "100 Table_ens3" >> /etc/iproute2/rt_tables
echo "101 Table_ens4" >> /etc/iproute2/rt_tables

ip rule add from 192.168.4.0/24 table Table_ens3
ip rule add from 192.168.8.0/24 table Table_ens4

ip route add default via 192.168.4.1 dev ens3 table Table_ens3
ip route add default via 192.168.8.1 dev ens4 table Table_ens4
```

![IP rule changes in test vm]({{ "/images/2024-09-07_10-39-ip-rule-route-changes.png" | absolute_url }}){:width="100%"}{:.glightbox}

After applying the above changes, the source server is able to ping IPs of both nics.

![Ping result after changes]({{ "/images/2024-09-07_10-39-ping-after-fix.png" | absolute_url }}){:width="100%"}{:.glightbox}

## References

- [1] http://linux-ip.net/html/tools-ip-rule.html
