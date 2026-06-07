# RG35XX-SP 主线（mainline）支持细节

> **来源等级声明**：本文档主要内容来自**对 mainline 源码树的静态检查**（TF-A lts-v2.14.2、U-Boot v2026.04、Linux 7.0.9 中的 `MAINTAINERS`、`configs/*defconfig`、`dts/`、`drivers/` 路径），属"源码侧二手分析"——不是设备实测、不是官方 PDF。
>
> 凡是涉及设备 BSP 现状的事实（原厂 PMIC 驱动、Mali kbase 版本、DTB 三选一、boot.img 格式、CPU OPP、thermal zone 等），以 `device-facts.md` 为准；本文档遇到这类事实时只引用条目编号，不重述。
>
> 关于"mainline 能否直接驱动 SP" 的最终判断仍需实机烧录验证（截至最后更新日未做）。

> 最后更新：2026-05-20
> 检查范围：TF-A lts-v2.14.2、U-Boot v2026.04、Linux 7.0.9

---

## 总览

| 组件 | 源码版本 | H616/H700 平台 | RG35XX-SP 板级 | 构建目标 |
|---|---|---|---|---|
| TF-A | lts-v2.14.2 | 完整 | 不需要板级区分 | `make PLAT=sun50i_h616 bl31` |
| U-Boot | v2026.04 | 完整（含 H700 DRAM） | DTS 有，缺专属 defconfig | `make anbernic_rg35xx_h700_defconfig` |
| Linux | 7.0.9 | 完整 | DTS 有，屏未 wire | `make defconfig` + 需补 panel DTS 节点 |

---

## TF-A（ARM Trusted Firmware）

**源码**：`tfa_src/lts-v2.14.2/`

### 平台支持

- `plat/allwinner/sun50i_h616/` 目录存在，含完整平台实现
- 构建目标：`PLAT=sun50i_h616`
- SoC ID：`SUNXI_SOC_H616 = 0x1823`（定义于 `plat/allwinner/common/include/sunxi_def.h`）

### 平台文件清单

| 文件 | 内容 |
|---|---|
| `platform.mk` | 构建配置，使用 `allwinner-common.mk`，BL31 驻留 DRAM，原生 PSCI（无 SCPI），加载 AXP805 PMIC + RSB 驱动 |
| `sunxi_power.c` | 电源管理 |
| `sunxi_idle_states.c` | idle 状态定义 |
| `sunxi_h616_dtb.c` | 运行时 DT 补丁：根据 H616 芯片版本动态修正 L2 cache 大小（早期 256KB，后期 1MB） |
| `include/sunxi_ccu.h` | CCU 寄存器定义 |
| `include/sunxi_cpucfg.h` | CPU 配置寄存器 |
| `include/sunxi_mmap.h` | 内存映射 |
| `include/sunxi_spc.h` | 安全分区控制器 |

### H700 与 H616 的关系

- TF-A 中**零 H700 引用**，统一走 `sun50i_h616` 平台
- H700 在 ATF 层与 H616 无差异，不需要额外适配
- 板级差异（如 RG35XX-SP 的 lid switch、RTC）全靠运行时传入的 DTS 处理
- 此结论与 `device-facts.md §10.3`（H700 = H616 同 die、仅封装不同）一致

### 与设备的关系

- 设备原厂 ATF 版本：BL31 `v1.0(debug):5a77824`（见 `device-facts.md §24`，含 `(debug)` 标记的安全影响说明）
- mainline TF-A 可直接替换原厂 ATF（BL31），与 U-Boot mainline 配合使用
- PSCI 实现是 mainline kernel cpuidle 正常工作的前提

---

## U-Boot

**源码**：`uboot_src/u-boot-2026.04/`

### 平台支持

- H700 不是独立 `MACH_` 选项，复用 `CONFIG_MACH_SUN50I_H616`
- H700 有独立的 DRAM PHY 地址映射：`CONFIG_DRAM_SUNXI_PHY_ADDR_MAP_1`（Kconfig help: "This pin mapping selection should be used by the H700"）
- DRAM 控制器：`CONFIG_DRAM_SUN50I_H616=y`

### defconfig

**唯一的 H700 defconfig**：`configs/anbernic_rg35xx_h700_defconfig`

| 参数 | 值 |
|---|---|
| `CONFIG_DEFAULT_DEVICE_TREE` | `allwinner/sun50i-h700-anbernic-rg35xx-2024` |
| `CONFIG_DRAM_CLK` | 672 MHz（与 `device-facts.md §5` BOOT0 实测一致） |
| `CONFIG_SUNXI_DRAM_H616_LPDDR4` | y（与 `device-facts.md §5` dram_type=8 一致） |
| `CONFIG_DRAM_SUNXI_PHY_ADDR_MAP_1` | y（H700 专用） |
| `CONFIG_AXP717_POWER` | y（mainline 对应 BSP 的 axp2202，见 `device-facts.md §10`） |
| DCDC2 电压 | 940 mV（与 `device-facts.md §15` 实测一致） |
| DCDC3 电压 | 1100 mV（与 `device-facts.md §15` 实测一致） |
| DRAM ODT | `0x08080808` |
| DRAM drive | `0x0e0e0e0e` |
| DRAM CA drive | `0x0e0e` |

**RG35XX-SP 无专属 defconfig**。要给 SP 用，需将 `CONFIG_DEFAULT_DEVICE_TREE` 改为 `allwinner/sun50i-h700-anbernic-rg35xx-sp`。

### DTS 文件

上游 DTS 目录 `dts/upstream/src/arm64/allwinner/` 中的 H700 设备树：

| 文件 | 说明 |
|---|---|
| `sun50i-h700-anbernic-rg35xx-2024.dts` | RG35XX 2024 基线 |
| `sun50i-h700-anbernic-rg35xx-plus.dts` | RG35XX Plus（include 2024） |
| `sun50i-h700-anbernic-rg35xx-h.dts` | RG35XX-H |
| `sun50i-h700-anbernic-rg35xx-sp.dts` | **RG35XX-SP**（include plus） |

所有 H700 DTS 均 include `sun50i-h616.dtsi` + AXP717 PMIC。

### MAINTAINERS

```
ANBERNIC RG35XX-2024
M:  Chris Morgan <macromorgan@hotmail.com>
S:  Maintained
F:  configs/anbernic_rg35xx_h700_defconfig
```

RG35XX-SP 无独立 MAINTAINERS 条目。

---

## Linux Kernel

**源码**：`linux_src/linux-7.0.9/`

### DTS 文件

`arch/arm64/boot/dts/allwinner/` 中的 H700 设备树：

| 文件 | 说明 |
|---|---|
| `sun50i-h700-anbernic-rg35xx-2024.dts` | RG35XX 2024 基线 |
| `sun50i-h700-anbernic-rg35xx-plus.dts` | RG35XX Plus（include 2024） |
| `sun50i-h700-anbernic-rg35xx-h.dts` | RG35XX-H |
| `sun50i-h700-anbernic-rg35xx-sp.dts` | **RG35XX-SP**（include plus） |

include 链：`sp → plus → 2024 → sun50i-h616.dtsi`

#### RG35XX-SP DTS 完整内容

```dts
// SPDX-License-Identifier: (GPL-2.0-only OR BSD-2-Clause)
/*
 * Copyright (C) 2024 Ryan Walklin <ryan@testtoast.com>.
 * Copyright (C) 2024 Chris Morgan <macroalpha82@gmail.com>.
 */

#include <dt-bindings/input/gpio-keys.h>
#include "sun50i-h700-anbernic-rg35xx-plus.dts"

/ {
	model = "Anbernic RG35XX SP";
	compatible = "anbernic,rg35xx-sp", "allwinner,sun50i-h700";

	gpio-keys-lid {
		compatible = "gpio-keys";

		lid-switch {
			label = "Lid Switch";
			gpios = <&pio 4 7 GPIO_ACTIVE_LOW>; /* PE7 */
			linux,can-disable;
			linux,code = <SW_LID>;
			linux,input-type = <EV_SW>;
			wakeup-event-action = <EV_ACT_DEASSERTED>;
			wakeup-source;
		};
	};
};

&r_i2c {
	rtc_ext: rtc@51 {
		compatible = "nxp,pcf8563";
		reg = <0x51>;
	};
};
```

SP 专属内容：lid switch（GPIO PE7，支持唤醒）+ 外置 PCF8563 RTC（R_I2C 0x51）。

### SoC dtsi（sun50i-h616.dtsi）关键节点

| 节点 | compatible | 说明 |
|---|---|---|
| `gpu@1800000` | `allwinner,sun50i-h616-mali`, `arm,mali-bifrost` | Mali-G31 GPU，走 Panfrost 驱动（BSP 侧用 `arm,mali-midgard` + kbase r20p0，见 `device-facts.md §3` / §20） |
| `iommu@30f0000` | `allwinner,sun50i-h616-iommu` | IOMMU（设备实测同址，见 `device-facts.md §6`） |
| `codec@5096000` | `allwinner,sun50i-h616-codec` | 音频 codec（设备实测同址，见 `device-facts.md §9`） |
| GPU thermal zone | — | 含 `gpu-trip-0` 临界温度 trip point；mainline DTS 只声明 cpu/gpu 两个 zone，BSP 实测有 4+1 个（见 `device-facts.md §13`） |

AXP717 PMIC 不在 SoC dtsi 中定义，在板级 DTS（2024/plus）中声明。

### 驱动支持

#### AXP717 PMIC（mainline 写法；与 BSP 私有 AXP2202 驱动栈的关系见 `device-facts.md §10`）

| 子系统 | 文件 | 状态 |
|---|---|---|
| MFD 核心 | `drivers/mfd/axp20x.c` | `AXP717_ID` 已注册，完整 regmap 范围 |
| 电池驱动 | `drivers/power/supply/axp20x_battery.c` | `axp717_batt_ps_desc` 完整（充放电/SOC/温度/电压/电流） |
| USB 电源 | `drivers/power/supply/axp20x_usb_power.c` | `axp717_usb_power_desc` 完整（VBUS 监测/电流限制/ADC） |
| Regulator | `drivers/regulator/axp20x-regulator.c` | AXP717 regulator 定义（DCDC1-3, LDO, BOOST, CPUSLDO） |

**注意**：BSP `axp2202` 私有驱动同时挂载了 `axp2101-regulator` 框架并报"unsupported AXP variant"错误（见 `device-facts.md §10` + §26）——mainline `axp20x_regulator` 的 AXP717 寄存器布局是否与 BSP 私有写入序列一致，需上电实测确认。

#### 显示引擎

| 组件 | 文件 | 状态 |
|---|---|---|
| DE3.3 mixer | `drivers/gpu/drm/sun4i/sun8i_mixer.c` | `SUN8I_MIXER_DE33` 路径已实现，compatible `allwinner,sun50i-h616-de33-mixer-0` |
| DE3.3 寄存器 | `drivers/gpu/drm/sun4i/sun8i_mixer.h` | `DE33_CH_BASE`(0x1000)、`DE33_CH_SIZE`(0x20000)、`DE33_VI_SCALER_UNIT_BASE`(0x4000) |
| TCON | `drivers/gpu/drm/sun4i/sun4i_tcon.c` | 存在 |
| HDMI | `drivers/gpu/drm/sun4i/sun8i_dw_hdmi.c` | 存在（SP 无 HDMI 口，见 `device-facts.md §7`） |

#### Panel 驱动

| 驱动 | 文件 | 状态 |
|---|---|---|
| NV3052C | `drivers/gpu/drm/panel/panel-newvision-nv3052c.c` | **存在**（SPI-3wire + RGB888，社区标称对应 WL-355608-A8——该屏型号本身未通过设备/官方 PDF 验证，见 `device-facts.md §7` 降级说明） |
| panel-mipi-dpi-spi | — | **不存在**（仅 Batocera patch 0009 有，未上游） |

#### GPU

| 驱动 | 目录 | 状态 |
|---|---|---|
| Panfrost | `drivers/gpu/drm/panfrost/` | 完整（panfrost_drv/gpu/job/device/gem + devfreq） |

#### 音频

| 驱动 | 配置 | 状态 |
|---|---|---|
| SUN50I codec analog | `CONFIG_SND_SUN50I_CODEC_ANALOG=m` | defconfig 已启用 |

#### 其他外设

| 驱动 | 配置 | 状态 |
|---|---|---|
| PCF8563 RTC | `CONFIG_RTC_DRV_PCF8563=m` | defconfig 已启用 |
| Panfrost GPU | `CONFIG_DRM_PANFROST=m` | defconfig 已启用 |
| DE mixer | `CONFIG_DRM_SUN8I_MIXER=m` | defconfig 已启用 |

### defconfig 覆盖情况

`arch/arm64/configs/defconfig` 中与 SP 相关的配置：

```
CONFIG_DRM_SUN8I_MIXER=m          # 显示引擎
CONFIG_DRM_PANFROST=m             # GPU
CONFIG_SND_SUN50I_CODEC_ANALOG=m  # 音频 codec
CONFIG_RTC_DRV_PCF8563=m          # 外置 RTC
```

### MAINTAINERS

无 RG35XX / H700 专属条目。

---

## 与原厂 BSP 的差异对照

> BSP 侧的事实条目均引自 `device-facts.md`，本表只对照差异，不重述事实。

| 维度 | 原厂 BSP 4.9.170 | mainline 7.0.9 |
|---|---|---|
| **PMIC** | `x-powers,axp2202` 私有双驱动栈（`device-facts.md §10`），HandsomeMod 树缺驱动（`verification-report.md §1`） | `x-powers,axp717` 标准 MFD 框架，完整 |
| **GPU** | Mali kbase r20p0 + libmali blob（`device-facts.md §20`），OpenGL ES 3.2 / Vulkan 1.1 | Panfrost（开源），OpenGL ES 3.1 / Vulkan 实验 |
| **显示引擎** | BSP disp2 完整（SmartColor 3.3 / AFBC / 6 通道 alpha / HDR——H700 brief 列出） | DE3.3 基础 blending（sun8i_mixer） |
| **视频解码** | Cedar VE 4K@60 H.265/VP9（闭源，H700 brief 列出） | cedrus 基础 H.264/H.265（无 4K@60） |
| **音频** | BSP Audio HUB v2（私有，见 `verification-report.md` SND_SOC_SUNXI 孤儿符号） | 标准 ASoC codec analog |
| **Panel** | BSP 私有 `fog_fj035fhd05_v1` 驱动（`device-facts.md §7`） | NV3052C（mainline）+ panel-mipi-dpi-spi（仅 Batocera） |
| **cpuidle** | BSP 私有 patch | 标准 PSCI/cpuidle |
| **DTB 选择** | U-Boot `lcd_type=<str>` 三选一（`device-facts.md §23`） | 单一 DTS，panel 驱动自行初始化 |
| **Boot 格式** | Android boot image v0（`device-facts.md §22`） | extlinux.conf 或标准 boot flow |
| **Thermal zone 数** | 4 (cpu/gpu/ve/ddr) + axp battery（`device-facts.md §13`） | mainline DTS 只声明 cpu + gpu |
| **CPU OPP 档数** | 9 档 480-1512 MHz（`device-facts.md §14`） | 与 sun50i-h616.dtsi 内 OPP 表一致需另查 |

---

## 当前差距（mainline 直接可用性）

> 以下"可直接工作"判断基于：mainline 源码侧驱动/DTS 存在性检查 + 与 `device-facts.md` 中 BSP 侧已验证硬件存在的交叉。**未经实机烧录验证**。

### 源码侧已具备（待实机验证）

- CPU / SMP 4 核（device-facts.md §1 实测设备有 4 核）
- PMIC（mainline AXP717）：regulator / battery / USB power（BSP 侧硬件存在见 §10/§15/§16/§17）
- SD 卡（mmc0）
- WiFi RTL8821CS（mmc1 SDIO）+ BT（uart1）（BSP 侧用厂商 8821cs 驱动，mainline 应改 `rtw88_8821cs`；固件不在 rootfs 中，需另装；见 `device-facts.md §18`）
- USB（ehci0 / ohci0 / usbotg）（BSP 侧 H616 USB 模块手册 `usb_drv_vbus_gpio="axp_ctrl"`，见 `device-facts.md §17`）
- UART0 console（BSP 侧 PH0/PH1，见 `device-facts.md §19`）
- 音频 codec（BSP 侧 0x05096000，见 `device-facts.md §9`）
- GPU（Panfrost）
- 外置 RTC（PCF8563）
- Lid switch
- 状态 LED、按键

### 需要额外工作的

1. **屏不亮**：mainline DTS 中 `&spi0`（panel SPI 命令通道）、`&tcon0`（时序控制器）、`&de`（显示引擎）、`backlight` 节点均未 wire——这是纯 DTS 层面的缺失，驱动本身已有（NV3052C）
2. **Panel 驱动选择**：mainline 的 `panel-newvision-nv3052c.c` 可用，但 Batocera 用的是 `panel-mipi-dpi-spi`（更灵活）。如果用 mainline NV3052C 驱动，需确认初始化序列与实际屏模组完全匹配——设备侧屏只有厂商代号 `fog_fj035fhd05_v1`（见 `device-facts.md §7`），与 NV3052C/WL-355608-A8 的对应关系**未独立验证**
3. **DE3.3 高级特性缺失**：SmartColor 3.3、AFBC、HDR EOTF、keystone、6 通道 alpha blend——这些在 mainline 的 sun8i_mixer 中未实现，对掌机游戏场景影响有限
4. **视频解码降级**：cedrus 仅基础 H.264/H.265，无 4K@60、无 VP9 profile2、无 AVS2
5. **libmali 不可用**：mainline 走 Panfrost，闭源 libmali blob 无法使用
