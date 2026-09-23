# RTL8812AU driver for Synology SA6400 / DSM 7.2.2

Build a Linux kernel module (`8812au.ko`) for:

* Synology SA6400
* DSM 7.2.2-72806
* DSM Update 9
* Kernel 5.10.55+
* x86_64
* Synology `epyc7002` kernel/toolchain
* USB Wi-Fi chipset RTL8812AU
* USB ID `0bda:8812`

The resulting module has been tested on a real SA6400-based system.

## Tested hardware

CPU:

```text
Intel N5105
```

Wi-Fi adapter:

```text
Realtek RTL8812AU
USB ID: 0bda:8812
```

The adapter is exposed by DSM as:

```text
eth80
```

after the driver is loaded.

## Verified result

The resulting module:

```text
8812au.ko
```

reports:

```text
vermagic: 5.10.55+ SMP mod_unload
```

and successfully loads on DSM:

```text
Linux 5.10.55+
synology_epyc7002_sa6400
```

The adapter supports 2.4 GHz / 5 GHz managed mode and 802.11ac on the tested system.

## Why this repository exists

The upstream RTL8812AU driver can normally be built against a standard Linux kernel.

Synology DSM is different because the running kernel is a Synology-specific 5.10.55 build and the matching kernel source, configuration and cross-toolchain must be used.

This repository records the exact build environment required to reproduce the working module.

It does **not** require the Synology kernel source or ToolChain to be committed into this repository. The build script downloads the required Synology GPL source and ToolChain.

## Pinned versions

### Synology

```text
DSM:          7.2.2-72806 Update 9
Kernel:       5.10.55+
Platform:     epyc7002
Architecture: x86_64
```

### Synology ToolChain

```text
epyc7002-gcc1220_glibc236_x86_64-GPL.txz
GCC: 12.2.0
```

MD5:

```text
117ae882569c315db4b19e5426f50eb9
```

### Kernel source

```text
linux-5.10.x
Kernel version: 5.10.55
Synology configuration: synoconfigs/epyc7002
```

### RTL8812AU source

Upstream:

```text
https://github.com/morrownr/8812au-20210820
```

The build uses a pinned commit rather than `main`, so future upstream changes do not silently change the resulting module.

## Build

Recommended build environment:

* Linux x86_64
* Podman or Docker
* Internet access
* At least 4 GB free disk space

### Option 1: container build

Build the container:

```bash
podman build -t synology-sa6400-8812au .
```

Run:

```bash
podman run --rm -it \
  -v "$PWD:/work:Z" \
  synology-sa6400-8812au \
  /work/build.sh
```

The resulting module will be written to:

```text
dist/8812au.ko
```

### Option 2: native build

See `scripts/build-module.sh`.

The native build requires:

```text
gcc
make
bc
bison
flex
git
wget/curl
xz
tar
gzip
bzip2
rsync
cpio
kmod
libelf-dev
libssl-dev
pkg-config
python3
perl
file
```

## Build configuration

The base Synology configuration is:

```text
synology/synoconfigs/epyc7002
```

The following kernel options are enabled for the wireless stack:

```config
CONFIG_WIRELESS=y
CONFIG_CFG80211=m
CONFIG_MAC80211=m
CONFIG_RFKILL=y
CONFIG_WLAN=y
CONFIG_LOCALVERSION="+"
CONFIG_LOCALVERSION_AUTO=n
```

The existing DSM kernel already contains compatible `cfg80211`, `mac80211`, `rfkill` and `libarc4` modules. Therefore this project only builds the RTL8812AU external module.

## Kbuild compatibility fixes

The Synology 5.10.55 source tree requires an additional fix before building the host `objtool` with the modern GCC toolchain.

The repository contains the patch:

```text
patches/0001-libsubcmd-fix-use-after-free.patch
```

It modifies:

```text
tools/lib/subcmd/subcmd-util.h
```

The patch corresponds to the upstream Linux `libsubcmd` fix for the `realloc(..., 0)` use-after-free issue.

The build script applies this patch automatically.

## Important build detail

Do not run a full `make modules_prepare` on this Synology source tree.

For this build, the required preparation is performed explicitly:

```text
make prepare
make tools/objtool
make archheaders
generate scripts/module.lds
```

The build script handles these steps automatically.

## Build output

After a successful build:

```text
dist/8812au.ko
```

Check:

```bash
modinfo dist/8812au.ko
```

Expected:

```text
vermagic: 5.10.55+ SMP mod_unload
```

The module should contain the USB alias:

```text
usb:v0BDAp8812d*
```

## Install on DSM

Copy the module to the NAS:

```bash
scp dist/8812au.ko root@NAS:/tmp/
```

Load:

```bash
insmod /tmp/8812au.ko
```

Check:

```bash
lsmod | grep 8812au
iw dev
dmesg | tail -50
```

The tested adapter appears as:

```text
eth80
```

after Synology/RR network management renames the interface.

## RR Wireless configuration

This project only provides the kernel driver.

For RR Wireless support, enable the RR wireless option and use:

```bash
/usr/bin/wireless_supplicant.sh "*" "SSID" "PSK"
```

RR's script starts `wpa_supplicant` and then invokes:

```text
/usr/syno/sbin/synonet --dhcp
```

## Synology routing

For IPv4 and IPv6 policy routing, enable DSM:

```text
Control Panel
→ Network
→ General
→ Advanced Settings
→ Enable Multiple Gateway
```

This allows DSM's native policy-routing mechanism to create the per-interface routing table used by `eth80`.

This repository does not add custom `ip rule` or `ip route` startup scripts.

## Reproducibility

The following components are intentionally pinned:

* Synology DSM release
* Synology kernel source
* Synology ToolChain
* GCC version
* RTL8812AU source commit
* kernel configuration fragment
* required Kbuild patch

Changing any of these may produce a different module or may require additional compatibility changes.

## License

The RTL8812AU driver source is derived from the upstream GPL-licensed project.

See the upstream repository and its license files:

https://github.com/morrownr/8812au-20210820

Synology kernel sources and toolchains are provided by Synology under their respective GPL/source licenses.

This repository contains build scripts, configuration fragments and patches rather than a copy of the Synology source tree.
