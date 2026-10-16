# rootfs 固件与数据准备

当前主线内核为 Linux 7.2.9。rootfs 需提供对应模块、WiFi 固件和基础用户态配置。
原厂多分区镜像与当前双分区布局不同，部署前应确认实际根分区。

## 当前主线需要的文件

| 文件 | 用途与来源 |
|---|---|
| `/lib/firmware/rtw88/rtw8821c_fw.bin` | RTL8821CS SDIO WiFi；来自 [linux-firmware](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/rtw88) |
| `/lib/firmware/regulatory.db` 及配套签名 | 无线监管数据库；优先使用发行版提供的匹配文件 |
| `/lib/modules/7.2.9/` | 由公开 BSP 的 `modules.tar.gz` 提供，须与启动内核匹配 |

当前配置禁用蓝牙，不将蓝牙固件或 bluez 列为必要依赖。
历史实验曾提取原厂 `rtl8821cs_config.bin`，其中可能含设备的 MAC / RF 参数，
不应以其他设备的同名配置替代。

Panfrost 使用开源驱动，不沿用原厂 libmali / mali_kbase 组合。
Cedrus、内置音频 codec、USB PHY 与当前 NV3052C 面板适配由内核驱动及设备树提供。

## 设备特定数据

| 数据 | 处理方式 |
|---|---|
| eFuse / SID、MAC 与校准数据 | 使用本设备的数据，备份自行保管 |
| ALSA 默认音量与路由 | 在目标系统通过 alsamixer 配置，再用 alsactl store 保存 |
| 背光状态 | 由用户态或 systemd-backlight 管理 |
| 前端及模拟器配置 | 随所选用户态单独准备 |
| 电池参数与面板时序 | 跟随已适配的设备树，不能只替换 rootfs 中的文件 |

## 基础系统配置

- 按发行版准备 udev、WiFi 管理工具与 alsa-utils。
- PCF8563 在设备树中设为 rtc0，并内建于内核；用户态使用 UTC 配置及正确的时间同步服务。
- 不沿用会把 `/dev/rtc` 指向 sun6i RTC 的历史固定规则。
- 配置网络时按实际接口和驱动匹配，地址与路由按使用环境设置。
- 移除原厂内核专用的模块加载项及不适用于当前主线的服务。
- 解包模块时保留 rootfs 的 `/lib` 目录软链，避免覆盖发行版布局。

构建及分区参数见[主线指南](mainline-guide.md)，硬件背景见[设备事实](device-facts.md)，
待办与历史修复见[问题清单](issues.md)。
