---
layout: post
title: Wiring CloudStack KVM hosts up to OVN
toc: true
categories: [CloudStack, KVM, OVN, Networking]
---

CloudStack's classic KVM guest networking story is "one Linux bridge or OVS bridge per VLAN, per host". That works, but it doesn't scale well once you want overlay networks, distributed routing, or anything resembling a modern SDN control plane. [OVN](https://www.ovn.org/) (Open Virtual Network) gives you that control plane on top of Open vSwitch: a central Northbound/Southbound database pair plus `ovn-controller` agents on every hypervisor, all talking Geneve to each other.

This post walks through wiring a small CloudStack KVM setup up to OVN: one dedicated controller node, a handful of Ubuntu KVM hosts, and the small CloudStack-side changes needed to make the Guest network actually land on OVN's integration bridge instead of a plain OVS bridge.

<!--more-->

## Layout

- **Controller**: a dedicated node (`mgmt2` in this environment) running `ovn-central` - `ovn-northd`, plus the NB and SB ovsdb-servers. Keeping this off the primary CloudStack management server means an OVN restart or upgrade can't take the API down with it.
- **Compute nodes**: every KVM host runs `ovn-host` + `openvswitch-switch`, joins the Southbound DB, and encapsulates guest traffic over Geneve.
- **Uplink**: guest traffic reaches the outside world through a `physnet1` provider network mapped to the existing `cloudbr1` OVS bridge on each host - the same bridge used for the (non-OVN) public network today.

The one detail that trips people up: OVN needs a routable IP to encapsulate Geneve traffic *from*, and on these hosts `eth0`/`eth1` sit on the untagged management/storage segments. The tagged VLAN sub-interface `eth1.2000` is the one that's actually reachable between hosts across racks, so that's the address OVN's `ovn-encap-ip` has to point at - not `eth1` directly.

## Step A - controller setup

```bash
apt install ovn-central ovn-common

systemctl enable --now ovn-northd
systemctl enable --now ovn-ovsdb-server-sb
systemctl enable --now ovn-ovsdb-server-nb

# controller IP = 10.0.0.10 (cloudbr0 on mgmt2)
ovn-nbctl set-connection ptcp:6641:10.0.0.10
ovn-sbctl set-connection ptcp:6642:10.0.0.10
```

`ovn-central` pulls in `ovn-common` as a dependency anyway, but it doesn't hurt to ask for both explicitly. Don't forget to open 6641/6642 (tcp) to the compute subnet - `ovn-nbctl`/`ovn-sbctl` will happily bind the socket and then silently reject everyone if the firewall is in the way.

## Step B/C - KVM host setup (every compute node)

```bash
apt install ovn-host openvswitch-switch libvirt-daemon-system

ovs-vsctl set open_vswitch . external_ids:system-id=$(hostname)

ovs-vsctl set open_vswitch . \
  external_ids:ovn-remote="tcp:10.0.0.10:6642"

ovs-vsctl set open_vswitch . \
  external_ids:ovn-encap-type="geneve" \
  external_ids:ovn-encap-ip="$(ip -4 addr show eth1.2000 | awk '/inet/ {print $2}' | cut -d/ -f1)"

ovs-vsctl set open_vswitch . \
  external_ids:ovn-bridge-mappings="physnet1:cloudbr1"

systemctl enable --now ovn-controller
```

Open UDP/6081 (Geneve) between chassis - it's easy to open the OVN DB ports and forget the data-plane tunnel port, and then wonder why `ovn-sbctl show` lists the chassis but no traffic actually flows.

### Why `eth1.2000`, and what the netplan looks like

Every host here has two NICs: `eth0` for management (`cloudbr0`) and `eth1`, which we already put behind the OVS bridge `cloudbr1` for the public network. Neither of those is the right place to hang OVN's Geneve tunnels off: `eth0`'s management segment isn't routed to where the other chassis live, and `cloudbr1` itself has no IP - it's a bridge, not a host interface, so it can't be `ovn-encap-ip` either.

What *is* routed between every host is a trunked VLAN carried over `eth1` - VLAN 2000 in this environment. So instead of inventing a new cable or bridge just for OVN, we add a tagged sub-interface, `eth1.2000`, on top of the same physical NIC that already carries `cloudbr1`, and give only that sub-interface an IP address. `cloudbr1` keeps using `eth1` untagged for public traffic exactly as before - the VLAN interface just rides alongside it on the same wire.

With netplan that's a few extra lines next to the existing bridge definitions:

```yaml
network:
  ethernets:
    eth1:
      dhcp4: no
  bridges:
    cloudbr1:
      dhcp4: no
      interfaces: [eth1]
      openvswitch: {}
  vlans:
    eth1.2000:
      id: 2000
      link: eth1
      addresses:
        - 10.250.x.x/20
  version: 2
```

`link: eth1` is what ties the VLAN to the physical NIC without disturbing `cloudbr1`'s own use of `eth1`; the `addresses` line is the only IP configuration OVN actually needs. Once `netplan apply` brings the interface up, `ovn-encap-ip` is read straight off it:

```bash
ovs-vsctl set open_vswitch . external_ids:ovn-encap-ip="$(ip -4 addr show eth1.2000 | awk '/inet/ {print $2}' | cut -d/ -f1)"
```

Skip this and point `ovn-encap-ip` at `eth1` (or `cloudbr1`) directly, and chassis either get no address to encapsulate from or pick up an address on a segment the other hosts can't reach - `ovn-sbctl show` will still list the chassis, but Geneve traffic between them silently goes nowhere.

## Step E - provider network (controller)

```bash
ovn-nbctl ls-add public
ovn-nbctl lsp-add public public-external
ovn-nbctl lsp-set-type public-external localnet
ovn-nbctl lsp-set-addresses public-external unknown
ovn-nbctl lsp-set-options public-external network_name=physnet1
```

Worth using `--may-exist`/`--if-exists` on all of these (`ovn-nbctl --may-exist ls-add public`, etc.) once you automate them - unlike most `ovs-vsctl` commands, the plain forms are not idempotent and will fail a second run.

## Step F - verify

```bash
# on the controller
ovn-sbctl show                      # one chassis entry per KVM host

# on a compute node
ovs-vsctl get open_vswitch . external_ids
#  ovn-remote, ovn-encap-ip, ovn-encap-type, ovn-bridge-mappings all present
```

## Step G - the CloudStack side

This is the part that's easy to get backwards. OVN creates its own integration bridge, `br-int`, on every host the moment `ovn-controller` starts - you don't create it, and you must not point CloudStack's guest network at `cloudbr1` any more once OVN is in the picture, because `cloudbr1` is now just the *uplink* bridge for the provider network, not where VM taps get plugged in.

**On the management server:**

```bash
apt-get install -y python3 python3-ovsdbapp jq
```

...and change the KVM traffic label for the **Guest** network from `cloudbr1` to `br-int`.

**On the KVM host**, `agent.properties` ends up with:

```
private.network.device=cloudbr0
public.network.device=cloudbr1
guest.network.device=br-int
network.bridge.type=openvswitch
```

`network.bridge.type=openvswitch` was already required for plain-OVS mode (it's what makes the agent load `OvsVifDriver`); the only change for OVN is `guest.network.device`, which now points at OVN's bridge instead of ours.

From here, the CloudStack OVN network extension (see the [CloudStack documentation](https://docs.cloudstack.apache.org/) for the extension framework) is what actually talks NB/SB via `ovsdbapp` to create logical switches/ports as networks and NICs come and go - Steps A-F just get the underlying OVN fabric to a state where that extension has something to drive.
