# RG35XX-SP 主线内核路线（路线 C）

> **状态**: 2026-05-23 一期实测跑通 95% 功能。P0 全部解决(7/7)、P1 8 项修复 + 8 项 cosmetic、P2 待处理。
>
> 完整问题清单: [`issues.md`](issues.md)
> LCD 调试全过程: [`display-worklog.md`](display-worklog.md)
> 自做 rootfs 固件清单: [`rootfs-prep.md`](rootfs-prep.md)

## 概述

完全主线三件套: TF-A `lts-v2.14.2` + U-Boot `v2026.04` + Linux `7.0.9`，加上 Batocera display patch 和 `rg35xxsp.config`。

**优势**: 干净 upstream 内核；rootfs 可复用原 Ubuntu 22.04；PSCI 标准 cpuidle（无路线 A 的 CPU 0 RCU stall 问题）。

**代价**: 放弃 libmali blob（走 panfrost 开源 GPU 驱动）；USB host 待修；蓝牙已弃用。

## 已实测功能

| 子系统 | 状态 | 路径 |
|---|---|---|
| SPL/BL31/U-Boot/Kernel 全链路 | ✅ | TF-A `lts-v2.14.2` + U-Boot `2026.04` + Linux `7.0.9` + Batocera display patch |
| 串口 console + getty + SSH | ✅ | extlinux cmdline U-Boot bootcmd 直接加载 |
| SSH over USB (CDC ECM gadget) | ✅ | `10.1.1.3`, systemd 持久化, 拔线回落 WiFi |
| 4 核 SMP + cpuidle (PSCI) | ✅ | mainline 标准 |
| cpufreq | ✅ | 8 档 OPP 480-1512 MHz; builtin |
| AXP717 PMIC + battery + USB power | ✅ | SOC/电压/电流 全可读 |
| 电源键 (KEY_POWER) | ✅ | `axp20x-pek` |
| WiFi RTL8821CS (mmc1 SDIO) | ✅ | rtw88_8821cs + rtw8821c_fw.bin |
| Audio (内置 codec) | ✅ | sun4i-codec, asound.conf 卡名修复 |
| RTC PCF8563 + sun6i RTC | ✅ | rtc0/rtc1 |
| Mali GPU (panfrost) | ✅ | Mali-G31, renderD128, 需 IOMMU + PRCM PPU builtin |
| 屏 (NV3052C 640×480) | ✅ | Batocera DE33/Tcon/LCD patch + 自家 SP panel 时序 |
| 游戏按键 (15 键 + 2 音量 + 合盖) | ✅ | GPIO keys, 全注册 |
| binfmt_misc / SO_PASSSEC | ✅ | rg35xxsp.config 启用 |
| TF2 卡槽 (第二 SD 卡) | ✅ | PE4/PE22, 59.5 GB SDXC, 仅 DTS 新增 |
| U-Boot uboot.env 红字 | ✅ | `CONFIG_ENV_IS_NOWHERE=y` |
| 蓝牙 RTL8821CS | ❌ | **显式弃用** (`CONFIG_BT=n`)，会拉死 WiFi，详见 [issues.md P0-2](issues.md) |
| 模拟摇杆 | ❌ | SP 无此硬件 |
| USB-OTG host (键盘等) | ⚠️ | gadget 可用; host 需 OTG 转接头 + PHY0 共存修复 |

## 构建产物结构

```
scripts/
├── build_tfa.sh        # TF-A bl31.bin (~45K)
├── build_uboot.sh      # 派生 SP defconfig + 嵌 bl31 → u-boot-sunxi-with-spl.bin (~737K)
├── build_kernel.sh     # rg35xxsp.config + defconfig + Image.gz + dtbs + modules
├── build_chroot.sh     # 通用 chroot 构建环境 (被 apps/ 下脚本 source)
├── flash_sd.sh         # 拔卡 dd 主路径 (首刷会备份前导 36 MiB + p2 全量)
├── deploy_ssh.sh       # WiFi 通后: scp Image.gz + dtb + modules + firmware 到设备 (免拔卡)
├── deploy_api_headers.sh  # 部署内核 UAPI 头文件
├── download_firmware.sh   # 下载 WiFi/regdb 固件
├── rootfs_mk.sh        # rootfs 构建主脚本 (解包 → update → config → 模块 → 打包)
├── rootfs_update.sh    # rootfs 包管理 (卸载无用包/升级/装依赖/清理旧 BSP 模块)
├── rootfs_config.sh    # rootfs 幂等配置 (hostname/SSH/fstab/network/服务/USB/音量)
└── apps/               # 应用构建: ffmpeg/mpv/retroarch/ppsspp 等

out/
├── firmware/           # WiFi/regdb 固件
├── Image.gz + dtb      # 内核产物
└── arch-rootfs.tar.gz  # rootfs 打包
```

## 编译 → 部署 → 测试 一条龙

```bash
cd scripts/

# 编译三件套 (首次约 15 分钟, 主要在 kernel)
./build_tfa.sh
./build_uboot.sh
./build_kernel.sh

# 部署: 首次拔卡 dd (自动备份前导 36 MiB + p2)
sudo ./flash_sd.sh --check          # 校验产物
sudo ./flash_sd.sh                  # 实际刷写

# 拔卡 → 装回设备 → 串口 picocom -b 115200 /dev/ttyUSB0 → 上电

# WiFi 通了之后免拔卡迭代:
./deploy_ssh.sh --reboot            # ssh 推 modules + firmware, 自动 reboot
sudo ./flash_sd.sh --rollback       # 回到 day-0 原厂
```

## minimal config 增量 (`src/linux-7.0.9/kernel/configs/rg35xxsp.config`)

```kconfig
# AXP717 子驱动
CONFIG_AXP20X_POWER=m
CONFIG_AXP20X_ADC=m
CONFIG_BATTERY_AXP20X=m
CONFIG_INPUT_AXP20X_PEK=m

# Audio
CONFIG_SND_SUN4I_CODEC=m

# WiFi RTL8821CS
CONFIG_RTW88=m CONFIG_RTW88_CORE=m CONFIG_RTW88_SDIO=m
CONFIG_RTW88_8821C=m CONFIG_RTW88_8821CS=m

# Bluetooth (显式弃用, 避免拉死 WiFi)
CONFIG_BT=n

# cpufreq / IOMMU / PRCM PPU (必须 builtin)
CONFIG_ARM_ALLWINNER_SUN50I_CPUFREQ_NVMEM=y
CONFIG_SUN50I_IOMMU=y
CONFIG_SUN50I_H6_PRCM_PPU=y

# Panel / LCD
CONFIG_DRM_PANEL_NEWVISION_NV3052C=m
CONFIG_SPI_GPIO=m
CONFIG_BACKLIGHT_GPIO=m

# binfmt_misc / SO_PASSSEC
CONFIG_BINFMT_MISC=y
CONFIG_SECURITY_NETWORK=y
```

## 必须的 patch

1. **mmc-pwrseq ENOENT 修复** (7 行):
   mainline 7.0.9 `drivers/mmc/core/pwrseq_simple.c` 中 reset control 获取失败返回 `-ENOENT` 而非 NULL, 导致 WiFi mmc1 永远 probe deferred。

2. **Display 子系统补丁** (Batocera 11 子 patch, 见 `display-worklog.md`):
   DE33 mixer/TCON 支持 H616 + dtsi 补全 display 节点 + 板级 panel/backlight/spi_lcd wire。加自家 NV3052C SP panel 时序 patch。

3. **phy-sun4i-usb EPROBE_DEFER 降噪** (commit `6a291bb29`):
   `dev_err` → `dev_err_probe` 避免启动日志刷 "failed to get reset usb0_reset"。

4. **tcon_tv0 阻塞修复** (commit `8dfd74be3`):
   `sun50i-h616.dtsi` 中 tcon_tv0 缺 `status = "disabled"` 导致 DRM component bind 被阻塞。

## 踩过的 10 个坑

1. **mmcblk0p2 (32 MiB FAT16) 装不下 Image (40 MiB)** → 用 `make Image.gz` 出 15 MiB 压缩版, U-Boot `booti` 自带 `image_decomp` 透明 gunzip
2. **U-Boot 默认 distroboot 扫到 p1 (Roms 2 GiB FAT) 卡死** → defconfig 覆盖 `CONFIG_BOOTCOMMAND` 直接 `load mmc 0:2 ${kernel_addr_r} /Image.gz` 跳过 p1
3. **`tar -xzf modules.tar.gz -C /` 把 rootfs 的 `/lib -> usr/lib` 软链替换成真目录** → systemd `/sbin/init -> /lib/systemd/systemd` 变 broken link, kernel panic。**必须用 `tar --keep-directory-symlink`**; 救修脚本 `fix_p5_lib_symlink.sh`
4. **mainline 7.0.9 mmc-pwrseq-simple 有 -ENOENT 回归** → 见上 patch
5. **AXP717 子驱动 defconfig 默认 =n** → battery / usb power / adc / pek 全无 sysfs 节点
6. **CPU 时钟管理 `sun50i-cpufreq-nvmem` =m 时 udev 不自动 modprobe** → 必须改 =y builtin
7. **BT `hci_uart` =m 与 serdev 有 probe race** → 必须 =y builtin (后决定弃用 BT)
8. **BT firmware `rtl8821cs_config.bin` 上游 linux-firmware 没收录** → 从设备抓 vendor blob (后决定弃用 BT)
9. **Anbernic `launcher.service` 每 30s 写 `drop_caches`** → `systemctl mask launcher.service` 清掉
10. **`/lib/modules/mali_kbase.ko` (BSP r20p0 散件) 在 rootfs 里** → `rm` 掉, 释放 30 MB

## 二期新增 (2026-05-23)

- **USB gadget CDC ECM**: `10.1.1.3`, systemd 持久化, 与 RG350 共用 10.1.1.0/24 网段
- **TF2 卡槽**: PE4(供电) + PE22(卡检测), 仅 DTS +28 行, `reg_cldo2` 供电
- **音频 asound.conf**: 卡名 `audiocodec`(BSP) → `Codec`(mainline), 默认 `aplay` 可用
- **部署加速**: `flash_sd.sh` / `deploy_ssh.sh` 均改为 rsync 增量同步
- **U-Boot**: `CONFIG_ENV_IS_NOWHERE=y` 消去启动红字

## 重要注意事项

- **libmali blob 与 `mali_kbase` 模块 ABI 绑定**: 换内核必须保留同版本 kbase 驱动 + 同版本 libmali。换 mainline 即彻底放弃 libmali（走 panfrost）。
- **DRAM 初始化在 SPL 阶段**: 改 kernel 不影响; 若改 U-Boot 需要 H700 DRAM lib。
- **蓝牙已弃用**: RTL8821CS BT 在 mainline 7.0.9 上会拉死 WiFi, `CONFIG_BT=n` 是有意决策, 不要还原。。
- **首次测试务必走"备份 SD → 单 SD 写 p4 → 插回试"流程**: 失败用读卡器写回 `boot.img` 即可恢复。

## 路线 B2 参考 (mainline 6.18.16, 已被 C 路线取代)

路线 B2 使用 `src/linux-6.18.16/` + Batocera 28 patch (27 原始 + 1 combined), 保留原厂 SPL+ATF+U-Boot, 仅替换 mmcblk0p4。C 路线在此基础上升级到 7.0.9 并替换了整个 boot chain (TF-A + U-Boot), 功能更完整。B2 路线的 5 个坑 (mkbootimg 2MB 对齐 / earlycon / loglevel / CMDLINE_FORCE / ramdisk 兼容) 在 C 路线中同样适用, 已全部解决。

相关脚本: `build/b/b_build.sh`, `build/b/b_deploy.sh`, `build/b/b_sd_flash.sh`
