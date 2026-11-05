# 上游支持与 BSP

## 启动链

| 组件 | 版本 | 说明 |
|---|---|---|
| TF-A | lts-v2.14.2 | `PLAT=sun50i_h616`，生成 BL31 |
| U-Boot | 2026.04 | 从 `anbernic_rg35xx_h700_defconfig` 派生 SP defconfig |
| Linux | 7.3.1 | vanilla 基线 + 本仓库设备补丁 |

U-Boot 使用直接 bootcmd 加载 `Image.gz` 和 DTB，不依赖 distroboot。
SP defconfig 只改默认设备树，并追加启动参数和 bootcmd。

## 内核补丁范围

`patches/linux/0001-rg35xxsp-linux-7.3.1.patch` 相对 vanilla 7.3.1 包含：

| 区域 | 内容 |
|---|---|
| 板级 DTS | RG35XX-SP 的 LCD、HDMI、音频、USB、TF2、按键、RTC、VE |
| H616 DTSI | display engine、DE33 总线、HDMI、AHUB、VE |
| DRM / sun4i | DE33 planes、per-channel CSC、VSU10/VSU8/ED scaler |
| DRM / TCON | H616 TCON TV、LCD HBP、TCON TOP 路由 |
| DRM / HDMI | H616 PHY、TMDS 模式限制、板级时钟上限 |
| 音频 | H616 AHUB / DAM / soundcard-mach、耳机 PA 门控 |
| PWM | `pwm-sun20i`、PWM 背光 |
| USB | PHY0 host/peripheral 角色切换、VBUS 控制 |
| Cedrus | H616 VPU、NV12 64 字节 stride 对齐 |
| 其他 | 可选 reset/regulator 处理、启动告警清理、VT 光标隐藏 |

## 与厂商 BSP 的区别

- 使用主线 DRM/KMS、Panfrost、rtw88、cedrus 和 AXP20X 驱动栈。
- 不包含厂商 `disp2`、`mali_kbase`、私有 AXP2202 regulator 和厂商音频驱动。
- 显示走 DE33 planes / mixer / TCON，而不是厂商 disp2。
- 蓝牙不编译，避免 RTL8821CS 组合芯片的 BT 初始化影响 WiFi。
- 没有模拟摇杆和 OPTEE 支持。

## 上游状态

DE33 显示、H616 TCON TV 和 HDMI PHY 的大部分补丁来自 Jernej Skrabec 的
上游系列；RG35XX 系列设备树来自 Ryan Walklin、Chris Morgan 等人的工作；
H616 AHUB 音频和 `pwm-sun20i` 来自 Batocera H700 补丁集及对应作者。
这些改动在 7.3.1 上仍需由 BSP 携带，不能假设 vanilla 内核已经具备完整支持。

## 不在 BSP 范围

BSP 只构建 TF-A、U-Boot、内核、DTB 和模块。rootfs、固件、菜单、模拟器和
游戏资源由使用方自行准备；相关入口见[主线内核指南](mainline-guide.md)和
[rootfs 准备](rootfs-prep.md)。
