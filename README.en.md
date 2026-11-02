# Custom Firmware for Netcore N30 Pro (MT7981B)

An ImmortalWrt firmware built specifically for the **Netcore N30 Pro** router
(same hardware family as netis NX35U / netis NX30 V2).
The device profile was re-ported on top of OpenWrt's upstream `netis_nx30v2`
to add the **4 Ethernet ports + USB 3.0** support missing from upstream.

> 中文文档: [README.md](README.md)

## Hardware Specification

| Item | Specification |
| --- | --- |
| SoC | MediaTek MT7981B (Filogic 820) |
| Flash | 128 MB SPI-NAND (ESMT F50L1G41LB) |
| RAM | DDR3 (refer to the device tree of your unit) |
| Ethernet | 4 × 1G LAN + 1 × 1G WAN |
| Wireless | MT7981 2.4 GHz + 5 GHz (Wi-Fi 6) |
| USB | 1 × USB 3.0 with GPIO-controlled 5V VBUS |
| Buttons | Reset / WPS |

## Features

### Ethernet

- **4 LAN ports** (`lan1`–`lan4`), all bridged into `br-lan`
- **1 WAN port** (`eth1`), DHCP by default
- LAN address `192.168.1.1/24`

### USB Tethering

Phones and portable Wi-Fi dongles can supply connectivity over USB.
All common tethering protocols are supported:

| Driver | Devices |
| --- | --- |
| `rndis_host` | Android phones / portable Wi-Fi |
| `cdc_ether` | Linux, macOS |
| `cdc_ncm` | iPhone, some Windows phones |
| `cdc_eem` | Some 4G modules |

When a device is plugged in, `/etc/hotplug.d/net/20-usbwan` automatically:

1. Detects the USB netdev type
2. Creates the `usbwan` interface (DHCP) and binds it to that netdev
3. Adds `usbwan` to the `wan` firewall zone so NAT applies
4. Triggers DHCP for IPv4 / IPv6

Priority: `usbwan` uses metric 20, lower than wired WAN, so wired wins when plugged in.

### USB Storage

- `usb-storage` / `uas` block device support
- Filesystems: `vfat` / `ext4` / `exfat` / `ntfs3`
- Auto-mounted under `/mnt` via `block-mount` + `automount`

### Extra Components

| Component | Purpose |
| --- | --- |
| `usb-vbus` | Controls USB 5V supply via GPIO 535 (SoC base 512 + pin 23) |
| `ttlfix` | Rewrites TTL on tethering interfaces with nftables so carriers do not flag the device as a hotspot |
| `usbwan-watchdog` | Checks connectivity every minute; after 3 consecutive failures it power-cycles USB VBUS and reloads the net drivers |

## Download and Install

Grab a build from the [Releases](../../releases) page. Each release contains four files:

| File | Purpose |
| --- | --- |
| `*-squashfs-sysupgrade.itb` | Main firmware, used for in-system upgrades |
| `*-initramfs.itb` | RAM-only image, for recovery or U-Boot TFTP boot |
| `*-spim-nand-bl31-uboot.fip` | Bootloader (FIP), only needed for the first flash from stock firmware |
| `*-spim-nand-preloader.bin` | Bootloader (BL2), only needed for the first flash from stock firmware |

Verify integrity before flashing:

```bash
sha256sum -c SHA256SUMS
```

### Case A: The device already runs ImmortalWrt / OpenWrt

```bash
# Upload the firmware
scp *-squashfs-sysupgrade.itb root@192.168.1.1:/tmp/

# Upgrade (-n means do not keep the old configuration; required when
# the device profile changes)
ssh root@192.168.1.1
sysupgrade -n /tmp/*-squashfs-sysupgrade.itb
```

The router reboots in about a minute. You can also use LuCI:
**System → Backup / Flash Firmware → Flash image**, and uncheck **Keep settings**.

### Case B: The device still runs vendor firmware

The bootloader must be replaced first. See the upstream OpenWrt commit
(commit 344bb7f9) for the authoritative procedure. Short version:

1. Set a static IP of `192.168.1.254` on your PC, run a TFTP server, and put
   `preloader.bin`, `bl31-uboot.fip` and `initramfs.itb` in the TFTP root
2. SSH into the vendor firmware (user `useradmin`, password is the web login password)
3. **Back up** the `BL2`, `FIP` and `Factory` partitions plus the `customer` volume first —
   they are your only way back to vendor firmware
4. Write the bootloader and erase the system volume:

```bash
cd /tmp
tftp -r *-preloader.bin -g 192.168.1.254
tftp -r *-bl31-uboot.fip -g 192.168.1.254
mtd write *-preloader.bin spi0.0
mtd write *-bl31-uboot.fip FIP
mtd erase ubi
```

5. Keep the PC at `192.168.1.254` and reboot the router; U-Boot will pull
   `initramfs.itb` over TFTP and boot it from RAM
6. Once booted, flash `squashfs-sysupgrade.itb` through LuCI to finish

> **Warning**: do not run `mtd erase ubi` until you have confirmed the backups
> are actually stored on your PC.

## Default Credentials

| Method | Address | User | Password |
| --- | --- | --- | --- |
| LuCI | http://192.168.1.1 | `root` | empty on first boot — set one immediately |
| SSH | `ssh root@192.168.1.1` | `root` | same as above |

## Building

This repository ships a GitHub Actions workflow. A push to `master` triggers a
build; you can also start one manually from the Actions tab.

Builds are triggered by changes under:

```
target/linux/mediatek/**
package/custom/**
package/libs/**
package/boot/**
files/**
ci/**
.github/workflows/build.yml
```

Build parameters: `TARGET = mediatek/filogic`, `PROFILE = netcore_n30pro`.
A full build takes roughly 2.5 hours on a 4-core runner.

### Local build

```bash
git clone https://github.com/RUI2580/immortalwrt-n30pro.git
cd immortalwrt-n30pro
./scripts/feeds update -a && ./scripts/feeds install -a
make defconfig
make -j$(nproc)
```

## Device Porting Notes

Key differences from upstream:

### 1. Dedicated device profile

Upstream `netis_nx30v2` declares only three LAN ports and has no USB node,
so it cannot provide 4 ports plus USB. This project adds the `netcore_n30pro`
profile backed by `mt7981b-netcore-n30pro.dts`:

- `switch` ports declared as `lan1`–`lan4`
- `&usb_phy` and `&xhci` enabled, with `vbus-supply` pointing at `reg_5v`
- `reg_5v` is a fixed 5V regulator controlled by GPIO 23

### 2. U-Boot support

- Adds `U-Boot/mt7981_netcore_n30pro` with `UBOOT_CONFIG := mt7981_netcore_n30pro`
- Adds `package/boot/uboot-mediatek/patches/505-add-netcore-n30pro.patch`,
  containing the U-Boot device tree, defconfig and default environment

### 3. libffi compatibility fix

libffi 3.8.0 generates `fficonfig.h` in the build root via
`AC_CONFIG_HEADERS([fficonfig.h])`, but upstream `InstallDev` still looks for the
old layout `$(GNU_TARGET_NAME)*/fficonfig.h`, which breaks the build.
This project tries both locations and treats a miss as non-fatal.

## Repository Layout

```
.github/workflows/build.yml                 CI build
ci/config-n30pro                            reference config
files/etc/hotplug.d/net/20-usbwan           USB tethering auto-setup
files/etc/uci-defaults/30-n30pro-network    first-boot network defaults
package/custom/ttlfix                       TTL correction
package/custom/usb-vbus                     USB 5V supply control
package/custom/usbwan-watchdog              tethering watchdog
target/linux/mediatek/dts/mt7981b-netcore-n30pro.dts   device tree
target/linux/mediatek/image/filogic.mk                 device definition
```

## Known Limitations

- Verified only on the `mediatek/filogic` target
- USB tethering auto-setup relies on hotplug events; on some devices you may
  need to run `ubus call network.interface.usbwan up` manually
- Wireless uses the upstream MT7981 driver without extra tuning

## References

- [Upstream OpenWrt device support commit](https://github.com/openwrt/openwrt/commit/344bb7f9162b76a864cec1ebd069651a9c2483cf)
- [ImmortalWrt upstream repository](https://github.com/immortalwrt/immortalwrt)

## License

This project is based on ImmortalWrt and follows its original licensing
(GPL-2.0 plus the individual licenses of bundled components).
