# RG35XX-SP BSP

Anbernic RG35XX-SP（Allwinner H700）的主线内核适配、硬件事实与开发记录。

## 当前进展

硬件验证记录更新至 **2026-10-11**；公开 BSP 构建仓库于 **2026-10-16** 同步到 Linux **7.2.9**。

- 7.2.9 已包含 TCON TOP 路由修复，LCD 左右边缘回绕问题已通过重启及视觉验收。
- RTC 使用内建 PCF8563 作为 rtc0；GPIO、MFD DMA mask、TF2 写保护声明及 nftables 配置已修复。
- 正常启动使用 `drm.debug=0`，保留标准启动消息和真实告警。
- 视频兼容性、USB Host、DE33 缩放及部分人工验收仍有待办，见[问题清单](issues.md)。

## 开始使用

| 目标 | 入口 |
|---|---|
| 构建主线内核与启动固件 | [主线内核指南](mainline-guide.md) |
| 获取构建脚本、补丁和配置 | [公开 BSP 仓库](https://github.com/hsyysy/bsp-rg35xxsp) |
| 保留原厂内核，更换用户态 | [原厂镜像路线](stock-image-guide.md) |
| 准备 rootfs 固件和模块 | [rootfs 准备清单](rootfs-prep.md) |

本仓库维护文档，编译脚本和设备补丁由 BSP 仓库维护。

## 支持范围

下表包含项目适配与实测结果，不等同于 vanilla 内核的原生能力清单。

| 子系统 | 状态 |
|---|---|
| LCD / DE33 | 点亮及边缘路由修复已验收；部分缩放能力仍不支持 |
| GPU | Panfrost 已有运行验证，依赖 IOMMU 与 PPU |
| WiFi | RTL8821CS SDIO 已验证，rootfs 需提供配套固件 |
| RTC / PMIC / TF2 | 已有校时、电源读数及挂载验证 |
| 音频与音量键 | 驱动和配置已实现，完整听音及实体按键验收记录待补 |
| 视频硬解 | Cedrus H.264 / HEVC 基础链路已通，部分视频兼容性和性能待验证 |
| USB | Gadget 已验证，Host 仍待独立验收 |
| 蓝牙 | 禁用，组合芯片初始化存在影响 WiFi 的问题 |

## 进一步阅读

- [硬件事实](device-facts.md)：设备实测、原厂固件与官方文档的交叉核对。
- [主线支持细节](mainline-support.md)：上游静态分析与项目适配的区别。
- [LCD 调试记录](display-worklog.md)：点屏、路由冲突根因与验证范围。
- [启动告警清理](boot-warning-cleanup.md)：已修复项与保留限制。
- [启动日志配置](kernel-debug-branches.md)：正常日志与诊断参数。
- [H.264 V4L2 Request 研究](v4l2-request-h264.md)：已验证样本与已知反例。

历史记录保留当时的内核版本和实验结论；当前状态以本页、主线指南和问题清单为准。
