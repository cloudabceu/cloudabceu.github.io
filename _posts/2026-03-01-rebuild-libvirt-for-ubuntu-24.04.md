---
layout: post
title: Rebuilding libvirt on Ubuntu 24.04 with CLVM + sanlock patch
toc: true
categories: [Linux, Libvirt]
---

This post documents how to fetch, patch and rebuild `libvirt` (libvirt 10.0.0) on Ubuntu 24.04. The instructions show how to obtain the original source (Launchpad / Ubuntu archives), install build dependencies, apply local patches, and produce installable .deb packages.

The upstream Ubuntu libvirt source and recent packaging patches are available on Launchpad: https://launchpad.net/ubuntu/+source/libvirt

<!--more-->

## Background

Ubuntu publishes libvirt source packages via Launchpad. Rebuilding the package locally lets you test small runtime fixes, patches for clustered LVM workflows (CLVM + `sanlock` / `lvmlockd`), or other custom changes while keeping the Debian/Ubuntu packaging intact.

If you're coming from other distributions, the high-level flow is similar: get the source, apply patch, build a binary package, then install. Debian/Ubuntu packaging applies patch files listed in `debian/patches/series`. You don't need to use `quilt` — simply place your patch under `debian/patches/` and update the `series` file. Working directly with the `debian/` tree keeps your changes reproducible for package rebuilds and uploads.

This post complements a companion guide for Oracle Linux 9 where I rebuilt `libvirt-daemon-driver-storage-logical` and applied a CLVM + `sanlock` patch to handle LV lock conflicts during logical pool activation. See [Rebuilding libvirt on Oracle Linux 9 with CLVM + sanlock patch]({% post_url 2026-02-28-rebuild-libvirt-for-oracle-linux-9 %}) for the original patch rationale and an example `vgchange` failure message.

## Prerequisites

On Ubuntu 24.04 install the usual packaging toolchain and build deps:

```bash
sudo apt update
sudo apt install -y build-essential devscripts dpkg-dev fakeroot git curl
sudo apt install -y dh-make dh-sequence debhelper pkg-config
```

You'll also need the specific build-dependencies for libvirt. The easiest way is to install them from the archive once you have the source directory:

```bash
# in the source directory (see next step)
sudo apt-get build-dep -y libvirt
```

## Fetch the libvirt source (from Launchpad / Ubuntu archive)

You can fetch the source package from Launchpad or via `apt source` (see options below).

Download from Launchpad (recommended):

- Using `dget` (from `devscripts`) to fetch and unpack the `.dsc` file:

```bash
# install dget if needed
sudo apt install -y devscripts
# fetch and unpack the source package (example .dsc URL from the package page)
dget -xu https://launchpad.net/ubuntu/+archive/primary/+sourcefiles/libvirt/10.0.0-2ubuntu8.12/libvirt_10.0.0-2ubuntu8.12.dsc
```

- Or manually download the `.dsc` and tarballs and extract with `dpkg-source`:


Use `apt source` (alternative — convenient when `deb-src` entries are enabled):

```bash
# enable deb-src in /etc/apt/sources.list (if necessary) and update
sudo sed -n '1,200p' /etc/apt/sources.list
# then
sudo apt update
apt source libvirt
# or request the exact version
# apt source libvirt=10.0.0-2ubuntu8.12
```

Both approaches produce the same unpacked source directory (e.g. `libvirt-10.0.0/`) ready for patching.

### Create a `deb-src` sources file (example)

If you prefer to add a `deb-src` sources file instead of editing existing deb822 files, create `/etc/apt/sources.list.d/ubuntu-debsources.sources` (note the `.sources` extension) with the following content (example file included in this repo at [assets/ubuntu-debsources.sources](assets/ubuntu-debsources.sources)):

```bash
sudo tee /etc/apt/sources.list.d/ubuntu-debsources.sources > /dev/null <<'EOF'
Types: deb-src
URIs: http://nl.archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb-src
URIs: http://security.ubuntu.com/ubuntu/
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF

sudo apt update
sudo apt-get build-dep -y libvirt
```

Remove the file once you're done if you prefer:

```bash
sudo rm /etc/apt/sources.list.d/ubuntu-debsources.sources && sudo apt update
```

## Prepare the build environment

Debian/Ubuntu packaging applies patch files listed in `debian/patches/series`. You don't need to use `quilt` — simply place your patch under `debian/patches/` and update the `series` file. Working directly with the `debian/` tree keeps your changes reproducible for package rebuilds and uploads.

## Apply your patch

To apply a patch to the DEB source tree, add it to `debian/patches` (preferred for reproducible packaging):

```bash
# copy your patch file into the source tree
cp ~/my-patches/clvm-sanlock.patch debian/patches/
# add to series (ensure correct ordering)
echo "clvm-sanlock.patch" >> debian/patches/series
```

## Update debian/changelog

Debian/Ubuntu policy expects the changelog to reflect local changes. Use `dch` from `devscripts` to add an entry for your rebuild:

```bash
dch -i "Apply CLVM/sanlock LV activation patch: shared VG visibility, exclusive LV activation for VM disks"
```

This ensures package versioning and changelog information are correct for building and later uploads.

## Build the package

Use `debuild` or `dpkg-buildpackage` to build the binary packages. A typical local build step:

```bash
# from inside libvirt-10.0.0/
debuild -us -uc -b
# or
# dpkg-buildpackage -us -uc -b
```

`-us -uc` skip signing for quick local builds. The `-b` switch builds binary packages only.

When the build completes successfully, resulting `.deb` files will be in the parent directory (`../`).

## Install and test

Install the rebuilt packages with `dpkg -i` (you may need to restart services):

```bash
cd ..
sudo dpkg -i libvirt-daemon* libvirt0* libvirt-clients* || sudo apt -f install -y
sudo systemctl restart libvirtd
```

Check the journal and libvirt logs for errors:

```bash
journalctl -u libvirtd -b --no-pager
sudo tail -n 200 /var/log/libvirt/libvirtd.log
```

Validate the specific CLVM or LV activation behavior: try creating/activating logical pools, creating VM LVs while another host holds locks, and observe whether `vgchange`/`lvchange` calls are invoked with the intended activation flags.

## Notes and troubleshooting

- If packaging scripts or tests fail, first try building with `DEB_BUILD_OPTIONS=nocheck` to skip runtime tests while debugging packaging issues.

```bash
DEB_BUILD_OPTIONS=nocheck debuild -us -uc -b
```

- Ubuntu packages sometimes split libvirt into multiple binary packages; ensure you rebuild and install the relevant `libvirt-daemon` and `libvirt-daemon-driver-storage-logical` packages.

- If you maintain multiple patches, prefer keeping them under `debian/patches` with a clear `series` ordering; this makes uploads and reproduction simpler.

- For source control, consider importing the upstream source into a `git` repo and using `git-buildpackage` or `gbp` for finer control over patch branches and uploads.

## References

- Launchpad libvirt source: https://launchpad.net/ubuntu/+source/libvirt
- Debian packaging with quilt: https://www.debian.org/doc/manuals/maint-guide/dpatch.html
- `debuild` / `dpkg-buildpackage` docs: `man debuild`, `man dpkg-buildpackage`
