# RG35XX-SP BSP（Board Support Package）

Anbernic RG35XX-SP (Allwinner H700) 的内核 / 固件 / rootfs 开发笔记与构建产物。

## 路线导航

| 路线 | 文档 | 状态 | 一句话 |
|---|---|---|---|
| **A. 原厂镜像** | [`stock-image-guide.md`](stock-image-guide.md) | ✅ 可用 | 保留原厂 kernel, 仅换 userspace 或追加 .ko |
| **B. BSP 4.9 重编** | [`stock-image-guide.md`](stock-image-guide.md) | ❌ 已死亡 | CPU 0 RCU stall, 原厂源码公网不存在 |
| **C. 主线内核** | [`mainline-guide.md`](mainline-guide.md) | ✅ **推荐** | TF-A + U-Boot + Linux 7.0.9, 屏/GPU/WiFi/音频/电池 全通 |

**快速开始**: 如果只是想换个 rootfs (Ubuntu/Alpine), 走路线 A; 如果需要现代化内核 + 主线 upstream, 走路线 C。

**原厂固件下载**: <https://win.anbernic.com/download/412.html>

## 参考资料

### Batocera 已有完整 mainline 适配

Batocera 主线 ([batocera-linux/batocera.linux#master](https://github.com/batocera-linux/batocera.linux)) 已维护 RG35XX-SP 生产级 mainline 适配, 从 2024-10 起持续更新。完整 recipe 在 `thirdparty/batocera-h700/`:

| 组件 | 版本 |
|---|---|
| Kernel | Linux 6.18.16 + 37 H700 patch + 59 H616 patch |
| U-Boot | 2025.10 mainline, defconfig `anbernic_rg35xx_h700` |
| ATF | [ARM-software/arm-trusted-firmware](https://github.com/ARM-software/arm-trusted-firmware) tag `lts-v2.12.8` |
| GPU 栈 | panfrost + Mesa3D (OpenGL ES 3.1, Vulkan 实验) |
| Boot 链 | SPL (sector 8192) → ATF BL31 → U-Boot → extlinux.conf → Image + dtb + initrd |

**Panel 驱动**: Batocera 用 Kikuchan 的 `panel-mipi-dpi-spi` 通用驱动 (patch 0009), 由 `.panel` 固件描述文件定义初始化序列。换屏只改 `.panel` 文件, 不动驱动。

**Patch 上下文问题**: 原始 29 个 patch 基于 6.17.7, 内核升到 6.18.16 后有 10 个 apply 失败。本仓库用 `0099-combined-fix-for-6.18-context.patch` 替代, 最终 28 OK / 0 FAIL。

### Mainline 上游支持现状 ([torvalds/linux](https://github.com/torvalds/linux) 7.1-rc3)

| 子系统 | 状态 | 说明 |
|---|---|---|
| RG35XX-SP DTS | ✅ 已合 | `sun50i-h700-anbernic-rg35xx-sp.dts` |
| GPU (panfrost) | ✅ | `arm,mali-bifrost`, `reg_dcdc2` 供电 |
| 音频 codec | ✅ | 扬声器/耳机走 74HC4052D 模拟 mux |
| AXP717/AXP2202 PMIC | ✅ | 完整 regulator 树 + battery + USB power |
| WiFi RTL8821CS (mmc1) | ✅ | pwrseq + 中断 PG15 |
| BT RTL8821CS (uart1) | ✅ | DTS 已 wire (但 mainline 驱动不完整, 见下) |
| 状态 LED / 按键 / lid switch / RTC PCF8563 | ✅ | |
| USB / SD / UART0 / PMIC | ✅ | |
| Panel 驱动 | 🟡 | binding 已合, 但 DTS 没 wire spi0/tcon0/de/backlight, **屏黑** |
| Display Engine | 🟡 | driver 在, DTS 未启用 |
| HDMI | 🟡 | driver 在, DTS 未启用 (SP 无 HDMI 口) |
| 视频解码 cedrus | 🟡 | mainline 支持有限, 无 4K@60 / VP9 |
| 蓝牙 | ⚠️ | DTS 已 wire, 但 hci_h5 不支持 8821CS, 会拉死 WiFi |

**与 BSP 的硬缺口**: 屏黑 (需补 ~40 行 dts override); 无 BSP DE 高级特性 (keystone/SmartColor/AFBC); 视频解码降级; libmali blob 不可用 (走 panfrost)。

### H700 与 H616 的关系

H616 datasheet/manual 的 pinmux + 寄存器层级**完全适用**于 H700。H700 = H616 同 die, 差异仅在封装 ball 数与 RGB LCD/NMI pin 引出。详见 [device-facts.md §10.3](device-facts.md) 与 `§十二`。

### PDF 文档清单

来源: [linux-sunxi.org/H616](https://linux-sunxi.org/H616)

| 文件 | 页 | 类型 | 关键价值 |
|---|---|---|---|
| `2021070513595227.pdf` | 4 | H700 Brief 营销稿 | 仅产品特性 + 框图, 无寄存器 |
| `H616_User_Manual_V1.0_cleaned.pdf` | **831** | **H616 User Manual** 寄存器级 | **本仓库最重要文档**。Ch 3 Memory Map / Ch 5 DRAMC / Ch 7 TCON/TVE/HDMI / Ch 9.6 Port Controller 等 |
| `H616_Datasheet_V1.0_cleaned.pdf` | 67 | **H616 Datasheet** 引脚 + 电气 | Ch 4 完整 pinmux 表 + Ch 5 AC 电气时序 + Ch 7.1 Pin Map |
| `H616_USB_module_manual.pdf` | 19 | BSP USB 配置 | menuconfig + board.dts USB 节点示例 |
| `H616_Android_Q_OTA_Development_and_Use_Guide.pdf` | 16 | Android OTA 指南 | 几乎无 mainline 价值 |

### 其他探查项目

- **[HandsomeMod/linux-allwinner-4.9](https://github.com/HandsomeMod/linux-allwinner-4.9)**: 路线 B 关键拼图, Allwinner BSP 4.9.118 完整源 (含 sun50iw9p1 = H616/H700 die)。含 disp2/cedar-ve/vin/codec/mali-midgard 驱动。**缺**板级 dts。
- **[Knulli](https://github.com/knulli-cfw/distribution) `h700-mainline` 分支**: 2024-06 WIP 占位, 已被 Batocera 超越。
- **[ROCKNIX](https://github.com/ROCKNIX/distribution) / JELOS**: 应有完整适配 (从 patch 0024 `rocknix-dt-id` 推断), 可作第二参考。
- **[linux-sunxi/linux-sunxi](https://github.com/linux-sunxi/linux-sunxi)**: `sunxi-next` 已 6.12 合入 mainline, 之后直接看 [torvalds/linux](https://github.com/torvalds/linux)。

### 工具链备注

- 路线 C: 系统 GCC (aarch64-linux-gnu-gcc 15.2.0)
- 路线 B: `toolchain/gcc-linaro-5.3.1-2016.05-x86_64_aarch64-linux-gnu/` (与 Anbernic BSP 同源)
