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

## Files

Expected release files:

```text
openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-factory.ubi
openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-sysupgrade.bin
sha256sums
```

Use the images as follows:

```text
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

```text
mtd17: 00080000 00020000 "0:APPSBLENV"
mtd23: 02800000 00020000 "rootfs"
mtd24: 02800000 00020000 "rootfs_1"
mtd28: 01c00000 00020000 "overlay"
```

Xiaomi boot flags are stored in `0:appsblenv`.

OpenWrt `fw_printenv/fw_setenv` config:

```text
/dev/mtd17 0x0 0x10000 0x20000
```

## Safe Install Layout

The stock-preserving installation expects stock firmware to be booted from
slot `0`:

```text
flag_boot_rootfs=0
ubi.mtd=rootfs
```

OpenWrt is then written only to `rootfs_1`:

```text
target partition: mtd24 / rootfs_1
target slot:      1
```

If stock is currently booted from `rootfs_1` (`flag_boot_rootfs=1`), stop and
do not use this stock-preserving procedure.

## Check the Active Slot

On stock firmware:

```sh
nvram get flag_boot_rootfs
cat /proc/cmdline
cat /proc/mtd
```

Expected for this procedure:

```text
flag_boot_rootfs=0
```

Cross-check the kernel command line:

```sh
sed -n 's/.*ubi.mtd=\([^ ]*\).*/\1/p' /proc/cmdline
```

Expected:

```text
rootfs
```

If the boot flag and `ubi.mtd=` disagree, stop and investigate before flashing.

## Install from Stock Firmware Shell

Copy the UBI image to the router:

```sh
scp -O openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-factory.ubi \
	root@192.168.31.1:/tmp/openwrt.ubi
```

Verify the current slot:

```sh
nvram get flag_boot_rootfs
cat /proc/cmdline
cat /proc/mtd
```

Expected:

```text
flag_boot_rootfs=0
```

Flash OpenWrt to `rootfs_1`:

```sh
ubiformat /dev/mtd24 -y -f /tmp/openwrt.ubi
```

Switch Xiaomi boot flags to slot `1`:

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

```sh
reboot
```

## Install or Upgrade from OpenWrt

Copy the sysupgrade image to the router:

```sh
scp openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-sysupgrade.bin \
	root@192.168.1.1:/tmp/openwrt.bin
```

Run sysupgrade:

```sh
sysupgrade -n /tmp/openwrt.bin
```

The BE7000 OpenWrt upgrade code writes the image to `rootfs_1`, strips
OpenWrt metadata before `ubiformat`, restores configuration, and sets Xiaomi
boot flags to slot `1`.

## Check After Boot

After OpenWrt boots:

```sh
cat /tmp/sysinfo/board_name
cat /proc/cmdline
cat /proc/mtd
fw_printenv flag_boot_rootfs flag_last_success flag_boot_success \
	flag_try_sys1_failed flag_try_sys2_failed flag_ota_reboot
```

Expected:

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
