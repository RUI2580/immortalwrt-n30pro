# Netcore N30 Pro (MT7981B) 定制固件

为 **磊科 Netcore N30 Pro**（硬件同 netis NX35U / netis NX30 V2 系列）定制的 ImmortalWrt 固件。
在 OpenWrt 官方 `netis_nx30v2` 基础上重新移植了设备定义，补齐了原版缺失的 **4 网口 + USB3.0** 支持。

> English documentation: [README.en.md](README.en.md)

## 硬件规格

| 项目 | 规格 |
| --- | --- |
| SoC | MediaTek MT7981B (Filogic 820) |
| Flash | 128 MB SPI-NAND (ESMT F50L1G41LB) |
| RAM | DDR3（以实机 device-tree 为准） |
| 有线网口 | 4 × 1G LAN + 1 × 1G WAN |
| 无线 | MT7981 2.4G + 5G (Wi-Fi 6) |
| USB | 1 × USB 3.0，带 5V VBUS 供电控制 |
| 按键 | Reset / WPS |

## 功能特性

### 网口

- **4 个 LAN 口**（lan1–lan4）全部桥接进 br-lan
- **1 个 WAN 口**（eth1），默认 DHCP
- LAN 地址 192.168.1.1/24

### USB 网络共享（Tethering）

支持手机 / 随身 WiFi 通过 USB 提供网络，覆盖主流共享协议：

| 驱动 | 适用设备 |
| --- | --- |
| rndis_host | Android 手机 / 随身 WiFi |
| cdc_ether | Linux、macOS |
| cdc_ncm | iPhone、部分 Windows 手机 |
| cdc_eem | 部分 4G 模块 |

插上设备后由 `/etc/hotplug.d/net/20-usbwan` 自动完成：

1. 识别 USB 网卡类型
2. 创建 usbwan 接口（DHCP）并绑定到对应网卡设备
3. 把 usbwan 加入 wan 防火墙 zone，启用 NAT
4. 触发 DHCP 获取 IPv4 / IPv6

接口优先级：usbwan metric = 20，低于 WAN，有线插上时优先用有线。

### USB 存储

- usb-storage / uas 块设备支持
- 文件系统：vfat / ext4 / exfat / ntfs3
- 通过 block-mount + automount 自动挂载到 /mnt

### 附加组件

| 组件 | 作用 |
| --- | --- |
| usb-vbus | 通过 GPIO 535（SoC base 512 + pin 23）控制 USB 5V 供电 |
| ttlfix | 通过 nftables 修正 USB 共享接口的 TTL，避免被运营商识别为热点 |
| usbwan-watchdog | 每分钟检测连通性，连续 3 次失败后复位 USB VBUS 并重载网卡驱动 |

## 下载与安装

从 [Releases](../../releases) 页面下载对应版本。每个版本包含四个文件：

| 文件 | 用途 |
| --- | --- |
| *-squashfs-sysupgrade.itb | 主固件，系统内升级用 |
| *-initramfs.itb | 内存运行镜像，救砖 / U-Boot TFTP 引导用 |
| *-spim-nand-bl31-uboot.fip | Bootloader (FIP)，仅首次从原厂刷入时需要 |
| *-spim-nand-preloader.bin | Bootloader (BL2)，仅首次从原厂刷入时需要 |

下载后先校验完整性：

```bash
sha256sum -c SHA256SUMS
```

### 情况 A：设备已经是 ImmortalWrt / OpenWrt

```bash
# 上传固件
scp *-squashfs-sysupgrade.itb root@192.168.1.1:/tmp/

# 升级（-n 表示不保留旧配置，跨 profile 升级必须加）
ssh root@192.168.1.1
sysupgrade -n /tmp/*-squashfs-sysupgrade.itb
```

约 1 分钟重启完成。也可以在 LuCI 网页里操作：**系统 → 备份/升级 → 刷写固件**，取消勾选「保留配置」。

### 情况 B：设备还是原厂固件

需要先写入 Bootloader，参考 OpenWrt 官方提交文档（commit 344bb7f9），简要流程：

1. PC 设静态 IP 192.168.1.254，架设 TFTP 服务，把 preloader.bin、bl31-uboot.fip、initramfs.itb 放进 TFTP 根目录
2. ssh 登录原厂系统（用户名 useradmin，密码为网页登录密码）
3. **先备份** BL2、FIP、Factory 分区与 customer 卷（回退原厂的唯一凭据）
4. 写入并擦除：

```bash
cd /tmp
tftp -r *-preloader.bin -g 192.168.1.254
tftp -r *-bl31-uboot.fip -g 192.168.1.254
mtd write *-preloader.bin spi0.0
mtd write *-bl31-uboot.fip FIP
mtd erase ubi
```

5. PC 保持 192.168.1.254，重启路由器，U-Boot 会自动从 TFTP 拉取 initramfs 启动
6. 进入系统后，用 LuCI 上传 squashfs-sysupgrade.itb 完成最终刷写

> **警告**：执行 mtd erase ubi 前必须确认备份文件已经存到 PC 上。

## 默认登录信息

| 方式 | 地址 | 用户名 | 密码 |
| --- | --- | --- | --- |
| LuCI | http://192.168.1.1 | root | 首次为空，请立即设置 |
| SSH | ssh root@192.168.1.1 | root | 同上 |

## 自行编译

本仓库带有 GitHub Actions 工作流，push 到 master 即自动构建，也可在 Actions 页面手动触发。

触发条件：修改以下路径会自动触发构建

```
target/linux/mediatek/**
package/custom/**
package/libs/**
package/boot/**
files/**
ci/**
.github/workflows/build.yml
```

构建参数：TARGET = mediatek/filogic，PROFILE = netcore_n30pro，构建约 2.5 小时（4 核 runner）。

本地编译：

```bash
git clone https://github.com/RUI2580/immortalwrt-n30pro.git
cd immortalwrt-n30pro
./scripts/feeds update -a && ./scripts/feeds install -a
make defconfig
make -j$(nproc)
```

## 设备定义说明

本项目相对上游的关键改动：

### 1. 独立设备 profile

上游 netis_nx30v2 只声明了 3 个 LAN 口且无 USB 节点，无法满足 4 口 + USB 需求。
因此新增 netcore_n30pro profile，对应 DTS 为 mt7981b-netcore-n30pro.dts：

- switch 端口声明 lan1–lan4
- &usb_phy 与 &xhci 使能，vbus-supply 指向 reg_5v
- reg_5v 为 GPIO 23 控制的固定 5V 稳压器

### 2. U-Boot 支持

- 新增 U-Boot/mt7981_netcore_n30pro，UBOOT_CONFIG := mt7981_netcore_n30pro
- 新增补丁 package/boot/uboot-mediatek/patches/505-add-netcore-n30pro.patch，包含 U-Boot DTS、defconfig、默认环境变量

### 3. libffi 兼容修复

libffi 3.8.0 用 AC_CONFIG_HEADERS([fficonfig.h]) 把 fficonfig.h 生成在构建根目录，
而上游 InstallDev 仍在找旧布局的 $(GNU_TARGET_NAME)*/fficonfig.h，导致编译失败。
本项目改为两种位置都尝试，缺失不致命。

## 目录结构

```
.github/workflows/build.yml                  CI 构建
ci/config-n30pro                             配置参考
files/etc/hotplug.d/net/20-usbwan            USB 共享自动接管
files/etc/uci-defaults/30-n30pro-network     首次开机网络默认值
package/custom/ttlfix                        TTL 修正
package/custom/usb-vbus                      USB 5V 供电控制
package/custom/usbwan-watchdog               USB 共享看门狗
target/linux/mediatek/dts/mt7981b-netcore-n30pro.dts   设备树
target/linux/mediatek/image/filogic.mk                 设备定义
```

## 已知限制

- 仅在 mediatek/filogic 目标下验证
- USB 共享的接口自动接管依赖 hotplug 事件，个别设备可能需要手动 ubus call network.interface.usbwan up
- 无线部分沿用上游 MT7981 驱动，未做额外调优

## 参考

- [OpenWrt 官方设备支持提交](https://github.com/openwrt/openwrt/commit/344bb7f9162b76a864cec1ebd069651a9c2483cf)
- [ImmortalWrt 上游仓库](https://github.com/immortalwrt/immortalwrt)

## 许可

本项目基于 ImmortalWrt，遵循其原始许可（GPL-2.0 及组件各自许可）。
