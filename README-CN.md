小米 BE7000 OpenWrt 安装说明
适用于小米 BE7000 的 OpenWrt 安装文件与刷机说明
本仓库用于安装 OpenWrt，同时保留小米原厂固件分区：
```text
rootfs    -> 小米原厂固件，请勿修改
rootfs_1  -> OpenWrt 系统分区，后续系统升级也将写入此分区
```
首次刷机完成后，执行 OpenWrt 的 sysupgrade 升级命令会持续使用 rootfs_1 分区。
原厂固件分区 rootfs 保留完好，支持随时回退原厂系统。
文件说明
发布包包含以下文件：
```text
openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-factory.ubi
openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-sysupgrade.bin
sha256sums
```
镜像文件使用区分：
```text
factory.ubi      -> 在原厂固件终端手动刷写分区使用
sysupgrade.bin   -> 在 OpenWrt / 临时内存系统 / LuCI 后台内执行系统升级使用
```
禁止在原厂固件环境下直接将 sysupgrade.bin 写入 rootfs 或 rootfs_1 分区。小米引导加载程序仅识别标准 UBI 镜像作为系统分区镜像。
前置准备
你需要具备以下任意一种操作环境：
小米原厂固件获取 root 终端（SSH 或 Telnet）
路由器已启动 OpenWrt 临时内存系统（initramfs）
UART 串口调试工具（设备变砖救砖必备）
若无串口工具，向闪存写入数据前，请仔细核对每一条命令。
已验证闪存分区布局
```text
mtd17: 00080000 00020000 "0:APPSBLENV"
mtd23: 02800000 00020000 "rootfs"
mtd24: 02800000 00020000 "rootfs_1"
mtd28: 01c00000 00020000 "overlay"
```
小米系统启动标记存储在分区 0:appsblenv。
OpenWrt 读写启动环境变量工具 fw_printenv/fw_setenv 配置：
```text
/dev/mtd17 0x0 0x10000 0x20000
```
安全安装分区逻辑
本保留原厂固件的刷机流程，要求路由器当前从 0 号原厂分区 启动：
```text
flag_boot_rootfs=0
ubi.mtd=rootfs
```
OpenWrt 仅写入 rootfs_1 分区：
```text
目标分区：mtd24 / rootfs_1
目标系统槽位：1
```
如果当前原厂固件运行在 rootfs_1 分区（参数 flag_boot_rootfs=1），请立刻停止操作，不要使用本套保留原厂固件的刷机流程。
查看当前运行系统槽位
在原厂固件终端执行：
```sh
nvram get flag_boot_rootfs
cat /proc/cmdline
cat /proc/mtd
```
符合刷机条件的输出：
```text
flag_boot_rootfs=0
```
核对内核启动参数确认分区：
```sh
sed -n 's/.*ubi.mtd=\([^ ]*\).*/\1/p' /proc/cmdline
```
预期输出：
```text
rootfs
```
若启动标记与内核挂载分区参数不一致，停止刷机并排查问题。
在原厂固件终端安装 OpenWrt
将 UBI 镜像上传至路由器临时目录：
```sh
scp -O openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-factory.ubi \
	root@192.168.31.1:/tmp/openwrt.ubi
```
再次确认当前启动槽位：
```sh
nvram get flag_boot_rootfs
cat /proc/cmdline
cat /proc/mtd
```
正常输出：
```text
flag_boot_rootfs=0
将 OpenWrt 刷入 rootfs_1 分区：
sh
ubiformat /dev/mtd24 -y -f /tmp/openwrt.ubi
```
修改小米启动标记，切换至 1 号分区启动：
```sh
nvram set flag_boot_rootfs=1
nvram set flag_last_success=1
nvram set flag_boot_success=1
nvram set flag_try_sys1_failed=0
nvram set flag_try_sys2_failed=0
nvram set flag_ota_reboot=0
nvram commit
```
重启路由器：
```sh
reboot
```
在 OpenWrt 内安装 / 升级系统
将升级镜像上传至路由器：
```sh
scp openwrt-qualcommbe-ipq95xx-xiaomi_be7000-squashfs-sysupgrade.bin \
	root@192.168.1.1:/tmp/openwrt.bin
```
执行系统升级：
```sh
sysupgrade -n /tmp/openwrt.bin
```
适配 BE7000 的 OpenWrt 升级脚本会自动完成：将镜像写入 rootfs_1、去除 OpenWrt 头部元数据后格式化 UBI、保留配置、同步小米启动标记为 1 号槽位。
刷机启动后校验
OpenWrt 启动完成后执行以下命令核对状态：
```sh
cat /tmp/sysinfo/board_name
cat /proc/cmdline
cat /proc/mtd
fw_printenv flag_boot_rootfs flag_last_success flag_boot_success \
	flag_try_sys1_failed flag_try_sys2_failed flag_ota_reboot
```
预期输出：
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
回退至原厂固件
若原厂固件仍保存在 rootfs 分区，修改启动标记切回 0 号分区：
```sh
fw_setenv flag_boot_rootfs 0
fw_setenv flag_last_success 0
fw_setenv flag_boot_success 1
fw_setenv flag_try_sys1_failed 0
fw_setenv flag_try_sys2_failed 0
fw_setenv flag_ota_reboot 0
reboot
```
补充说明
小米官方网页升级工具不兼容本 OpenWrt 镜像，请勿使用；
factory.ubi 仅用于原厂固件环境下手动刷写备用系统分区；
sysupgrade.bin 仅可在 OpenWrt 系统内执行升级；
OpenWrt UBI 镜像沿用原厂默认卷名 ubi_rootfs；
小米原厂持久化数据单独存放在 mtd28 分区，分区名称 overlay；若需要保留回退原厂固件功能，请勿擦除此分区。
风险警告
刷写固件始终存在变砖风险。若无 UART 串口工具，写入错误分区、配置错误启动标记会大幅增加救砖难度。向闪存写入任何数据前，请务必确认当前运行系统槽位并校验文件哈希值。
