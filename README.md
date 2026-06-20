# Xiaomi BE7000 OpenWrt Installer

OpenWrt installation files and flashing notes for the Xiaomi BE7000.

This repository is intended for installing OpenWrt while preserving the stock
Xiaomi firmware slot:

```text
rootfs    -> Xiaomi stock firmware, keep untouched
rootfs_1  -> OpenWrt installation and future sysupgrade target
```

After the first install, OpenWrt `sysupgrade` should keep using `rootfs_1`.
The stock `rootfs` slot remains available for rollback.
rootfs    → 小米原厂固件，请勿改动
rootfs_1  → OpenWrt系统分区，后续在线升级也将写入此分区
首次刷入后，执行 OpenWrt 系统升级命令sysupgrade会持续使用rootfs_1分区。
原厂固件分区rootfs完整保留，随时可回退原厂系统。
## Files

Expected release files:
发布包内含以下文件：
```text
openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-factory.ubi
openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-sysupgrade.bin
sha256sums
```

Use the images as follows:
文件用途区分：
```text
factory.ubi      → 在原厂固件SSH终端内手动刷机使用
sysupgrade.bin   → OpenWrt环境下升级（支持内存临时系统/网页管理LuCI）
factory.ubi      -> manual flashing from stock firmware shell
sysupgrade.bin   -> OpenWrt sysupgrade from OpenWrt/initramfs/LuCI
```

Do not write `sysupgrade.bin` directly to `rootfs` or `rootfs_1` from stock
firmware. The Xiaomi bootloader expects a UBI image in the selected firmware
slot.

## Requirements

You need one of these:

- root shell on stock Xiaomi firmware, for example SSH or telnet
- OpenWrt initramfs already booted on the router
- UART access, if recovery is needed

Without UART, check every command carefully before writing to flash.

## Verified Partition Layout
禁止在原厂固件下直接将 sysupgrade.bin 写入rootfs或rootfs_1分区。小米 Bootloader 引导程序仅识别标准 UBI 镜像作为系统分区镜像。
前置条件
需具备以下任意一种操作环境：
原厂小米固件获取 ROOT 终端（SSH/Telnet）
路由器已临时启动 OpenWrt 内存版（initramfs）
UART 串口调试工具（系统变砖恢复必备）
若无串口工具，向闪存写入数据前务必逐条核对所有命令。
已验证分区布局
```text
mtd17: 00080000 00020000 "0:APPSBLENV"
mtd23: 02800000 00020000 "rootfs"
mtd24: 02800000 00020000 "rootfs_1"
mtd28: 01c00000 00020000 "overlay"
```

Xiaomi boot flags are stored in `0:appsblenv`.

OpenWrt `fw_printenv/fw_setenv` config:
小米系统启动标记存储于分区0:appsblenv。
OpenWrt 读写启动环境变量工具fw_printenv/fw_setenv配置：
```text
/dev/mtd17 0x0 0x10000 0x20000
```

## Safe Install Layout

The stock-preserving installation expects stock firmware to be booted from
slot `0`:
安全安装分区逻辑
本保留原厂固件的刷机流程，要求路由器当前从0 号分区原厂固件启动：
```text
flag_boot_rootfs=0
ubi.mtd=rootfs
```

OpenWrt is then written only to `rootfs_1`:
OpenWrt 仅写入rootfs_1分区：
```text
target partition: mtd24 / rootfs_1
target slot:      1
```

If stock is currently booted from `rootfs_1` (`flag_boot_rootfs=1`), stop and
do not use this stock-preserving procedure.

## Check the Active Slot

On stock firmware:
若当前原厂固件运行在rootfs_1分区（参数flag_boot_rootfs=1），立即停止操作，不要使用本套保留原厂刷机流程。
查看当前启动槽位
在小米原厂固件终端执行：
```sh
nvram get flag_boot_rootfs
cat /proc/cmdline
cat /proc/mtd
```

Expected for this procedure:
符合刷机条件输出：
```text
flag_boot_rootfs=0
```

Cross-check the kernel command line:
核对内核启动参数确认：
```sh
sed -n 's/.*ubi.mtd=\([^ ]*\).*/\1/p' /proc/cmdline
```

Expected:
预期返回：
```text
rootfs
```

If the boot flag and `ubi.mtd=` disagree, stop and investigate before flashing.

## Install from Stock Firmware Shell

Copy the UBI image to the router:
若启动标记与内核挂载分区参数不一致，停止刷机并排查问题。
在原厂固件终端刷入 OpenWrt
将 UBI 镜像上传至路由器临时目录
```sh
scp -O openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-factory.ubi \
	root@192.168.31.1:/tmp/openwrt.ubi
```

Verify the current slot:
再次确认当前启动分区
```sh
nvram get flag_boot_rootfs
cat /proc/cmdline
cat /proc/mtd
```

Expected:
正常输出：
```text
flag_boot_rootfs=0
```

Flash OpenWrt to `rootfs_1`:
将 OpenWrt 写入 rootfs_1 分区
```sh
ubiformat /dev/mtd24 -y -f /tmp/openwrt.ubi
```

Switch Xiaomi boot flags to slot `1`:
修改小米引导标记，切换至 1 号分区启动
```sh
nvram set flag_boot_rootfs=1
nvram set flag_last_success=1
nvram set flag_boot_success=1
nvram set flag_try_sys1_failed=0
nvram set flag_try_sys2_failed=0
nvram set flag_ota_reboot=0
nvram commit
```

Reboot:
重启路由器
```sh
reboot
```

## Install or Upgrade from OpenWrt

Copy the sysupgrade image to the router:
OpenWrt 系统内升级
上传升级固件至路由器
```sh
scp openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-sysupgrade.bin \
	root@192.168.1.1:/tmp/openwrt.bin
```

Run sysupgrade:
执行升级（-n 不保留配置，按需删除参数）
```sh
sysupgrade -n /tmp/openwrt.bin
```

The BE7000 OpenWrt upgrade code writes the image to `rootfs_1`, strips
OpenWrt metadata before `ubiformat`, restores configuration, and sets Xiaomi
boot flags to slot `1`.

## Check After Boot

After OpenWrt boots:
BE7000 适配的 OpenWrt 升级脚本会自动完成：写入镜像至 rootfs_1、剔除 OpenWrt 头部元数据后格式化 UBI、保留自定义配置、同步小米分区启动标记为 1 号槽位。
刷机完成后校验
OpenWrt 启动后执行以下命令核对状态：
```sh
cat /tmp/sysinfo/board_name
cat /proc/cmdline
cat /proc/mtd
fw_printenv flag_boot_rootfs flag_last_success flag_boot_success \
	flag_try_sys1_failed flag_try_sys2_failed flag_ota_reboot
```

Expected:
标准输出：
```text
xiaomi,be7000
ubi.mtd=rootfs_1
flag_boot_rootfs=1
flag_last_success=1
flag_boot_success=1
flag_try_sys1_failed=0
flag_try_sys2_failed=0
flag_ota_reboot=0
```

## Roll Back to Stock

If the stock firmware is still in `rootfs`, switch boot flags back to slot `0`:
回退小米原厂固件
原厂固件仍完整存放在 rootfs 分区时，修改启动标记切回 0 号分区：
```sh
fw_setenv flag_boot_rootfs 0
fw_setenv flag_last_success 0
fw_setenv flag_boot_success 1
fw_setenv flag_try_sys1_failed 0
fw_setenv flag_try_sys2_failed 0
fw_setenv flag_ota_reboot 0
reboot
```

## Notes

- The Xiaomi web updater is not suitable for these OpenWrt images.
- `factory.ubi` is for manual slot flashing from stock firmware.
- `sysupgrade.bin` is for OpenWrt sysupgrade only.
- The OpenWrt UBI image uses the stock root volume name `ubi_rootfs`.
- Stock Xiaomi persistent data is stored separately in `mtd28` named
  `overlay`; do not erase it if you want stock rollback to remain useful.

## Warning

Flashing firmware always carries a risk. Without UART, a wrong partition write
or wrong boot flag can make recovery difficult. Verify the active slot and file
hash before writing anything to flash.

补充说明
小米官方网页升级工具不兼容本 OpenWrt 镜像，不可使用；
factory.ubi 仅用于原厂固件下手动刷写备用系统分区；
sysupgrade.bin 仅限 OpenWrt 环境内系统升级；
OpenWrt UBI 镜像沿用原厂默认卷名ubi_rootfs；
小米原厂持久化数据单独存放于 mtd28 分区 overlay；如需保留回退原厂功能，请勿格式化该分区。
风险警告
刷写固件存在变砖风险。若无串口调试工具，写入错误分区、配置错误启动标记将大幅提升救砖难度。向闪存写入数据前，务必确认当前启动槽位并校验文件哈希值。
