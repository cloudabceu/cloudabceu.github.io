---
layout: post
title: CloudStack HowTo - Create Debian 13 Cloud images for VMware and XenServer testing
toc: true
categories: [CloudStack]
---

Debian 13 (Trixie) cloud images are official Debian images, pre-installed and customized by the Debian Cloud Team for use with cloud-init on IaaS platforms and generic virtualised environments.

This article introduces how to create Debian 13 cloud images for testing on VMware and XenServer, other than KVM.
<!--more-->

## Debian 13 Cloud Images

Debian publishes several image variants for each release, all built from the same `cdimage.debian.org` cloud image directory:

- `generic` - a full set of kernel drivers, suitable for OpenStack, bare metal, and hypervisors with less common virtual hardware (VMware, Xen).
- `genericcloud` - the same image with a reduced driver set, trimmed for large-scale KVM/OpenStack clouds. Debian's own docs say plainly: "if it does not work for your use case, you should use the generic images" - which is exactly our case here, since we want the image to also boot on VMware and XenServer.
- `nocloud`, `azure`, `ec2`, `raspi` - platform-specific variants we don't need for this.

Because we're targeting VMware and XenServer as well as KVM, we use the **generic** variant, not `genericcloud`.

For Debian 13 (trixie), the images are located at https://cdimage.debian.org/images/cloud/trixie/latest/

You can download the qcow2 generic cloud image for the latest Debian 13 release by

```
# amd64 (x86_64)
wget https://cdimage.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2

# arm64
wget https://cdimage.debian.org/images/cloud/trixie/latest/debian-13-generic-arm64.qcow2
```

The default login user baked into these images is `debian` (not `ubuntu`, `root`, or `admin`) - keep that in mind when testing SSH access after boot.

## Register the Debian 13 template on CloudStack (KVM)

Before we can customize the image, we register it as a KVM template so CloudStack can deploy a VM from it. Using `cloudmonkey`:

```
register template name="debian-13-generic-amd64" \
    displaytext="Debian 13 Generic" \
    format=QCOW2 \
    hypervisor=KVM \
    ostypeid=<debian-12-or-11-ostype-uuid> \
    url="https://cdimage.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2" \
    zoneid=<zoneid>
```

Debian 13 may not yet be in the guest OS list on your CloudStack version - picking the closest Debian entry (e.g. Debian 12) is fine, it only affects display/optimisation hints, not what actually boots.

The generic image ships with a small root disk (a couple of GB), which isn't enough to comfortably install extra guest tools. Override the root disk size to 5 GB when deploying the VM below, rather than resizing the template itself.

## Create/register an SSH keypair

The cloud image has no password set for `debian` or `root` - the only way in is via SSH key injected by cloud-init. Create a keypair through CloudStack so it can be attached at deploy time:

```
create sshkeypair name=debian13-test
```

Save the returned private key to a file and lock down its permissions:

```
chmod 400 debian13-test.pem
```

(If you already manage keys outside CloudStack, use `register sshkeypair` with your existing public key instead.)

## Deploy a test VM from the template

```
deploy virtualmachine templateid=<templateid> \
    serviceofferingid=<serviceofferingid> \
    zoneid=<zoneid> \
    keypair=debian13-test \
    rootdisksize=5
```

## Expose the VM if it's on an isolated network

If the VM was deployed into an isolated network (guest VM has only a private IP, no public IP of its own), you need to expose it before you can SSH in from outside the network. Pick one of CloudStack's public IP services, depending on what the network offering supports:

- **Static NAT** - maps a public IP 1:1 to the VM, all ports pass through.
- **Port forwarding** - maps a single port (e.g. 22 for SSH) from a public IP to the VM.
- **Load balancer** - useful when testing more than one VM behind the same public IP/port, but overkill for a single test VM; static NAT or port forwarding is simpler here.

Whichever option you use, none of them open the port by themselves - the network's firewall/ACL still blocks it by default. Remember to also add a firewall (or network ACL, depending on network offering) rule allowing the port (e.g. TCP 22 for SSH), narrowed to your own IP/range where possible.

## Access the VM and install hypervisor guest tools

Once the VM is `Running` and reachable (directly if on a shared network, or via the public IP set up above if isolated), SSH in as `debian` using the private key - not `root`, and not a password:

```
ssh -i debian13-test.pem debian@<vm-ip-or-public-ip>
```

The `debian` user has full `sudo` privilege. The image ships with **no root password set** (root login is locked, not "a random password" you can recover), so if you want a usable `root` account for testing, set one explicitly:

```
sudo passwd
```

Since this same image will end up running on KVM, VMware, and XenServer, install the guest agent/tools for all three hypervisors now, in one pass:

```
sudo apt update
# for KVM: qemu-guest-agent
# for VMware: open-vm-tools
# for Xen: xenstore-utils libxenstore4
sudo apt install -y qemu-guest-agent open-vm-tools xenstore-utils libxenstore4
```

### Fix cloud-init datasource detection

As covered in the "Issue with cloud images on VMware" section below, `ds-identify` cannot reliably detect the CloudStack datasource on VMware/XenServer when multiple datasources are present. Fix it now, while we're already customizing the image:

```
# Enable cloud-init without any aid from ds-identify
echo "policy: enabled" > /etc/cloud/ds-identify.cfg
```

```
# /etc/cloud/cloud.cfg.d/cloudstack.cfg
cat > /etc/cloud/cloud.cfg.d/cloudstack.cfg <<EOF
datasource_list: [ ConfigDrive, CloudStack, None ]
datasource:
    CloudStack: {}
    None: {}
EOF
```

### Clean the image before it becomes a template

Before stopping the VM and turning its disk into templates for the other hypervisors, wipe the cloud-init state and instance-specific data so the image re-runs first-boot setup cleanly on every future deploy:

```
sudo cloud-init clean --machine-id -s
sudo rm -rf /var/log/*
rm ~/.ssh/authorized_keys
history -c
exit
```

Now stop the VM from CloudStack. The disk backing this VM (on KVM primary storage, or downloaded via "Create Template from Volume" → download template) is the customized qcow2 to use for the conversions below.

## Create Debian 13 Cloud images for XenServer

This can be done by qemu-img

```
# Convert to vhd
qemu-img convert -f qcow2 -O vpc debian-13-generic-amd64.qcow2 debian-13-generic-amd64.vhd
```

## Create Debian 13 Cloud images for VMware

Debian does not publish a ready-made `ova` image, so - just like with Ubuntu - we convert the qcow2 to vmdk first.

```
# Convert to vmdk
qemu-img convert -f qcow2 -O vmdk debian-13-generic-amd64.qcow2 debian-13-generic-amd64.vmdk
```

However, `vmdk` is not ready for use in Apache CloudStack, so we need to convert to `ova` format. A configuration file is required for the conversion. Create a file `debian-13-generic-amd64.vmx` with content below.

```
.encoding = "UTF-8"
displayname = "debian-13-generic-amd64"
guestos = "debian12-64"
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
scsi0:0.fileName = "debian-13-generic-amd64.vmdk"
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

(The vmx is adapted from the one used for Ubuntu cloud images - only `displayname`, `guestos` and `scsi0:0.fileName` need to change per OS/version.)

Note on `guestos`: VMware's guest-OS identifier list doesn't have a dedicated entry for Debian 13 (Trixie) yet at the time of writing, so `debian12-64` is used as the closest supported match - the guest still boots and runs fine, VMware just displays it as a Debian 12 guest in the UI. If your `ovftool`/ESXi version has since added `debian13-64`, use that instead.

We need a program `ovftool` for the next conversion. We can download it from VMware website for free on https://developer.broadcom.com/tools/open-virtualization-format-ovf-tool/latest .

Now run the command below
```
ovftool debian-13-generic-amd64.vmx debian-13-generic-amd64.ova
```

Now we can register a VMware template with the generated `debian-13-generic-amd64.ova` file.

## Issue with cloud images on VMware

Please note, for the vm instances running on VMware or XenServer/XCP-ng hypervisors, if there are multiple cloud-init data sources, it is a known issue that ds-identify is not able to detect if "CloudStack" DataSource is enabled. To fix the problem, please run the following command to enable cloud-init without any aid from ds-identify.

```
# Enable cloud-init without any aid from ds-identify
echo "policy: enabled" >  /etc/cloud/ds-identify.cfg
```

Then ask cloud-init to use the data sources defined.
```
# Add ConfigDrive to datasource_list
cat > /etc/cloud/cloud.cfg.d/cloudstack.cfg <<EOF
datasource_list: ['ConfigDrive', 'CloudStack']
datasource:
  CloudStack:
    max_wait: 120
    timeout: 50
EOF
```

This is exactly what we already applied while customizing the KVM VM above - it's called out here separately as this is the fix if you hit `ds-identify`/datasource detection failures on an image that wasn't pre-customized this way.

## Appendix: enabling root SSH login

Setting a root password with `sudo passwd` (above) is not enough on its own to let `root` log in over SSH with that password - a few things still block it, all put there deliberately by the cloud image/cloud-init:

- `sshd_config` restricts root login to key-only by default (`PermitRootLogin prohibit-password`), and cloud images typically also disable password authentication entirely. Edit `/etc/ssh/sshd_config` and allow both root login and password login:

  ```
  # /etc/ssh/sshd_config
  PermitRootLogin yes
  PasswordAuthentication yes
  ```

  Then restart sshd:

  ```
  sudo systemctl restart ssh
  ```

- cloud-init also locks out `root` independently by writing a `command="..."` restriction (or a full login-disabled banner) into `/root/.ssh/authorized_keys` alongside your injected key. Remove that file so root's SSH access isn't gated by it:

  ```
  sudo rm -f /root/.ssh/authorized_keys
  ```

With all of that done, `root` can log in over SSH using the password set with `sudo passwd`. Since this weakens the image's security posture (root + password login, open to the network), only do this on throwaway test images - never carry it into a template you intend to reuse for anything beyond hypervisor testing.
