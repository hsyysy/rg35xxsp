# 原厂固件分析

> **来源等级声明**：本文档记录对 Anbernic 官方固件镜像的拆解分析。分区结构、内核版本、U-Boot 环境变量等均来自镜像二进制提取，属"设备实测/镜像分析"层级。

## 固件版本历史

| 版本 | 内核 build | 编译时间 | 镜像日期 | 来源 |
|------|-----------|---------|---------|------|
| (早期) | `#11` | 2025-12-24 | — | 设备实测 `uname -a` |
| **V1.1.5** | **`#13`** | **2026-05-22** | 2026-05-27 | `RG35XXSP-V1.1.5-CN16GB-260522.IMG` |

- **内核版本**：始终为 `4.9.170 SMP PREEMPT`
- **编译主机**：`flower@flower-B85M-D2V`（两个版本一致）
- **编译器**：GCC 5.3.1 (Linaro GCC 5.3-2016.05)
- **SDK 路径**（V1.1.5 kernel 暴露）：`/home/flower/cc_share/allwinner/h700/rg35xxsp/sdk/dynamic_standby/kernel/linux-4.9`

---

## V1.1.5 镜像结构

- **文件**：`RG35XXSP-V1.1.5-CN16GB-260522.IMG`
- **大小**：15,476,981,760 bytes (14.41 GiB)
- **分区表**：GPT

| 分区 | 起始扇区 | 大小 | 文件系统 | 说明 |
|------|----------|------|----------|------|
| p1 | 73728 | 2 GiB | FAT32 | Roms/BIOS/字体/语言包（非用户 ROM） |
| p2 | 4268032 | 32 MiB | FAT16 | bootlogo.bmp + 字体 + 充电资源 |
| p3 | 4333568 | 16 MiB | raw | U-Boot 环境变量 |
| p4 | 4366336 | 64 MiB | Android bootimg | kernel + ramdisk + DTB |
| p5 | 4497408 | 7 GiB | ext4 | Ubuntu 22.04 rootfs |
| p6 | 19177472 | 4 GiB | ext4 | Anbernic 应用数据 |
| p7 | 27566080 | 1.3 GiB | ext4 | 用户数据 (dmenu/save) |

前 36 MiB 为 Allwinner sunxi BSP 标准布局：BOOT0 在 8KB 偏移 + U-Boot/ATF/DTB 紧随其后。

---

## 启动链

```
BOOT0 (8KB offset) → ATF BL3-1 → U-Boot 2018.05 → boot.img (p4) → kernel + ramdisk → switch_root → p5 ext4
```

### BOOT0 / ATF（来自串口日志）

| 阶段 | 版本 | 备注 |
|------|------|------|
| BOOT0 | commit `ac58e911-dirty` | DRAM BOOT DRIVE INFO V0.651 |
| ATF (BL3-1) | `v1.0(debug):5a77824` | 2024-08-02 15:20:22, **debug build** |
| U-Boot | `2018.05 (Dec 25 2025 - 12:00:24)` | Allwinner 定制版 |

- **CPU/总线时钟**（U-Boot 阶段）：CPU 1008 MHz, PLL6 600 MHz, AHB 200 MHz, APB1 100 MHz, MBus 400 MHz
- **LPDDR4 训练**：两轮 read_calibration 各失败 10 次后 fallback retraining，最终 32-bit 1 rank 成功

### Boot Image (p4) 拆解

| 文件 | 类型 | 大小 |
|------|------|------|
| kernel | ARM64 Linux Image | 17 MB |
| ramdisk | gzip initramfs (BusyBox) | 2.5 MB |
| DTB | Device Tree Blob | 135 KB |

**Header v2**：
- Board name: `sun50i_arm64`
- Kernel load: `0x40080000`
- Ramdisk load: `0x42000000`
- DTB load: `0x44000000`

### Ramdisk init 流程

1. 挂载 proc/sys/dev
2. 从 `/proc/cmdline` 读取 `root=` 设备
3. 支持 NAND / eMMC / NOR Flash 三种启动方式
4. GPT 模式下默认根分区 = `mmcblk0p5`
5. `switch_root` 切换到真正的根文件系统

---

## U-Boot 环境变量 (p3)

共 25 个变量：

| 变量 | 值 | 含义 |
|------|-----|------|
| `mmc_root` | `/dev/mmcblk0p5` | 根文件系统 |
| `console` | `ttyS0,115200` | 串口调试 |
| `bootcmd` | `run setargs_nand boot_normal` | 默认启动流程 |
| `boot_normal` | `sunxi_flash read 45000000 boot;bootm 45000000` | 从 flash 读 boot.img |
| `selinux` | `0` | SELinux 关闭 |
| `bootdelay` | `0` | 无延迟直接启动 |
| `cma` | `64M` | CMA 连续内存 |
| `keybox_list` | `hdcpkey,widevine` | DRM 密钥（Widevine + HDCP） |
| `mac` / `wifi_mac` / `bt_mac` | (空) | MAC 由 eFuse/驱动运行时读取 |

---

## DTB 关键节点

V1.1.5 的 DTB（135 KB）包含以下关键硬件描述：

### PMIC (AXP2202)

- compatible: `x-powers,axp2202` + `axp2202-bat-power-supply` + `axp2202-usb-power-supply`
- 22 路 regulator：dcdc1-4, aldo1-4, bldo1-4, cldo1-4, cpusldo, rtcldo, drivevbus, vmid
- PEK: `x-powers,axp2101-pek`

### 显示

- lcd0: `allwinner,sunxi-lcd0`，驱动名 `fog_fj035fhd05_v1`
- lcd1: `allwinner,sunxi-lcd1`（未使用）
- SPI0/SPI1 总线可用
- PWM0-5（lcd_pwm_used 控制背光）
- HDMI 节点存在（`hdmi@06000000`），SP 无物理 HDMI 口

### 音频

- codec: `0x05096000`（sunxi-snd-codec）
- AHub: ahub0-3 daudio0-3
- HDMI audio codec（SP 无 HDMI 口）

### WiFi / BT

- WiFi: sdmmc2 (`allwinner,sunxi-wlan`)，`wlan_power` / `wlan_io_regulator`
- BT: `allwinner,sunxi-bt` + `allwinner,sunxi-btlpm`
- `bt_power` / `bt_rst_n` / `bt_wake` / `bt_hostwake`

### 其他

- RTC: `nxp,pcf8563`（纽扣电池供电）+ sun6i 片上 RTC
- Hall: `hall_key`（合盖唤醒源）
- `wakeup-source` 支持

---

## p5 rootfs (Ubuntu 22.04)

### 内核模块

```
/lib/modules/4.9.170/
  └── mali_kbase.ko    # Mali GPU 驱动
```

### 固件文件

| 路径 | 文件 | 用途 |
|------|------|------|
| `rtl_bt/` | `rtl8723b_config.bin`, `rtl8723b_fw.bin`, `rtlbt_config`, `rtlbt_fw`, `rtlbt_fw_new` | BT 固件 (RTL8723B 格式) |
| `rtlbt/` | `rtl8821c_config`, `rtl8821c_fw` | BT 固件 (RTL8821C 格式) |
| `rtlwifi/` | rtl8188e/8192c/8723a/8812a/8821a/92cu/8723au 系列 | WiFi 固件 (BSP 4.9 驱动用) |
| `regulatory.db` | — | 无线监管域数据库 |

> **注意**：`rtl_bt/` 下的 rtl8723b 文件与 `rtlbt/` 下的 rtl8821c 文件是不同 Realtek 芯片的固件。设备实际使用 RTL8821CS（`rtl8821c_config` + `rtl8821c_fw`）。

### Anbernic 系统服务

```
rc-local.service          # 自定义启动脚本
sshd.service              # SSH 服务
dbus-fi.w1.wpa_supplicant1.service
dbus-org.freedesktop.nm-dispatcher.service
dbus-org.freedesktop.resolve1.service
```

### 动态待机

SDK 路径含 `dynamic_standby`，表明原厂 BSP 有深度休眠/唤醒的定制电源管理实现。

---

## p6 应用数据

| 目录 | 内容 |
|------|------|
| `bin/` | `dmenu.bin` (主菜单), `ebook`, `evtest`, `fbtest3`, `fileM`, `cexpert`, `charg.dge` |
| `lib/` | SDL 1.2 系列 (libSDL, libSDL_gfx, libSDL_image, libSDL_mixer, libSDL_ttf) |
| `ctrl/` | 手柄映射配置 |
| `deep/` | 深度相关数据 |
| `oem/` | OEM 定制 |

---

## p2 boot 资源

| 文件 | 类型 | 说明 |
|------|------|------|
| `bootlogo.bmp` | 640×480 24-bit BMP | 开机 logo |
| `fastbootlogo.bmp` | 246×257 24-bit BMP | fastboot 模式 logo |
| `font24.sft` / `font32.sft` | 字体 | U-Boot 控制台字体 |
| `magic.bin` | 512 bytes | 魔数/标识 |
| `bat/` | — | 充电相关资源 |

---

## 与路线 C mainline 的关系

- p4 保留原厂 boot.img 作回滚锚点
- 路线 C 已将 p2 改作 mainline boot 分区（Image.gz/dtb/extlinux.conf），原厂 logo 可从 `device-backup/p2-orig.tar` 恢复
- p5 rootfs 已不可逆改动（`/lib/modules/4.9.170/` 已被清理）
- 前 36 MiB SPL 区域被 mainline SPL/BL31 替换，与原厂布局兼容（8KB 偏移）
