# rootfs 固件与数据准备

当前主线内核为 Linux 7.3.1。rootfs 需要提供匹配的模块、WiFi 固件和基础
用户态配置。部署前先确认实际根分区。

## 必要文件

| 文件 | 用途 |
|---|---|
| `/lib/firmware/rtw88/rtw8821c_fw.bin` | RTL8821CS SDIO WiFi，来自 [linux-firmware](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/rtw88) |
| `/lib/firmware/regulatory.db` 及签名 | 无线监管数据库，使用发行版提供的匹配文件 |
| `/lib/modules/7.3.1/` | 由 BSP 的 `modules.tar.gz` 提供，必须与启动内核匹配 |
| ALSA 配置 | 通过 `alsamixer` 调整后用 `alsactl store` 保存 |

当前配置禁用蓝牙，不需要蓝牙固件或 bluez。Panfrost 使用开源驱动，不沿用
原厂 `libmali` / `mali_kbase`。Cedrus、音频 codec、USB PHY 和 NV3052C 面板
由内核驱动及设备树提供。

## 基础系统

- 安装 udev、WiFi 管理工具和 `alsa-utils`。
- PCF8563 在设备树中固定为 rtc0 并内建；用户态使用 UTC 和正常的时间同步。
- 不要把 `/dev/rtc` 固定到 sun6i RTC。
- 网络接口按驱动匹配，地址和路由按实际环境设置。
- 移除原厂内核专用的模块加载项和不适用于主线的服务。
- 解包模块时保留 rootfs 的 `/lib` 目录软链，例如使用 tar 的
  `--keep-directory-symlink`。

## 设备数据

eFuse、SID、MAC 和校准数据属于设备本身，应自行备份。不要用其他设备的
`rtl8821cs_config.bin` 或校准数据替代。

面板时序、电池参数和背光由设备树及内核驱动提供；只替换 rootfs 不会改变这些
硬件配置。

构建和分区参数见[主线内核指南](mainline-guide.md)，硬件背景见
[硬件事实](device-facts.md)。
