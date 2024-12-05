---
layout: post
title: Configure Destination NAT and Port Forwarding in Linux Router
toc: true
categories: [Linux, Networking]
---

This blog post introduces how to configure Destination NAT (DNAT, or Static NAT) and port forwarding in a Linux router, using iptables.

<!--more-->

## Step 1: Configure a Linux Router

Please refer to a previous blog post [Configure Linux Server as Router with Dynamic BGP (part 1)]({{ "linux/networking/2024/09/10/configure-linux-server-as-dynamic-bgp-router-part1/" | absolute_url }})

## Step 2: Configure Destination NAT (DNAT)

Firstly add the IP address to the Linux Router, then add iptables rules. 

For example, the Public IP is `10.0.x.13/20`, the Internal IP (e.g. VM) is `10.200.y.129`

```
$ ip addr add dev eth0 10.0.x.13/20

$ iptables -t nat -A OUTPUT -d 10.0.x.13/32 -j DNAT --to-destination 10.200.y.129
$ iptables -t nat -A POSTROUTING -s 10.200.y.129/32 -o eth0 -j SNAT --to-source 10.0.x.13
$ iptables -t nat -A PREROUTING -d 10.0.x.13/32 -j DNAT --to-destination 10.200.y.129
```

## Step 3: Configure Port Forwarding

Same as DNAT, firstly add the IP address to the Linux Router, then add iptables rules.

For example, the Public IP is `10.0.x.14/20`, the public port is `22`. The Internal IP (e.g. VM) is `10.200.y.213`, the internal port is `22` (ssh)

```
$ ip addr add dev eth0 10.0.x.14/20

$ iptables -A PREROUTING -d 10.0.x.14/32 -i eth1 -p tcp -m tcp --dport 22 -j DNAT --to-destination 10.200.y.213:22
$ iptables -t nat -A PREROUTING -d 10.0.x.14/32 -i eth1 -p tcp -m tcp --dport 22 -j DNAT --to-destination 10.200.y.213:22
$ iptables -t nat -A PREROUTING -d 10.0.x.14/32 -i eth0 -p tcp -m tcp --dport 22 -j DNAT --to-destination 10.200.y.213:22
```

## Summary

It is very easy to setup Destination NAT and Port forwarding in a Linux router. 

With Destination NAT, we can access all ports of an internal server (e.g. VM). 

With Port forwarding, we can access a specific port of an internal server (e.g. VM). We can configure multiple port forwarding rules for different internal servers (e.g. VMs) with different public ports. 

Based on use cases, we can choose Destination NAT or Port forwarding.
