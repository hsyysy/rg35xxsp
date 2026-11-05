# RG35XX-SP BSP

Anbernic RG35XX-SP（Allwinner H700）的主线 Linux BSP、硬件说明与开发文档。

当前基线：Linux **7.3.1**、U-Boot **2026.04**、TF-A **lts-v2.14.2**。
构建脚本、内核补丁和配置见 [bsp-rg35xxsp](https://github.com/hsyysy/bsp-rg35xxsp)。

## 当前支持

| 子系统 | 状态 |
|---|---|
| 启动链 | TF-A / U-Boot / Linux 7.3.1，可从 SD 卡启动 |
| LCD | 640×480 NV3052C 面板，DE33 显示引擎，PWM 背光 |
| HDMI | mini-HDMI 1080p60 输出，H616 AHUB HDMI 音频 |
| 音频 | 扬声器、耳机、音量键；插耳机时自动关闭扬声器 PA |
| 网络 | RTL8821CS SDIO WiFi；USB CDC-ECM gadget 提供 SSH |
| USB | 默认 gadget；运行时切换 Host，支持 U 盘、键盘、鼠标 |
| 电源 | AXP717 PMIC、电池、电源键、合盖熄屏 |
| 存储 | TF1 启动/根分区，TF2 exFAT 用户卡 |
| 视频 | Cedrus 硬件解码：H.264、HEVC Main、MPEG-2、VP8 |
| 蓝牙 | 禁用；RTL8821CS 组合芯片的 BT 初始化会影响 WiFi |

更细的验收范围和未完成项见[支持与限制](status.md)。

## 开始使用

| 目标 | 入口 |
|---|---|
| 构建主线内核与启动固件 | [主线内核指南](mainline-guide.md) |
| 获取构建脚本、补丁和配置 | [公开 BSP 仓库](https://github.com/hsyysy/bsp-rg35xxsp) |
| 保留原厂启动链，更换用户态 | [原厂镜像路线](stock-image-guide.md) |
| 准备 rootfs 固件和模块 | [rootfs 准备](rootfs-prep.md) |

## 进一步阅读

- [硬件事实](device-facts.md)：SoC、显示、音频、电源、存储和外设。
- [上游支持与 BSP](mainline-support.md)：主线已有能力与本仓库补丁的范围。
- [支持与限制](status.md)：已验证功能、已知限制和待办。
- [显示与 DE33](display.md)：LCD、HDMI、YUV 格式和硬件缩放。
- [视频硬解](video.md)：Cedrus / V4L2 Request 的支持范围。
- [启动告警](boot-warning-cleanup.md)与[启动日志](kernel-debug-branches.md)。

本仓库只维护 BSP 和硬件适配文档，不包含 ROM、BIOS 或游戏资源。
