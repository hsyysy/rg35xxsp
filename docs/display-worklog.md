# RG35XX-SP 主线点屏(P0-3)工作记录

> 配套 `issues.md` P0-3「屏黑」。跟踪从调研 → 落地 → 点亮的全过程。
> 启动:2026-05-22。文档为活文档,有进展或发现先追加 §99「进展日志」。

---

> **2026-10-10 收尾：LCD 左右边缘回绕问题已关闭。根因是 TCON TOP 双 mixer 路由冲突，已通过同图、同时序单变量往返对照确认。7.2.9 原生包含上游修复，已部署并实际重启，路由自动为 `0x20`，未加载临时模块；用户确认正常测试图无回绕。后续升级按正常显示回归检查执行，详见 §99；旧日志中“改 HBP 已解决”的结论不适用于本次问题。**

## 00. 来源等级约定(2026-05-22 加)

文档中事实/判断按以下五档标注来源,可靠度由高到低:

| 标 | 含义 | 可靠度 | 典型 |
|---|---|---|---|
| `[源:厂商]` | 一手 — Allwinner / Anbernic 官方文档、原厂二进制(boot.img / DTB / U-Boot 字符串)、SoC datasheet、原厂 BSP kernel image 内含符号 | 最高 | H616 Datasheet、p4 dtb 抠出的 `lcd_driver_name` |
| `[源:实测]` | 一手 — `ssh root@DEVICE_HOST` 直读 `/sys /proc /dev/mem`、dd raw 分区、dmesg、串口日志、设备物理测量 | 高 | 当前 cmdline、`/sys/kernel/debug/pinctrl/...` 输出 |
| `[源:社区]` | 二手 — linux-sunxi wiki、Batocera / ROCKNIX、kernel 上游 commit、GitHub 公开仓 | 中 | Batocera 0002 patch、linux-sunxi H616 wiki |
| `[源:推断]` | 二手 — 我们综合多个来源推出的结论 / 跨设备类比 | 低 | 不同信息合并、未单独验证的因果 |
| `[源:假设]` | 二手 — 未经任何证据支持,仅作工作前提 | 待证 | 上电前的「应该可用」类判断 |

混合来源用 `[源:厂商+社区]` 等组合。**关键论断必须在结论后括号内标源**;描述性 / 项目工程性内容(任务清单、命令示例)不标。

---

## 0. 当前状态(2026-10-10)

下表的当前版本与验收状态已更新。带旧 commit、旧版本或里程碑日期的条目保留其历史含义；最新完整证据见 [路由修复记录](#lcd-route-fix-20261003)。

| 项 | 状态 |
|---|---|
| 阶段 | ✅ 7.2.9 已部署；原生 TCON TOP 路由修复，2026-10-10 重启后用户验收通过。 |
| 选定方案 | A(LCD-only + GPIO 背光,后续再升 PWM) |
| Panel 版本 | ✅ **v1**(`fog_fj035fhd05_v1`,640×480 — [源:厂商] p4 BSP dtb 实测;[源:实测] 用户 2026-05-22 二次确认物理分辨率) |
| GPIO 走线 | ✅ 5+1 个 GPIO 与 Batocera DTS 一致([源:厂商] p4 dtb `lcd_gpio_N` × [源:社区] Batocera 0002,见 §8) |
| LCD 时序(dtsi 侧)| ✅ 全部抠出([源:厂商] p4 dtb,见 §8.2) |
| H616 CCU clocks | ✅ 全齐(§4.6) |
| H616 SRAM-C 控制器 | 使用 a64 fallback；当前显示通路已通过实机验证。§4.7 为早期风险分析。 |
| Mainline panel 时序 | ✅ 已 fork `fog_fj035fhd05_v1_panel_info` + 新 compatible `anbernic,rg35xx-sp-panel`(§4.8) |
| Batocera patch 应用 | ✅ `git mailsplit + git am -3` 应用 11 个子 patch 全部干净(§99 #5) |
| Build break #1 | ✅ 修复 — sun8i_(vi|ui)_layer_init_one 跨模块调用需 EXPORT_SYMBOL_GPL |
| Build break #2 | ✅ 修复 — `sun8i_mixer ↔ sun50i_planes` 模块循环 → 合并到 sun8i-mixer.ko |
| KERNELRELEASE 干净度 | 开发基线与设备均为 `7.2.9`。构建显式传 `LOCALVERSION=`，模块版本须与目标内核一致。 |
| AXP717 cldo3 | ✅ enabled 3.3V(§4.10) |
| Panel reset 极性 | ✅ active-low,一致(§4.10) |
| NV3052C SPI init 序列 | 已从原厂 Image 重定位与反汇编恢复，前 189 对命令/数据与当前初始化数组完全一致；sleep-out 延迟位置差异另记。 |
| **产物** | 7.2.9 Image、DTB 与全量模块已构建并部署。旧 7.2.3 修复记录保留供历史追溯。 |
| 已落地的 Kconfig | `CONFIG_SUN50I_IOMMU=y`(P0-4)+ `CONFIG_DRM_PANEL_NEWVISION_NV3052C=m` + `CONFIG_SPI_GPIO=m` + `CONFIG_BACKLIGHT_GPIO=m` |
| 源码基线 | Linux 7.2.9 vanilla + 公开 BSP 设备适配补丁。 |
| 启动 BSP? | ✅ 2026-06-30 启动原厂镜像,确认 BSP 屏幕**正常无 1px 问题** → TCON 寄存器对比完成(见 §99 #14) |
| 调试设施 | ✅ `CONFIG_DRM_DEBUG_DRIVER=y` + `CONFIG_DRM_DEBUG=y` + 9 个 `[sun4i-drv-debug]` printk |
| 显示寄存器验证 | 早期层参数检查没有排除共享路由错误；此次已确认 `TCON_TOP_PORT_SEL=0` 冲突，修复后开机为 `0x20`。 |
| 左边缘 1px 偏移 | 已修复：右端白列回到右边界，左边界红列正常；修复前后的图像内存相同。 |
| **1px 根因定位** | **TCON TOP 双 mixer 路由冲突**；同图、同时序单变量往返复现，正式修复重启后通过。旧 DCLK/HBP 推断已更正。 |
| 下一步(显示) | LCD 边缘回绕问题已关闭，无待办修复；后续迁移 7.3 正式版时进行常规显示回归。 |

---

## 1. 三层缺失清单(根因审计,2026-05-22)

早期将适配概括为「~40 行 SP DTS overlay」是**低估**([源:实测] 检查 7.0.9 源码得到的反驳证据)。实际是 driver + dtsi + 板级 DTS + defconfig **四层**都缺。

### 1.1 驱动层(`linux/drivers/gpu/drm/sun4i/` + `panel/` + `pwm/`)

[源:实测] 在本地 `linux-7.0.9` 树 grep 各 `compatible` 表得出:

| 文件 | 已支持 | 缺失 |
|---|---|---|
| `sun8i_mixer.c:925` | ✅ `allwinner,sun50i-h616-de33-mixer-0` | — |
| `sun4i_tcon.c` 兼容表 | `sun8i-r40-tcon-tv`、`sun20i-d1-tcon-lcd/tv` | ❌ `sun50i-h616-tcon-lcd`、`sun50i-h616-tcon-tv`、`sun8i-r40-tcon-lcd` |
| `sun8i_tcon_top.c:284` | `sun8i-r40-tcon-top`、`sun50i-h6-tcon-top` | ❌ `sun50i-h616-tcon-top`(可考虑 h6 fallback) |
| `sun4i_drv.c` 兼容表 | — | ❌ `allwinner,sun50i-h616-display-engine` |
| `pwm-sun4i.c:368` | `sun50i-h6-pwm` | ❌ H616 / D1 PWM(仅 PWM 背光路径需要) |
| `sun8i_vi_layer.c` | DE2 完整 | ❌ DE33 plane/format 限制 |
| `sun8i_planes.c` | 文件不存在 | ❌ `sun50i-h616-de33-planes` driver |
| `panel-newvision-nv3052c.c:666` | ✅ `anbernic,rg35xx-plus-panel` → `wl_355608_a8_panel_info` | ❓ `anbernic,rg35xx-sp-v2-panel` 缺(若设备是 v2) |
| `panel-newvision-nv3052c.c:644` | `wl_355608_a8` 寄存器序列 + 显示模式 | 实测尚未对照 `fog_fj035fhd05_v1` 厂商序列 — 见 §4.1 |

### 1.2 dtsi 层(`linux/arch/arm64/boot/dts/allwinner/sun50i-h616.dtsi`)

整个文件**没有任何显示节点**。需添加:
- `de: display-engine`(顶层 / 节点)
- `bus@1000000`(DE33 总线 — display_clocks / planes / mixer0 / mixer1)
- `de3_sram` SRAM-C 子节点
- `hdmi@6000000` + `hdmi_phy@6010000`(仅 HDMI 路径需要)
- `tcon_top@6510000`、`tcon_lcd0@6511000`、`tcon_tv0@6515000`
- `lcd0_rgb888_pins` 与 `lcd0_lvds_pins` pinctrl 组(LCD 走 PD bank)
- `pwm@300a000` 控制器节点(仅 PWM 背光路径)

### 1.3 板级 DTS 层(`sun50i-h700-anbernic-rg35xx-{2024,plus,sp}.dts`)

应改 **`2024.dts`**(SP 通过 `#include` 链继承,无须在 sp.dts 重复声明):
- `backlight` 节点:`gpio-backlight` → PD28(后续升 `pwm-backlight` 走 pwm0)
- `reg_lcd`:`regulator-fixed`,3.3V,enable PI15
- `spi_lcd`:`spi-gpio`,sck=PI9 / mosi=PI10 / cs=PI8
- `panel: panel@0`:compatible 见 §4.1,reset=PI14,spi-3wire,3125000 Hz,pinctrl=`<&lcd0_rgb888_pins>`,backlight=`<&backlight>`,power-supply=`<&reg_lcd>`,output 端点接 `tcon_lcd0_out_lcd`
- `&pio { vcc-pd-supply = <&reg_cldo3>; }`(LCD RGB 走 PD bank,必须有电)
- `&de { status = "okay"; };`
- `&tcon_lcd0 { status = "okay"; }; &tcon_lcd0_out_lcd { remote-endpoint = <&panel_in_rgb>; };`

### 1.4 defconfig + rg35xxsp.config

| Kconfig | 当前 | 需要 |
|---|---|---|
| `CONFIG_DRM_SUN4I=m` | ✅ defconfig | — |
| `CONFIG_DRM_SUN8I_MIXER=m` | ✅ defconfig | — |
| `CONFIG_DRM_SUN8I_DW_HDMI=m` | ✅ defconfig | — |
| `CONFIG_PWM_SUN4I=m` | ✅ defconfig | — |
| `CONFIG_BACKLIGHT_PWM=m` | ✅ defconfig | — |
| `CONFIG_REGULATOR_FIXED_VOLTAGE=y` | ✅ defconfig | — |
| `CONFIG_REGULATOR_GPIO=y` | ✅ defconfig | — |
| `CONFIG_DRM_PANEL_NEWVISION_NV3052C` | ❌ 未启 | **=m**(`rg35xxsp.config` 加) |
| `CONFIG_SPI_GPIO` | ❌ 未启 | **=m**(panel 命令通道) |
| `CONFIG_BACKLIGHT_GPIO` | ❌ 未启 | **=m**(方案 A 首版背光) |

---

## 2. Batocera 可复用 patch 清单(Batocera H700 补丁集)

[源:社区] 二手来源,Batocera 团队针对 6.18-context 提交,需 rebase 到我们的 7.0.9 上下文。按 LCD 路径必要性筛选。Batocera `0002-rg35xx-enable-HDMI-LCD.patch` 是一个 21 子 patch 的合并文件,我们按需抽。

| Patch (子序号) | 标题 | LCD-only(方案 A) | + HDMI(方案 C) |
|---|---|---|---|
| 0002 01/21 | `drm/sun4i: vi_layer: Limit formats for DE33` | ✓ | ✓ |
| 0002 02/21 | `clk: sunxi-ng: de2: Export register regmap for DE33` | ✓ | ✓ |
| 0002 03/21 | `drm/sun4i: Add planes driver` | ✓ | ✓ |
| 0002 04/21 | `drm/sun4i: switch DE33 to new bindings` | ✓ | ✓ |
| 0002 05/21 | `drm/sun4i: Add H616 TCON TV support` | ✗ | ✓ |
| 0002 06/21 | `drm/sun4i: Add support for H616 HDMI PHY` | ✗ | ✓ |
| 0002 07/21 | `drm/sun4i: Add compatible for H616 display engine` | ✓ | ✓ |
| 0002 08/21 | `arm64: dts: h616: Add display pipeline`(+308 行 dtsi) | ✓ | ✓ |
| 0002 09/21 | `arm64: dts: h616: Enable HDMI on several boards` | ✗ | ✓(可选,只动他厂板,不动 RG35XX) |
| 0002 10/21 | `dt-bindings: TCON_TOP_LCD clock defines` | ✓(bindings 不强制,但 patch 14 引用) | ✓ |
| 0002 11/21 | `dt-bindings: H616 DE33 bus binding` | 可省(YAML) | 可省 |
| 0002 12/21 | `dt-bindings: sun4i: compatible strings` | 可省 | 可省 |
| 0002 13/21 | `dt-bindings: sun4i: TCON compatibles` | 可省 | 可省 |
| 0002 14/21 | `dt-bindings: sun4i: add R40/H616` | 可省 | 可省 |
| 0002 15/21 | `dt-bindings: sram: H616 SRAM C` | 可省 | 可省 |
| 0002 16/21 | `drm/sun4i: tcon: add support for R40 LCD` | ✓(给 `sun8i-r40-tcon-lcd` 提供 quirks,h616 走它 fallback) | ✓ |
| 0002 17/21 | `arm64: dts: h616: add LCD and LVDS pins`(`lcd0_rgb888_pins`) | ✓ | ✓ |
| 0002 18/21 | `arm64: dts: h616: add lcd tcon endpoint` | ✓ | ✓ |
| 0002 19/21 | `arm64: dts: rg35xx-2024: Enable LCD output`(reg_lcd / spi_lcd / panel / &de / &tcon_lcd0) | ✓ | ✓ |
| 0002 20/21 | `arm64: dts: rg35xx: add HDMI connector` | ✗ | ✓ |
| 0002 21/21 | `arm64: dts: rg35xx: add GPIO backlight`(PD28,default-on) | ✓ | ✓ |
| 0005 | `sun20i: add pwm driver`(新 pwm-sun20i-d1.c) | ✗(方案 B 才需) | ✗ |
| 0006 | `h616: add pwm node` | ✗ | ✗ |
| 0007 | `rg35xx: enable pwm backlight`(覆盖 21/21 的 gpio-backlight) | ✗(方案 B 才需) | ✗ |
| 0013 | `add rg35xx panel variants dts`(新增 `sun50i-h700-anbernic-rg35xx-sp-v2-panel.dts`) | ❓(仅当 §4.1 测出 v2 才需要) | — |

**方案 A 最终需要应用的 patch**:0002 子 01/02/03/04/07/08/16/17/18/19/21 共 **11 个子 patch**。

> 备注:Batocera `0099-combined-fix-for-6.18-context.patch` 是 hsyysy(2026-05-20)为 6.18 上下文重打的 DTS-only 合并版,只动 DTS 不含驱动 patch,**前置假设 0002 + 0007 已应用**。我们 7.0.9 不能直接套 0099。

---

## 3. 三个方案对比(2026-05-22 选定方案 A)

| 方案 | 范围 | 工作量 | 风险 | 用户选择 |
|---|---|---|---|---|
| **A** | LCD only + GPIO 背光 | 11 个 patch + 3 个 Kconfig + 0 行 SP DTS | 中(0002 在 7.0.9 上下文上可能有 reject,需逐 patch 修偏移) | ✅ 已选 |
| B | A + PWM 背光(亮度可调) | A + 3 个 patch(0005/0006/0007)+ 1 个新 pwm 驱动 | 中-高(pwm-sun20i 跟 pwm-sun4i 可能冲突,需取舍 Kconfig) | 后续升级 |
| C | A + HDMI 双路 | A + 4 个 patch(子 05/06/09/20) | 中(掌机不依赖 HDMI,坞站才用) | 不做 |

---

## 4. 不确定项 → 已用原厂 BSP 数据消除的部分(2026-05-22 更新)

### 4.1 panel 版本 / compatible — ✅ **确定 v1** [源:厂商]

通过 ssh root@DEVICE_HOST 拉 p4 boot 镜像反编译三个候选 dtb(`lcd_type=0/1/2`):

| dtb | 偏移 | `lcd_driver_name` | 分辨率 | `lcd_dclk_freq` | 适用机型 |
|---|---|---|---|---|---|
| 0 | 0x1334800 | `fog_fj035fhd05_v1` | **640×480** | 24 MHz | RG35XX-SP(当前实测 `lcd_type=old` 走这个) |
| 1 | 0x135B800 | `rg34xxsp_v1` | 720×480 | 26 MHz | RG34XX-SP 横屏机型(与我们无关) |
| 2 | 0x13B8800 | `fog_fj035fhd05_v1` | **640×480** | 24 MHz | RG35XX-SP 时序微调备份 |

**LCD 物理分辨率 = 640×480**([源:厂商] dtb 0/2 `lcd_x=0x280 lcd_y=0x1e0`;[源:实测] 用户 2026-05-22 二次确认)。`fog_fj035fhd05_v1` 是厂商代号,**社区**映射为 NV3052C + WL-355608-A8([源:社区] linux-sunxi、Batocera 团队,未独立验证)— mainline `panel-newvision-nv3052c.c:666` 的 `anbernic,rg35xx-plus-panel → wl_355608_a8_panel_info` 直接可用,**不需要 Batocera 0013 patch**([源:推断])。

### 4.2 GPIO 走线 v1/v2 一致性 — ✅ **完全一致** [源:厂商+社区交叉]

[源:厂商] 原厂 dtb 0 抠出的 `lcd_gpio_0..4`(格式 `<port pin mode mul drive pull>`,port=0x08='I'):

| BSP 用途 | port,pin | 等价 mainline | Batocera 推断 | 一致? |
|---|---|---|---|---|
| `lcd_gpio_0` SPI sck | I, 9 | **PI9** | PI9 | ✅ |
| `lcd_gpio_1` SPI mosi | I, 10 | **PI10** | PI10 | ✅ |
| `lcd_gpio_2` SPI cs | I, 8 | **PI8** | PI8 | ✅ |
| `lcd_gpio_3` panel reset | I, 14 | **PI14** | PI14 | ✅ |
| `lcd_gpio_4` LCD VDD enable | I, 15 | **PI15** | PI15 | ✅ |
| 背光 | `lcd_pwm_ch=0`, `lcd_pwm_freq=50000`, `lcd_pwm_pol=1` | PD28 / pwm0 / 50 kHz / active-high | PD28 / pwm0 / 40 kHz | ✅ pin 匹配,频率需用 BSP 的 50 kHz |

### 4.3 `vcc-pd-supply` — 仍必要 [源:推断]

[源:推断] BSP 自己实现的 disp 不走 mainline pinctrl framework,直接驱动 PD bank;mainline 必须给 `&pio.vcc-pd-supply = <&reg_cldo3>;`,否则 RGB pins 没电。该推断的旁证:Batocera 0002 patch 19/21 也加了这行([源:社区])。

### 4.4 NV3052C `wl_355608_a8_panel_regs[]` ↔ `fog_fj035fhd05_v1` init 序列 — ✅ **兼容(实测确认,TTY 显示正常)** [源:实测]

**2026-05-22 调研结论**:
- [源:厂商] 原厂 init 序列在 p4 boot.img 内,函数符号名 `lcd_panel_fog_fj035fhd05_v1_init` 在 BSP 4.9.170 kernel ARM64 Image 中确实存在(字符串偏移 `0x974a3a`,panel 结构 `__lcd_panel.name[32]` 在 `0xf8c580`)
- [源:实测] 本地三棵 BSP 树(`linux-4.9.118` mainline、`linux-allwinner-4.9` 标 BSP、`linux-allwinner-4.9.170` 用户自建混合)都**没有** `fog_fj035fhd05_v1.c` 源码 — Anbernic vendor 私有 patch,未公开
- [源:实测] GitHub 公开搜索亦无该函数名命中(2026-05-22 Web Search)
- [源:推断] 反汇编 17 MB ARM64 Image 拼 SPI 寄存器序列工作量 > 一上午,**不划算**

**判断** [源:推断]:同 NV3052C controller、同 640×480 分辨率、同 Plus/SP 屏模组家族 — mainline `wl_355608_a8_panel_regs[]`([源:社区] macroalpha82 来自 RG35XX-Plus 实测提交)大概率直接可用。

**升级路径**(若首次上电出现白屏/花屏/竖纹/色差,顺序优先):
1. 先看 dmesg 是否真有 `panel-newvision-nv3052c ... panel: powered on` — probe 不成功是 DTS / clock / regulator 问题,不是 init 序列
2. 拍屏判断:**全白** = panel 没 DCS exit_sleep → 检查 `wl_355608_a8_panel_regs[]` 是否含 0x11 `EXIT_SLEEP_MODE` cmd;**花屏/竖纹** = init 序列对但时序不对 → §8.2 BSP 时序参数对照 `wl_355608_a8_mode[0]`;**色彩翻转/反相** = gamma / `lcd_frm` / dithering 差异
3. 真不行才反汇编:`aarch64-linux-gnu-objdump -D -b binary -m aarch64 --adjust-vma=0xffff800010080000 orig-kernel.bin 2>/dev/null | grep -B 1 -A 30 lcd_panel_fog`
4. 或物理示波:逻辑分析仪抓 PI9/10 上 panel power-on 前 50 ms 的 SPI 序列

### 4.5 TCON-LCD 走 `sun8i-r40-tcon-lcd` fallback — ⏳ 上电试 [源:假设]

[源:社区] Batocera 0002 patch 16/21 给 `sun8i-r40-tcon-lcd` 加 quirks,h616 走 fallback;[源:假设] 不兼容时 dmesg 会有 timing 配置失败信息,需上电验证。

### 4.6 H616 CCU 中 LCD 相关 clock — ✅ **全部齐备** [源:实测]

[源:实测] grep `linux-7.0.9/drivers/clk/sunxi-ng/ccu-sun50i-h616.c`:

| 时钟 ID | clock 名 | CCU 寄存器偏移 | parents |
|---|---|---|---|
| `CLK_TCON_LCD0` = 130 | `tcon-lcd0` | 0xb60 | `pll-video0/0-4x/1/1-4x` |
| `CLK_BUS_TCON_LCD0` = 131 | `bus-tcon-lcd0` | 0xb7c bit 0 | ahb3 |
| `CLK_TCON_TV0` = 119 | `tcon-tv0` | 0xb80 | 同上 |
| `CLK_BUS_TCON_TV0` = 121 | `bus-tcon-tv0` | 0xb78 bit 0 | ahb3 |
| `CLK_BUS_TCON_TOP` | `bus-tcon-top` | 0xb5c bit 0 | ahb3 |
| `pll-video0/1/2` | `pll-video[0-2]` | — | osc24M |
| `pll-video[0-2]-4x` | 4x 倍频 | fixed factor | — |

Batocera 0002 patch 08/21 引用的 `<&ccu CLK_TCON_LCD0>`、`<&ccu CLK_BUS_TCON_LCD0>`、`<&ccu CLK_TCON_TV0>`、`<&ccu CLK_BUS_TCON_TV0>`、`<&ccu CLK_BUS_TCON_TOP>` **全都能编译通过**。无阻塞。

### 4.7 SRAM-C controller H616 支持 — ⚠️ **走 a64 fallback** [源:实测]

[源:实测] grep `linux-7.0.9/drivers/soc/sunxi/sunxi_sram.c`:

- 仅注册 `allwinner,sun50i-a64-sram-c`(line 99)+ `allwinner,sun50i-a64-sram-controller`(line 428)
- **没有** `sun50i-h616-sram-c` compatible
- Batocera 0002 patch 08/21 写 `de3_sram` 兼容串为 `"sun50i-h616-sram-c", "sun50i-a64-sram-c"` 双串 → kernel 走 a64 fallback driver

[源:推断] 风险:**a64 的 SRAM-C 子驱动在 H616 上是否对 DE33 mixer 真的能 claim/release**。a64 走 sun8i-h3 风格的 SRAM region 协议,H616 SRAM-C 物理上一致但寄存器 layout 可能微差。**上电验证**:首屏失败时看 dmesg 是否有 `sun50i-a64-sram-c.*alloc.*fail` 类信息。

### 4.8 mainline `wl_355608_a8_mode[0]` ↔ BSP dtb 时序 — ❌ **不一致,必须 fork** [源:实测]

[源:实测] `linux-7.0.9/drivers/gpu/drm/panel/panel-newvision-nv3052c.c:607-620`:

2026-10-04 勘误：下表 BSP 列已按 `lcd_hbp/lcd_vbp` **包含同步宽度**换算成 DRM 的纯前后沿；该换算说明不代表本轮修改了运行中的模式时序。

| 参数 | mainline（当时版本） | BSP dtb 换算值 | diff |
|---|---|---|---|
| clock (kHz) | 24000 | 24000 | ✅ |
| hdisplay | 640 | 640 | ✅ |
| hfp(hsync_start − hdisplay) | **64** | **32** | ❌ |
| hsync(hsync_end − hsync_start) | **20** | **8** | ❌ |
| hbp(htotal − hsync_end) | **46** | **40** | 接近 |
| htotal | **770** | **720** | ❌ -50 |
| vdisplay | 480 | 480 | ✅ |
| vfp | **21** | **32** | ❌ |
| vsync | 4 | 4 | ✅ |
| vbp | **15** | **44** | ❌ |
| vtotal | **520** | **560** | ❌ +40 |
| 帧率 | 59.88 Hz | 59.52 Hz | 微差 |
| HSYNC/VSYNC 极性 | both NEG ([源:实测] DRM_MODE_FLAG_NHSYNC + NVSYNC) | both 低 ([源:厂商] `lcd_hv_sync_polarity=3`) | ✅ |

[源:推断] 同一 NV3052C panel + 同一 640×480 分辨率,但 mainline 的 mode 来自 RG35XX-Plus(2024)实测,跟我们 SP 厂商实测有显著差异。**用 mainline mode 大概率花屏/无信号**。

**修复**(已加入 §10 落地清单):
- 选项 A:直接 patch `wl_355608_a8_mode[0]` 改成 BSP 时序值 — 影响 RG35XX-Plus 用户(但他们没在我们 kernel 上跑,无关)
- 选项 B(更稳):新增 `fog_fj035fhd05_v1_panel_info`(单独 mode array)+ 新 compatible `anbernic,rg35xx-sp-panel`,加进驱动的 `nv3052c_of_match[]`,板级 DTS 用新 compatible
- 选 B,顺便给后续 panel 变体留扩展空间

### 4.9 Batocera `0002` 用 `patch -p1` 不能整体应用 — ⚠️ **必须 git am** [源:实测]

[源:实测] 2026-05-22 dry-run `patch -p1 --dry-run < 0002-rg35xx-enable-HDMI-LCD.patch`:

- 大部分 hunks 仅 offset(`succeeded at X (offset N lines)`)
- **1 个 hunk FAILED**:`sun50i-h616.dtsi:1154` —— 这是 sub-patch 18/21 给 `tcon_lcd0_out` 加 sub-endpoint。失败原因:**sub-patch 18 假设 sub-patch 08(添加 tcon_lcd0 节点)已应用**,但 `patch -p1` 一次性扫整文件,无法处理 sub-patch 间顺序依赖

[源:推断] 解决方案:**用 `git am` 把 21 个子 patch 拆成 21 个独立 commit 顺序 apply**。Patch 已是 mailbox 格式(每子 patch 含 `From ... Subject:` 头),`git mailsplit + git am -3` 直接可用。

### 4.10 Panel reset 极性 + AXP717 cldo3 实测 — ✅ [源:厂商+实测]

- **Reset 极性**:[源:厂商] BSP dtb `lcd_gpio_3 = <0x08 0x0e 0x01 ... 0x01>` 末位 `data=0x01`(默认 high = deassert)→ active-low;[源:实测] mainline NV3052C `gpiod_set_value(reset, 1)` 后跟 `set_value(reset, 0)` + Batocera DTS `GPIO_ACTIVE_LOW` → 一致
- **`reg_cldo3`(vcc-io,给 PD bank 供电)**:[源:实测] `ssh root@DEVICE_HOST 'cat /sys/class/regulator/.../name,state,microvolts'` → `vcc-io enabled 3300000` ✅(P0-7 AXP DMA mask 问题不影响 cldo3 输出)

---

## 8. 原厂 BSP 关键数据汇总(2026-05-22 抠出)

[源:厂商] 一手数据。来源:`ssh root@DEVICE_HOST 'dd if=/dev/mmcblk0p4 ...'` + `dtc -I dtb -O dts`,见 §99 日志 2026-05-22 #2。

### 8.1 三个候选 dtb 在 p4 boot.img 的位置

```
Android bootimg (32 MB) 内 dtb magic d00dfeed 出现位置:
  offset 0x1334800 size 0x21964 → lcd_type=0 (=old, 实测)  fog_fj035fhd05_v1 640×480 24 MHz
  offset 0x135B800 size 0x219FB → lcd_type=1               rg34xxsp_v1 720×480 26 MHz (RG34XX-SP)
  offset 0x13B8800 size 0x21A32 → lcd_type=2               fog_fj035fhd05_v1 640×480 24 MHz
```

### 8.2 RG35XX-SP LCD 完整时序(从 dtb 0 抠)

```
lcd_driver_name = "fog_fj035fhd05_v1"
lcd_if = 0           # RGB parallel
lcd_hv_if = 0        # HV parallel
lcd_x = 640
lcd_y = 480
lcd_width = 0x96     # DT 原始声明值 150，未经物理尺寸核验
lcd_height = 0x5e    # DT 原始声明值 94，不能据此推导 3.5" 物理尺寸
lcd_dclk_freq = 24   # MHz
lcd_hbp = 48         # 含 hsync 的后沿区间；纯后沿 40 + hsync 8
lcd_ht = 720         # active 640 + 纯后沿 40 + hsync 8 + 前沿 32
lcd_hspw = 8         # h sync pulse width
lcd_vbp = 48         # 含 vsync 的后沿区间；纯后沿 44 + vsync 4
lcd_vt = 560         # active 480 + 纯后沿 44 + vsync 4 + 前沿 32
lcd_vspw = 4         # v sync pulse width
lcd_hv_clk_phase = 2
lcd_hv_sync_polarity = 3   # HSYNC + VSYNC both active-low
lcd_frm = 1                # BSP FRM 配置；mainline 也有 FRM 控制，当前实测关闭
lcd_gamma_en = 0
lcd_bright_curve_en = 0

# 计算帧率: 24e6 / (720 * 560) = 59.52 Hz ≈ 60 Hz ✓
```

### 8.3 PWM 背光参数(BSP dtb 0,方案 B 用)

```
lcd_pwm_used = 1
lcd_pwm_ch = 0           # 走 PWM0
lcd_pwm_freq = 50000     # 50 kHz (Batocera 用 40 kHz, 应改 50 kHz 对齐原厂)
lcd_pwm_pol = 1          # active-high
lcd_pwm_max_limit = 0xff # 8-bit 亮度
lcd_backlight = 0x32     # 默认亮度 50/255 ≈ 20%
```

### 8.4 SoC 信息

```
SMCCC SOC_ID: jep106:091e:1823 Revision 0x00000002   # H616/H700 silicon rev 2
```

[源:实测] `dmesg | grep SMCCC` 当时 mainline 7.0.9 启动日志直接读到。

---

## 9. 决策:是否启动原厂 BSP 镜像?(2026-05-22)

### 9.1 触发问题

用户提:既然原厂 4.9.170 在 p4 还在,启动它能不能从运行时拿到比当前 mainline 更有用的数据,用来辅助主线点屏?

### 9.2 BSP 启动能拿到的数据(按价值倒序)

| # | 项 | 价值 | 拿法 | 来源标 |
|---|---|---|---|---|
| 1 | **panel SPI 命令字节序列**(消除 §4.4 风险的唯一直接证据) | ⭐⭐⭐⭐ | ❌ **拿不到**(§9.4 详释) | [源:推断] |
| 2 | TCON_LCD 寄存器实际值 | ⭐⭐⭐ | `devmem 0x6511000`,验证 §4.5 | [源:推断] |
| 3 | pinctrl PD0..23 实际 function 值 | ⭐⭐⭐ | `/sys/kernel/debug/pinctrl/.../pinmux-pins` | [源:推断] |
| 4 | `clk_summary` 显示链路实际 clock 树 | ⭐⭐⭐ | `/sys/kernel/debug/clk/clk_summary` | [源:推断] |
| 5 | LCD 上电时序窗口(reset / VDD / SPI 延迟) | ⭐⭐ | dmesg | [源:推断] |
| 6 | regulator 实际工作电压 + 启停顺序 | ⭐⭐ | `/sys/kernel/debug/regulator/regulator_summary` | [源:推断] |
| 7 | PWM 寄存器实际工作值 | ⭐ | `devmem 0x300a000` | [源:推断] |

[源:推断] 上述清单基于本文档作者对 sunxi BSP / disp2 框架的了解推出,**未经实测**(因 9.3 障碍未启动 BSP)。

### 9.3 启动 BSP 的代价

| 组件 | 当前 | 恢复手段 | 可恢复? | 来源 |
|---|---|---|---|---|
| SPL @ sector 16 | mainline 2026.04 | `device-backup/prehead-36m.bin` | ✅ | [源:实测] device-backup 已存在 |
| U-Boot / ATF | mainline | 同上 | ✅ | 同上 |
| p2 boot FAT | mainline | `device-backup/p2-orig.tar` | ✅ | 同上 |
| p4 Android boot image | 原厂保留 | 不动 | ✅ | [源:实测] dd 读出验证 |
| **p5 rootfs** | Ubuntu 22.04 | 原厂 4.9.170 rootfs | **❌** | [源:厂商+issues.md P2-2] `/lib/modules/4.9.170` 在首次试刷时被 `tar -xzf` 误删,即使其他都回滚,modules 也没法重新生成 |

[源:推断] 即使刷回 SPL + U-Boot + p2,4.9.170 kernel 启动会在 `modprobe sunxi_disp` / `sunxi_lcd` 阶段失败(模块全失踪),无法跑到 LCD 阶段。**实际启动 BSP 需要重新刷一张干净的原厂 TF 卡**(物流和折腾时间 ≥ 1-2 天,或一台备机做对比)。

### 9.4 当时对 panel SPI 命令序列可获取性的评估

> 2026-10-04 更新：本节关于正常启动难以捕获历史 SPI 流量的讨论仍有适用条件；“原厂序列拿不到”不是最终结论。已结合 Image 的 RELATIVE 重定位成功反汇编恢复初始化序列，见最新记录。

[源:推断] 即便能启动 BSP:
- BSP 内核无源 → 不能加 dynamic_debug
- disp2 框架内 SPI 调用([源:实测] 见 §99 #3 — `__lcd_panel.name[32]` 后续 8 个槽位全 0)不暴露 byte log,真正的 ops 表通过 `panel_array[]` 间接索引
- ftrace 函数级跟踪能跟到 `lcd_panel_fog_fj035fhd05_v1_init`,但参数(SPI command byte)不在 ARM64 ftrace 调用栈
- **唯一可靠路径:物理逻辑分析仪抓 PI9/PI10**(BSP 启动与否对此无影响)

### 9.5 mainline 7.0.9 上能拿到什么(2026-05-22 实测)

[源:实测] `ssh root@DEVICE_HOST` 在当时运行的 mainline 7.0.9 上跑同样命令的结果:

| 数据 | mainline 7.0.9 上能否拿 | 说明 |
|---|---|---|
| pinctrl PD0..23 实际 mux | ❌ 全部 `UNCLAIMED` | 显示流水线没起,pinctrl 没接管 |
| `clk_summary` 显示链路 | ❌ `tcon_lcd0` / `tcon_tv0` / `pll-video` 都没出现 | h616.dtsi 缺这些节点(§1.2) |
| `regulator_summary` cldo3 | ❌ 输出空 | axp717 mfd 树没正确构建(关联 P0-7 AXP DMA mask) |
| `/dev/mem` 直接读 TCON 物理地址 | ✅ 可读但 register state 是 reset 值 | TCON 没被任何驱动初始化 |
| `lsmod` 显示子系统 | ✅ panfrost / drm_kms_helper 加载,但 sun4i-drm 未加载 | 没 display engine 设备 → 不会 probe |

[源:推断] 结论:**当前 mainline 状态下读 sysfs 数据取代不了 BSP**,因为子系统没 probe。但**只要方案 A 落地、显示子系统起来**,这些 sysfs / debugfs 数据就自动可见,且**可与 mainline 自己的配置直接对比**(比与 BSP 对比更有针对性:对比的就是我们写的 DTS / driver 的实际生效结果)。

### 9.6 决策

**❌ 不启动 BSP**。理由按重要性 [源:推断]:

1. 最高价值数据(SPI 序列)启 BSP **也拿不到**(9.4)— 代价不抵收益
2. 启动 BSP 需要找新 TF 卡 + 重刷原厂固件(9.3,p5 rootfs 无备份)— 物流 ≥ 1-2 天
3. 次要数据(TCON / pinctrl / clk_summary / regulator)等方案 A 落地后跑 mainline 同样可读(9.5),且对比更直接
4. 若首屏失败,那时再决定是否启 BSP 作对照系作 fallback;现在过早

### 9.7 备用方案(若首屏失败 / §9.6 路线走不通)

按代价从低到高 [源:推断]:

| 路径 | 估算工作量 | 设备需求 |
|---|---|---|
| (a) 反汇编 17 MB ARM64 Image 抠 SPI 序列:`aarch64-linux-gnu-objdump -D -b binary -m aarch64 ... | grep -B 1 -A 30 lcd_panel_fog` | 半天 | 无 |
| (b) 物理逻辑分析仪抓 PI9/PI10(panel power-on 前 50 ms) | 几小时(假设有 LA) | Saleae / DSLogic 等 |
| (c) 重刷原厂 TF 卡作对照系(完整恢复) | 1-2 天物流 + 1 小时刷卡 | 干净 TF 卡 + 网购的原厂固件 |

---

## 10. 当前构建入口

历史实验曾拆分并选择性移植 Batocera 补丁。公开 BSP 已维护适用于 7.2.9 的累积适配补丁，
使用方法见[主线指南](mainline-guide.md)。无需按旧实验步骤重复叠加补丁。


## 11. 相关引用

- `issues.md` P0-3 — 原始问题描述
- `issues.md` P0-4 — 同期 IOMMU/panfrost 修(独立问题,共享 rebuild)
- `mainline-support.md` 第 212/263/292 行 — Panel 驱动状态、DE3.3 高级特性差距
- `device-facts.md` §7 / §10.4 / §23 — LCD 物理事实、lcd_type 三选一
- Batocera 的 HDMI / LCD 适配补丁 — 主要参考 [源:社区]
- Batocera 后续适配补丁 — 后续升级 / panel 变体 [源:社区]

---

## 12. 补丁维护

公开源码、配置与补丁见 [BSP 仓库](https://github.com/hsyysy/bsp-rg35xxsp)。
上游版本由 submodule 固定；设备适配由独立补丁维护，7.2.9 已含的路由修复不重复回移。


## 99. 进展日志(倒序,最新在上)

### 2026-10-11 — 控制台与显示回归

- fbcon 与前端共用 UI 主层；通过解绑 framebuffer vtconsole、将前端输出接入 journal，避免界面被文本覆盖。整机重启已复核。[源:实测]
- 开发阶段 LCD 测试覆盖三个平面和 64 个平面 / 格式组合；143 项提交通过，六项缩放组合返回 ERANGE。自动提交检查不代替逐图案人工验收。[源:实测]
- 测试图案及 Start 返回前端的交互已有人工交互验证；该测试程序不作为公开 BSP 构建仓库的组成部分。
- 正常日志与启动告警清理分别见[日志配置](kernel-debug-branches.md)和[告警记录](boot-warning-cleanup.md)。

<a id="deploy-7-2-9-20261010"></a>

### 2026-10-10 — 7.2.9 迁移与重启验收

- 基于 vanilla 7.2.9 迁移板级适配；该版本已包含 TCON TOP 路由修复，不再重复回移。[源:源码]
- Image、DTB 与对应版本模块完成构建和部署，重启后 NV3052C、DE33、Panfrost、Cedrus 及音频驱动正常初始化，PCF8563 为 rtc0，hctosys=1。[源:实测]
- 只读检查路由为 `0x0651001c=0x20`，没有临时修正模块。正常测试图经视觉验收，无左右边缘回绕；未重复全部外设功能测试。[源:实测]

<a id="lcd-route-fix-20261003"></a>

### 2026-10-03 / 04 — LCD 边缘回绕根因与持久化修复

原厂 BSP 使用 `mixer0→TCON0、mixer1→TCON2`；修复前主线两个 mixer 同时选择 TCON0。
在相同图案、时序、时钟和 framebuffer 下，只把 `TCON_TOP_PORT_SEL` 从 `0x00` 改为 `0x20`，
回绕消失；往返切换可重复复现。这确认了路由冲突是本次问题的根因。
HBP / DCLK 相位及面板寄存器的早期推断没有构成该问题的最终修复。[源:实测]

上游修复按可用 TCON 端点分配不同路由，避免板级硬编码。
修复先回移到 7.2.3，替换对应正式模块并通过重启、寄存器只读检查和视觉验收；
之后升级到自带该修复的 7.2.9。[源:实测+源码]

测试图边缘为最左红、最右白，内侧左绿右蓝；图像缓冲回读一致。
后续升级需核对原生修复仍存在，并在重启后重复边缘回归，不能只检查 framebuffer 内容。
芯片内部如何形成一像素错位仍缺少总线测量证据。[源:实测]

上游依据：[补丁讨论](https://lists.openwall.net/linux-kernel/2026/08/03/1946)、
[主线源码](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/gpu/drm/sun4i/sun8i_tcon_top.c?h=v7.3-rc1)。

以下为早期实验历史，保留各日期的观察范围，不将其当作当前待办或正式驱动能力。

### 2026-06-30 #14 — 原厂 BSP 寄存器对照（历史推断已更正）

> **2026-10-04 勘误**：原厂无回绕的观察仍有效。下文把参数差异直接认定为根因的推断已被 2026-10-03 路由对照更正；`lcd_hbp` 包含同步宽度，且 `0xe0000000` 的 bit 28 为 0，并非 `rgb_endian=1`。不要按本节旧方案继续补偿像素。 最新结论见 [路由修复记录](#lcd-route-fix-20261003)。

**目的**:启动设备原厂镜像(kernel 4.9.170),确认 BSP 屏幕是否有 1px 问题,并读取运行时 TCON 寄存器与 mainline 对比。

#### 结论 [源:实测]

**原厂 BSP 屏幕正常,无 1px 问题**。三重视觉确认:
1. dmenu launcher 正常显示,左边缘无竖条(2026-06-30 dmenu.bin 画面实测)
2. DE2 层边缘像素图案测试(simple 程序,左红 / 中黑 / 右白 + 绿蓝边界线):左右边缘均正确,无回绕(2026-07-01 实测)
3. DE2 层边缘像素图案测试(edge_v2 程序,多色边界标记):左右边缘均正确,无回绕(2026-07-01 实测)

**根因锁定**:BSP vs mainline 存在 **3 项关键差异**:

| 差异项 | BSP (4.9.170) | Mainline 7.0.9 (#11) | 差异性质 |
|--------|---------------|----------------------|----------|
| **TCON0_IO_POL** | `0x07000000` | `0x00000000` | **关键** |
| `clk_inv` (bit 26) | **1** | 0 | DCLK 驱动沿反转 |
| `sync_inv` (bits 25:24) | 3 (HSYNC_NEG\|VSYNC_NEG) | 3 | 相同 |
| `dclk_sel` (bits 30:28) | 0 | 0 | 相同 |
| **TCON0_BASIC1 hbp** | **48** | 56 | 少 8 像素 |
| **TCON0_IO_TRI rgb_endian** (bit 28) | **1** (big-endian) | 0 (little-endian) | 字节序反转 |
| TCON0_BASIC2 vbp | 48 | 52 | 少 4 像素 |
| TCON0_CTL start_delay | **31** (max) | 30 | 微差 |
| TCON0_DCLK div | **12** | — | BSP 用分频 12 |

#### BSP IO_POL 分解 [源:实测]

- BSP `IO_POL = 0x07000000`:
  - `data_inv = 0x000000`(24bit 数据极性,全正常)
  - `sync_inv = 3`(HSYNC 反相 + VSYNC 反相 → 两个都 active-low)
  - `clk_inv = 1`(DCLK 下降沿驱动)
  - `de_inv = 0`(DE 高有效)
  - `dclk_sel = 0`(3 位 DCLK 相位选择)
  - `io_output_sel = 0`

- 根据 BSP 源码 `de_lcd.c:921-946`,`clk_inv=1, dclk_sel=0` → `lcd_hv_clk_phase = 2`。与 BSP dtb `lcd_hv_clk_phase = 2` **一致**。

#### 与 #13 测试结果的再分析 [源:实测+推断]

#13 测了 4 种 IO_POL 组合:
- Phase 0 (`0x00000000`, clk_inv=0, dclk_sel=0): 回绕
- Phase 1 (`0x20000000`, clk_inv=0, dclk_sel=2): 回绕消失但右移 1px
- Phase 2 (`0x04000000`, clk_inv=1, dclk_sel=0): 回绕(同 Phase 0)
- Phase 3 (`0x24000000`, clk_inv=1, dclk_sel=2): 不稳定

**BSP 跑的是 Phase 2**（`clk_inv=1, dclk_sel=0 → IO_POL=0x04000000`,加上 sync_inv 后 = `0x07000000`)。

但 #13 测 Phase 2 (`0x04000000`) 时屏幕上**仍有回绕**——而 BSP 无回绕。这说明 **BSP 还有额外的补偿机制**,不仅仅是 IO_POL。

#### 差异分析:为什么 BSP Phase 2 无回绕而我们 Phase 2 有? [源:推断]

BSP 相较于 mainline 多了 3 个差异:

1. **`clk_inv=1`** (DCLK 下降沿采样):与我们 #13 Phase 2 测试相同 → 单独不足以消除回绕

2. **`rgb_endian=1`** (`IO_TRI bit 28`):**改变 RGB 数据线上 R/G/B 通道的字节序**。在 BSP 寄存器布局中,`tcon0_io_tri_reg_t.bits.rgb_endian` 控制输出数据的端序。**这可能是关键旋钮**:改变字节序意味着 DE 在数据线上的锁存时刻对应的像素位置发生微妙偏移,可能恰好补偿了 1 DCLK 的时序偏差

3. **HBP 不同**:BSP HBP=48,mainline HBP=56(BSP dtb 原始值 HBP=48)。**HBP 差 8 像素可能意味着 DE 信号的起始时间不同**,影响 panel 内部对"第一个有效像素"的判读

#### 修正方案 [源:实测+推断]

~~放弃 #13 的 dclk_sel=2 + 补偿路线~~,直接用 BSP 的方案:

2026-07-01 实测验证:在 mainline 上通过 `/dev/mem` 写入 BSP 的 IO_POL=0x07000000 + IO_TRI=0xe0000000 后,**1px 回绕问题仍在**(左边缘仍为白色)。BASIC1 寄存器 bus error 无法实时写入。

**关键发现**:mainline `sun4i_tcon0_mode_set_rgb()` 计算 `bp = htotal - hsync_start = 720-664 = 56`(含 hsync),写入 BASIC1 的 hbp_reg=55;**BSP 写的是真实后沿 `hbp_reg = 47`(即 hbp=48,不含 hsync)**。差异 = 8px = hsync 宽度。

**两处修改点**:

1. **`sun4i_tcon0_mode_set_rgb()`(`sun4i_tcon.c:539`)**:
   `bp = mode->crtc_htotal - mode->crtc_hsync_start`(当前:56)
   改为 `bp = mode->crtc_htotal - mode->crtc_hsync_end`(BSP:48)
   **但这会影响所有使用该函数的 LCD** — 需加 quirks 判断仅对 H616/R40 生效

2. **或改 panel 时序**(`panel-newvision-nv3052c.c:648-650`):
   把 `hsync_start/hsync_end` 各减 8,让 hbp=48 正确传出。但这违反 DRM 规范(htotal=hsync_end+hbp)。

**推荐方案**:修改 panel 侧 `fog_fj035fhd05_v1_mode`:
```c
.hsync_start = 640 + 16,   /* hfp = 16 (曾为 24) */
.hsync_end   = 640 + 16 + 8, /* hsync = 8 */
.htotal      = 640 + 16 + 8 + 56,  /* 720, bp自然成为56 */
```
或通过 bus_flags 引入新标志让驱动使用正确的 HBP 计算。

**下一步**:直接修改 panel 模式时序或 sun4i_tcon.c 的 H616 quirks,rebuild 内核测试。

更新章节:§0(状态)、§99(本条)、§99 #15

### 2026-07-01 #16 — 纯色画面与归档状态（历史记录）

> **2026-10-04 勘误**：本节固定阴影是另一个历史观察，不能用来解释此次已修复的边缘回绕；其硬件成因未由本轮实验核验。后文 master/产物/HBP 状态均属于当时版本，当前以本页顶部及最新记录为准。 最新结论见 [路由修复记录](#lcd-route-fix-20261003)。

**目的**:区分剩余的边缘彩线问题与用户观察到的“中间灰、四周黑”拖影/阴影。

#### 结果 [源:实测]

- 在纯色灰色 framebuffer 下，阴影仍出现在与测试图案相同的物理位置。
- 更换边缘彩色图案、光标/TTY 内容后，阴影的环形轮廓和中心位置不随图案移动。
- 边缘彩线（左侧白/绿、右侧红）与该环形阴影可以同时出现，二者是可分离现象。
- 逐步改变 TCON `IO_POL`/`IO_TRI` 会改变边缘采样颜色或造成渐暗，但不能稳定消除环形阴影；正式 modeset 后彩色标记可消失而阴影仍在。

#### 结论 [源:实测+推断]

环形“中心较亮/较灰、四周较暗/黑”的固定纹理不是 framebuffer 内容、DE33 缩放/CSC、NV3052C 地址窗口或 TCON 水平时序造成的图案残留。它更符合 LCD 模组的光学/机械不均匀（导光板、扩散片、面板受压或 mura）。继续修改内核显示时序预计不能消除该现象；后续应转向拆机检查、均匀背光/面板压力和模组替换对照。

#### Git 归档

- `master` (`ad6c36d74`): 仅保留已验证的 H616 LCD 实时后沿（HBP=48）修复。
- `lcd-experiments` (`ddfb22b5c`): 保留 NV3052C 地址窗口、DE33 scaler/CSC、寄存器转储等实验提交，作为可复现实验依据，不作为默认修复。

### 2026-09-04 #17 — Git 整理与最终证据分级

本轮实验后的归档规则如下：

| 内容 | 结论 | 归档位置 |
|---|---|---|
| H616/R40 LCD 后沿计算使用真实 HBP（排除 HSYNC） | 已通过设备 dmesg 与画面验证，稳定有效 | `master`，`ad6c36d74` |
| NV3052C CASET/RASET 显式窗口 | 现场验证无效 | `lcd-experiments`，`ab010f14d` |
| DE33 scaler/CSC、alpha、格式等尝试 | 未证明与竖线成因相关 | `lcd-experiments`，`78fda029c` |
| H616 TCON/DE 寄存器转储工具 | 仅调试用途 | `lcd-experiments`，`ddfb22b5c` |
| `IO_POL`/`IO_TRI` 热写组合 | 会改变颜色/采样或造成渐暗，但没有稳定零偏修复 | 仅保留实验记录，不进主线 |
| DCLK `dclk_sel=2` | 可消除回绕但整体右移 1px；未形成完整修复 | 仅保留实验记录，不进主线 |
| 纯色画面固定环形阴影 | 与图案无关，判定为 LCD 模组光学/机械不均匀 | 硬件问题，非内核修复项 |

因此，默认构建应使用 `master`；需要复现实验或继续采集寄存器时再切换到 `lcd-experiments`。工作区仅剩构建生成的 VDSO 二进制改动，未纳入任何提交。

### 2026-06-27 #13 — DCLK 相位组合实验（历史观察，非最终根因）

> **2026-10-04 勘误**：改变相位曾改变可见现象，但未获得边缘像素完整的修复。“真正旋钮”和“已找到一半根因”的推断不再作为结论；此次正式修复只分离 mixer 路由，没有做 DCLK 相位或像素补偿。 最新结论见 [路由修复记录](#lcd-route-fix-20261003)。

**目的**:#12 结论称「DCLK_PHASE=1 无效 / =2 黑屏」,但该测试基于 mainline `SUN4I_TCON0_IO_POL_DCLK_PHASE(phase) = (phase & 3) << 28` 宏(仅 2 位 bits 29:28)。本次重新审视 BSP `tcon0_io_pol` 位定义,发现 **BSP `dclk_sel` 是 3 位(bits 30:28),与 mainline 2 位定义不同**,且 BSP 用 `clk_inv`(bit 26)+ `dclk_sel`(bits 30:28)双字段组合表达 4 个相位。重测全部 4 组合。

#### §1 BSP vs mainline 位定义差异 [源:实测+厂商]

**BSP** `tcon0_io_pol_reg_t`(`linux-allwinner-4.9/.../de_lcd_type.h:222-232`):

| bit | 字段 | 宽度 |
|---|---|---|
| 26 | `clk_inv` | 1 |
| 27 | `de_inv` | 1 |
| 28-30 | `dclk_sel` | **3** |
| 31 | `io_output_sel` | 1 |

**Mainline** `SUN4I_TCON0_IO_POL_*`(`linux-7.0.9/.../sun4i_tcon.h:115-117`):

| bit | 字段 | 宽度 |
|---|---|---|
| 26 | `DCLK_DRIVE_NEGEDGE`(≡ BSP `clk_inv`) | 1 |
| 27 | `DE_NEGATIVE`(≡ BSP `de_inv`) | 1 |
| 28-29 | `DCLK_PHASE` | **2**(丢弃 bit 30) |

**关键差异**:BSP `dclk_sel` 占 3 位,mainline `DCLK_PHASE` 只占 2 位。BSP phase 1/3 的 `dclk_sel=2`(`010`)写入 bits 29:28 = `10`,正好落在 mainline 2 位范围内;但语义上 BSP 是「3 位选择器」,mainline 是「2 位相位」。

BSP `lcd_hv_clk_phase` → 寄存器映射(`de_lcd.c:489-509`)[源:厂商]:

| `lcd_hv_clk_phase` | `clk_inv` | `dclk_sel` | IO_POL 值 |
|---|---|---|---|
| 0 | 0 | 0 | 0x00000000 |
| 1 | 0 | 2 | 0x20000000 |
| 2 | 1 | 0 | 0x04000000 |
| 3 | 1 | 2 | 0x24000000 |

#### §2 测试方法 [源:实测]

1. 设备:`ssh root@sp`,内核 7.0.9
2. 测试图案(写 `/dev/fb0`,640×480 XRGB8888):x=0 红、x=1 绿、x=638 蓝、x=639 白,中间 x=320 红竖线 + y=240 绿横线
3. 逐个写 `TCON0_IO_POL`(0x06511088)为 4 个相位值,每个停留观察左/右边缘颜色
4. 判据:左边缘应为红(正常)/ 白(x=639 回绕)/ 绿(x=0 丢失,右移);右边缘应为白(正常)/ 蓝(x=638 回绕后露)/ 黑(x=639 丢失)

#### §3 测试结果 [源:实测]

| Phase | clk_inv | dclk_sel | IO_POL | 屏幕状态 | 左边缘 | 右边缘 | 现象解读 |
|---|---|---|---|---|---|---|---|
| 0 | 0 | 0 | 0x00000000 | 亮 | 白 | 蓝 | 回绕(x=639→x=0, x=638→x=639) |
| 1 | 0 | 2 | 0x20000000 | 亮 | 绿 | 黑 | **回绕消失**,但整体右移1px(x=0红丢失→看到x=1绿, x=639白丢失→右边黑) |
| 2 | 1 | 0 | 0x04000000 | 亮 | 白 | 蓝 | 回绕(同 phase 0,clk_inv 无影响) |
| 3 | 1 | 2 | 0x24000000 | **不稳定** | 紫(渐暗) | 黑 | 红+蓝混色,亮度逐渐下降,不可用 |

#### §4 关键结论 [源:实测+推断]

1. **`dclk_sel`(bits 30:28)才是 1px 回绕的真正旋钮,`clk_inv`(bit 26)无效**:
   - `dclk_sel=0`(phase 0/2):回绕存在
   - `dclk_sel=2`(phase 1/3):回绕消失
   - `clk_inv` 翻转(phase 0→2, 1→3)对回绕无影响,仅 phase 3 出现不稳定(可能是 `clk_inv=1`+`dclk_sel=2` 组合的硬件边界)

2. **#12 「DCLK_PHASE=2 黑屏」系误判**:#12 用 mainline `DCLK_PHASE(2) = (2&3)<<28 = 0x20000000` 测试,记录为「黑屏」。本次实测 `0x20000000` 是**亮屏且回绕消失**(phase 1)。推测 #12 当时可能误写为 `0x24000000`(phase 3,本次实测不稳定/渐暗)或漏写 `clk_inv` 位。**#12 该条结论作废,以本条为准**。

3. **Phase 1(0x20000000)是可行解的一半**:消除回绕,但引入 1px 右移(x=0 内容丢失,右边露出 1px 黑)。这与 #12 「水平定位由 DE/DCLK 相位决定」的判断一致——`dclk_sel=2` 让 DCLK 延迟半个周期,数据采样点右移 1 像素,回绕消失但整体右移。

4. **根因机制(推断)**:`dclk_sel=0` 时 DCLK 上升沿采样数据,但 DE/DATA 建立时间不足,第 1 个像素被采样到上一行的末尾(回绕);`dclk_sel=2` 延迟采样,建立时间足够,但延迟过量导致丢失 1px。理想情况应是 `dclk_sel=1`(延迟 1 个 DCLK),但 BSP `lcd_hv_clk_phase` 只定义了 0 和 2 两档,未定义 1 [源:厂商+推断]。

#### §5 下一步 [源:推断]

Phase 1 消除回绕但右移 1px,需补偿这 1px:
1. **DE33 LAYER_COORD x=-1**:layer 输出左移 1px,抵消 DCLK 右移。风险:负坐标可能不被硬件支持
2. **TCON0 BASIC1 HBP+1**:增加后沿 1px,DE 窗口左移 1px。风险:可能重新引入回绕
3. **BSP 对比**:查 BSP 实际 `lcd_hv_clk_phase` 值,确认 vendor 是否用 phase 1 + 某种补偿

**优先级**:先查 BSP `lcd_hv_clk_phase` 实际值(若 vendor 也用 phase 1,必有配套补偿),再决定补偿方向。

### 2026-06-24 #12 — NV3052C V0.2 数据手册分析（历史推断已更正）

> **2026-10-04 勘误**：此处曾将偏重 MIPI 的 V0.2 手册推论外推到 RGB 模式，“没有任何寄存器”“唯一修复方向”等绝对断言缺乏充分证据。后续 NV3052CGRB 手册与正确 9-bit SPI 测试见最新记录；本次已证实根因为 TCON TOP 路由，不采用下面的软件移位 workaround。 最新结论见 [路由修复记录](#lcd-route-fix-20261003)。

**目的**:#11 收尾时三个「下一步」之一是「获取 NV3052C datasheet,查找源输出时序相关的未文档化寄存器」。本条通读 `doc/NV3052C-Datasheet-V0.2.pdf`(214 页,原厂 2018-10 文档)回答这个问题。

**核心结论:NV3052C 没有任何「水平方向(像素/PCLK 单位)」的位置寄存器。1px 水平左移回绕在芯片侧无寄存器可调** [源:厂商]。

证据(逐条对照 datasheet):

| datasheet 事实 | 页/行 | 对 1px 问题的含义 |
|---|---|---|
| 所有 GIP shift 寄存器(`gip_vst*_shift`、`gip_vend*_shift`、`gip_clka/b/c*_shift`)单位**全是 `line`(行 / 垂直)** | §5.3.15-5.3.20(00H-71H) | 这些只能在**垂直**方向移信号,改不了水平。#11 试的 Page3[0x00]=`gip_vst` 类寄存器属此类 → 注定对水平无效 |
| `SS` bit(MADCTL 36H / page 0xE0 区)只做 source **整屏左右翻转**(`SS=0` 左→右,`SS=1` 右→左) | line 1469-1470 / 2143-2146 | 只能整屏镜像,不能平移 1px |
| 640RGB 分辨率下 source 映射**固定**为 `S[2400:1441], S[960:1]`,由分辨率档位决定,无逐像素起点寄存器 | line 516 | 活动区在 source driver 上的落点写死,软件不可微调 |
| RGB/HV 并口模式下,芯片**「单行数据累计满后整行 latch」**,数据水平落点完全由输入流(DE / HSYNC + HBP)决定,芯片不自带水平相位寄存器 | line 341-342, §3.1.6 | **水平定位的「方向盘」在 TCON 侧,不在 panel 侧** |
| datasheet 中唯一的 HBP/HFP/HSYNC 数字(HBP typ 42 / HFP typ 44 / htotal 778-808 @720×1280)只出现在 **MIPI video 模式时序参考表**,不是可写寄存器 | line 6680-6727 | 这些是「输入应满足的时序范围」,不是 panel 内部可调寄存器 |

**这解释了 #11 为什么 11 种 NV3052C 寄存器改动全部无效**:Page1/Page2/Page3 改的要么是 GIP 垂直时序、要么是 gamma/source 幅度,**没有一个能影响活动区的水平像素位置**——因为这种寄存器在 NV3052C 上根本不存在。继续在 panel SPI 寄存器里找水平修正寄存器是死路 [源:厂商+推断]。

**问题真正的归属**(重定位):水平 1px 落点由 **RGB 输入流的 DE/DCLK/HBP 相位**决定,即 **TCON 侧 + DE 对齐**,不是 panel 内部 [源:厂商+推断]。佐证:#11 实测 TCON 改 HBP 无效 → 说明 panel 工作在 **DE 跟随模式**(活动区锁 DE,不按 HSYNC 数 HBP),所以 HBP 改了没反应是自洽的;此时水平位置由「DE 边沿 vs 数据」的相对相位决定,而 DCLK 相位是唯一沾边的旋钮。

> **[勘误 2026-06-27 #13]**:本条原称「#11 试 DCLK_PHASE=1 无效 / =2 黑屏」,经 #13 重新实测 4 组合后**作废**:`0x20000000`(mainline `DCLK_PHASE(2)`,即 BSP phase 1)实际是**亮屏且回绕消失**,仅引入 1px 右移。原「黑屏」判断系位域混淆误判。详见 §99 #13。DCLK 相位确为旋钮,且 `dclk_sel=2` 是可行解的一半(需补 1px 右移)。

**对「反汇编 vendor BSP 抠 SPI 序列」这条路的影响**:**降级,基本可放弃** [源:推断]。datasheet 已证明 vendor 也无法用 SPI 寄存器做水平 1px 修正——vendor 若画面正确,差异只会在 **TCON 时序 / DE 生成 / DCLK 边沿** 上,不会在 SPI init 序列里。所以即便抠出 vendor SPI 序列,对这个 1px 问题也大概率无帮助(对其它问题如花屏/色差仍有价值,但本问题无关)。

**剩余可行路径**(按性价比):
1. **软件 workaround(推荐,立即可做)**:framebuffer 输出整体右移 1px 补偿。datasheet 已证根因在芯片侧无解、TCON 侧也只剩 DCLK 边沿一个旋钮且已试过,此 workaround **不是「掩盖可修的 bug」,而是补偿一个芯片本身没有调节能力的固有相位** → 工程上正当。代价:右边缘丢 1px 或显示 1px 黑边。

   > **[勘误 2026-06-27 #13]**:本路径的前提「DCLK 边沿旋钮已试过」当时基于 #11 的误判(见上勘误)。#13 实测 `dclk_sel=2` 可消除回绕但引入 1px 右移,因此**根因修复已找到一半**,软件 workaround 可降级为备选——优先走 #13 §5 的「Phase 1 + 1px 补偿」方向。

2. **TCON DE/DCLK 相位深挖**:确认 panel 实际是 DE 模式还是 HV sync 模式(datasheet line 479-489 RGB 接口引脚:HSYNC/VSYNC/DCLK/DE 都在;BSP `lcd_hv_if=0` = HV sync,但 mainline sun4i TCON 默认同时输出 DE)。若能让 panel 走纯 HV sync 模式并精确给 HBP,或反过来调 DE 与数据的 setup,可能根治。这是唯一的「根因修复」方向。

   > **[更新 2026-06-27 #13]**:此路径已部分兑现——#13 实测确认 `dclk_sel`(bits 30:28)是有效旋钮,`dclk_sel=2` 消除回绕。剩余问题转为「如何补偿 Phase 1 引入的 1px 右移」,见 §99 #13 §5。

3. **逻辑分析仪对比 vendor**:抓 vendor 的 **DE/DCLK/HSYNC vs RGB data 相位关系**(不是 SPI!),与 mainline 对比 1 个 DCLK 的差异。比抠 SPI 序列更对症。

   > **[更新 2026-06-27 #13]**:此路径仍有效——#13 推断 `dclk_sel=1`(延迟 1 个 DCLK)可能是理想值,但 BSP `lcd_hv_clk_phase` 未定义该档。逻辑分析仪可验证 vendor 实际相位,或直接试写 `dclk_sel=1`(0x10000000)观察。

**结论**:#11 标「⚠️ 未做 / 需 datasheet」的项可结案——datasheet 看过了,答案是**「此问题芯片侧无寄存器,不要再在 NV3052C SPI 上花时间」**。建议优先上软件 workaround(路径 1),根因修复留给 TCON DE/DCLK 相位(路径 2)。

> **[勘误 2026-06-27 #13]**:本结论中「根因修复留给路径 2」的判断正确,但当时误以为 DCLK 相位已试完。#13 实测证明 DCLK 相位(`dclk_sel`)是有效旋钮,根因修复已有具体方向(Phase 1 + 1px 补偿)。建议优先走 #13 §5,软件 workaround 降级为备选。

更新章节:§0(下一步)、§99(本条)

### 2026-06-23 #11 — 帧缓冲读写一致性验证 + 左边缘竖条排查

> **2026-10-04 勘误**：内存读写一致与层参数正确不能证明整个显示通路正确；当时遗漏了 TCON TOP 的共享路由冲突。面板内部根因和“全链路验证通过”属于已更正推断；仅凭几条边线不应断言全行左移。旧 panel_reg/panel_fix 工具缺少 9-bit D/C 帧，旧 SPI 阴性结果不能排除对应寄存器的作用。 最新结论见 [路由修复记录](#lcd-route-fix-20261003)。

**目的**:验证 framebuffer 内存读写是否一致;排查屏幕左边缘竖条(右侧像素复制到左侧)的根因。

#### 1. 帧缓冲色彩一致性测试(帧缓冲读写测试)

**工具**:新建 帧缓冲读写测试工具 — 本地生成 18 种测试图案(纯色/棋盘/渐变/十字/像素标记)为 raw XRGB8888,通过 DRM ioctl(GETFB + MAP_DUMB) + mmap 写入设备 framebuffer,读回后本地 numpy 逐像素对比。

**结果**([源:实测] `ssh root@DEVICE_HOST` + numpy 分析):

| 测试项 | fb[52] (menu.bin) | fb[50] ([fbcon]) |
|---|---|---|
| 18 种图案 × 2 层 = 36 测试 | 18/18 PASS,差异像素=0 | 18/18 PASS,差异像素=0 |

- 首次运行 fb[50] 的 blue 图案有 128 像素差异(y=96-97,被清零) — **根因:fbcon 光标闪烁**([源:推断] 内核 fbcon 驱动在 console 层上绘制/擦除光标)
- **修复**:测试前写 `0 > /sys/class/graphics/fbcon/cursor_blink` 抑制光标,测试后恢复。修复后 36/36 全部 PASS
- **结论**:**framebuffer 内存读写完全一致**,写入的数据和读回的数据逐字节相同。内存层面无问题。

#### 2. 设备运行内核 vs 本地构建内核对比

**目的**:确认设备实际运行的内核是否包含 git diff 中的未提交修复(DE33 CSC / scaler_mask / alpha 等)。

**结果**([源:实测] `strings` + 文件大小对比):

| 位置 | 文件大小 | 日期 | 含 DE33 修复? |
|---|---|---|---|
| 设备 `/lib/modules/7.0.9/kernel/drivers/gpu/drm/sun4i/sun8i-mixer.ko` | 277,824 B | 2026-06-07 08:45 | ✅ 含(`sun8i_de33_ccsc_setup` / `yuv2rgb_de33` / `scaler_mask` 等字符串) |
| 构建端 sun8i-mixer.ko | 58,592 B | 2026-06-06 15:29 | ❌ 不含 |

- **结论**:设备跑的**已经是修复后的版本**。git diff 中的 5 个文件(sun50i_planes / sun8i_csc / sun8i_mixer / sun8i_ui_layer / sun8i_vi_layer)的改动已编译部署到设备。

#### 3. TCON 寄存器实读

**目的**:验证 TCON 时序寄存器是否与 dmesg / 驱动代码一致。

**方式**:通过 `/dev/mem` mmap 直读 `0x06511000`(tcon_lcd0 物理基址),python3 解析([源:实测])。

**结果**([源:实测] `devmem2` + python mmap):

| 寄存器 | 偏移 | 实读值 | 解码 | 与 dmesg 一致? |
|---|---|---|---|---|
| TCON0_CTL | 0x40 | `0x800001e0` | TCON_EN=1, CLK_DELAY=30 | ✅ dmesg: "TCON 0 clock delay 30" |
| TCON0_BASIC0 | 0x48 | `0x027f01df` | hdisplay=640, vdisplay=480 | ✅ |
| TCON0_BASIC1 | 0x4c | `0x02cf0037` | htotal=720, hbp+hsync=56(HBP=48, HFP=24) | ✅ dmesg: "htotal 720, backporch 56" |
| TCON0_BASIC2 | 0x50 | `0x04600033` | vtotal=1120(÷2=560), vbp+vsync=52(VBP=48) | ✅ dmesg: "vtotal 560, backporch 52" |
| TCON0_BASIC3 | 0x54 | `0x00070003` | hsync=8, vsync=4 | ✅ dmesg: "HSYNC 8, VSYNC 4" |
| TCON0_IO_POL | **0x88**（原记录误写 0x58） | 旧值 `0x00000000` 需按正确地址重新核对 | 不能从错误地址验证极性 | 2026-10-03 正确地址实测 `0x04000000` |

- **时序完全匹配** `fog_fj035fhd05_v1_mode` 定义([源:实测] 驱动源码 `panel-newvision-nv3052c.c:630-642`)和 BSP dtb 数据([源:厂商] display-worklog §8.2)
- **CLK_DELAY=30** = min(vtotal-vdisplay, 30) = min(80, 30) = 30([源:实测] `sun4i_tcon.c:276-292`)

#### 4. DE33 层寄存器 dump(drm_regdump.ko)

**目的**:dump DE33 显示引擎层寄存器,确认 framebuffer 配置是否正确写入硬件。

**工具**:新建 寄存器转储测试模块 — 内核模块,创建 `/proc/display-regdump`,dump TCON0 + planes 区域寄存器。

**关键发现 1:层寄存器在 planes 区域,不在 mixer**([源:实测]):

首次尝试读 mixer 区域(0x01280000)的层寄存器,**全部为零**。通过扫描发现 framebuffer DMA 地址(0x7bf00000)和 pitch(2560)实际位于 **planes 区域**(0x01100000)偏移 0xc100c-0xc1010。

- **根因**([源:实测] 驱动源码 `sun50i_planes.c:65-69`):DE33 的层初始化传入的是 `planes->regs`(来自 DTS `planes@100000` 节点,物理地址 0x01100000),不是 `mixer->engine.regs`(来自 `mixer@280000`,物理地址 0x01280000)
- **验证**([源:实测] `python3` + `/dev/mem` mmap 扫描):在 planes 区域 0x01100000 偏移 0xc100c 找到 pitch=2560,0xc1010 找到 DMA addr=0x7bf00000

**关键发现 2:DE33 planes channel 偏移**([源:实测]):

| Channel | Planes 区域偏移 | 物理地址 | 用途 |
|---|---|---|---|
| ch0 | 0x40000 | 0x01140000 | VI layer (plane-0, inactive) |
| ch1 | 0xc1000 | 0x011c1000 | UI layer (plane-1, active, menu.bin) |

- 注意:ch1 偏移 0xc1000 ≠ `DE33_CH_BASE(0x1000) + ch * DE33_CH_SIZE(0x20000)` = 0x21000。实际 planes 区域的 channel 布局与 mixer 区域不同([源:实测] 扫描确认)
- DTS 定义([源:实测] `sun50i-h616.dtsi`):`planes@100000 { reg = <0x100000 0x180000>; }` — 1.5MB 区域

**关键发现 3:Channel 1 层寄存器完整 dump**([源:实测] `drm_regdump.ko` + `/proc/display-regdump`):

| 寄存器 | 偏移 | 实读值 | 解码 | 与 DRM state 一致? |
|---|---|---|---|---|
| LAYER_ATTR | +0x00 | `0x00000401` | EN=1, FBFMT=4(XRGB8888), ALPHA=0x00 | ✅ |
| LAYER_SIZE | +0x04 | `0x01df027f` | h=480, w=640 | ✅ |
| LAYER_COORD | +0x08 | `0x00000000` | x=0, y=0 | ✅ |
| LAYER_PITCH | +0x0c | `0x00000a00` | 2560 bytes/line | ✅ |
| LAYER_TOP_LADDR | +0x10 | `0x7bf00000` | DMA addr (menu.bin fb52) | ✅ |
| OVL_SIZE | +0x88 | `0x01df027f` | h=480, w=640 | ✅ |

- **结论**:**层寄存器配置完全正确**,与 DRM debugfs state 一致。framebuffer 地址、分辨率、像素格式、pitch 全部正确写入硬件。

**Mixer 区域(0x01280000)层寄存器**([源:实测]):

Mixer 区域的层寄存器(0x1000-0x1c00 范围)有非零值,但**不含 framebuffer 参数**(无 DMA addr、无 pitch)。这些是 mixer 内部的路由/混合配置,不是层数据配置。

- 非零模式:每 0x400 一组重复(4 个 channel),每组结构相同 → mixer 的内部 pipeline 配置,非层数据

#### 5. 左边缘 1px 水平偏移问题

**现象**:屏幕左边缘有 1 像素宽的异常竖条。棋盘格图案时肉眼可见。

**关键判断**:当前 `帧缓冲读写测试` **无法抓取此问题**。原因:

```
Framebuffer 内存 → Plane → DE33 Mixer → TCON → Panel
  (§1 验证了这里)  (§4 验证了这里)  (§3 验证)  (问题在这里)
```

**精确现象描述**([源:实测] 2026-06-23):

**不是"复制",是"左移回绕"**:x=639 的像素移动到 x=0 位置,所有其他像素整体左移 1 像素。

验证方法:
1. 写入测试图案:每行 x=0=红(255,0,0),x=639=绿(0,255,0),其余黑色
2. 截图确认 framebuffer 内容正确([源:实测] 截图工具 + numpy 分析)
3. 拍屏结果:**x=0 显示黄色(红+绿叠加),x=639 显示黑色**
4. 垂直方向测试(上红下绿):**上下边缘完全正确,无偏移**

结论:问题仅影响水平方向,是源输出时序导致的 **1px 左移 + 回绕**。

**已排除的修复尝试**([源:实测] 2026-06-23):

| 尝试 | 方法 | 结果 |
|---|---|---|
| DCLK 相位 | TCON0_IO_POL DCLK_PHASE 设为 2 | ❌ 屏幕全黑(面板不支持) |
| CASET/RASET | 面板 enable 函数加显式列/行地址设置 | ❌ 无效果 |
| TCON HBP | BASIC1 寄存器 HBP 从 55 改为 47(BSP 值) | ❌ 无效果 |
| Page2[0x01] | NV3052C 源输出时序寄存器 0x01→0x00 | ❌ 无效果 |
| Page1[0x03] | NV3052C 寄存器 0x03→0x41/0x3f | ❌ 无效果 |

> **[勘误 2026-06-27 #13]**:上表「DCLK 相位 设为 2 → 黑屏」一条**作废**。#13 重新实测 4 组合发现 `0x20000000`(即 mainline `DCLK_PHASE(2)`,BSP phase 1)实际**亮屏且回绕消失**,仅引入 1px 右移。原「黑屏」判断系位域混淆误判(可能误写为 `0x24000000` phase 3,该值本次实测不稳定/渐暗)。详见 §99 #13。**DCLK 相位非但不是「面板不支持」,反而是 1px 回绕的真正旋钮**。

| Page3[0x00] | GIP 垂直起始寄存器 0x2a→0x2b | ❌ 无效果 |

**排查状态**:

| 排查项 | 状态 | 结论 | 来源 |
|---|---|---|---|
| 内存读写一致性 | ✅ 已验证 | 36/36 PASS | [源:实测] §1 |
| 驱动修复部署 | ✅ 已验证 | DE33 修复已在设备运行 | [源:实测] §2 |
| TCON 时序寄存器 | ✅ 已验证 | 全部正确,与 BSP 一致(含 DCLK_NEG=1) | [源:实测] §3 |
| DE33 层寄存器 | ✅ 已验证 | 配置完全正确(ADDR/PITCH/FORMAT/SIZE) | [源:实测] §4 |
| **水平 1px 左移回绕** | **✅ 已确认** | **仅水平方向,x=639 移到 x=0,垂直无问题** | **[源:实测] 拍屏验证** |
| TCON HBP 调整 | ❌ 无效 | HBP 55→47 无变化 | [源:实测] |
| DCLK 相位调整 | ❌ 无效/黑屏 | DCLK_PHASE=2 导致黑屏 | [源:实测] |
| NV3052C CASET/RASET | ❌ 无效 | 显式设置列地址无效果 | [源:实测] |
| NV3052C 源输出时序 | ❌ 无效 | Page1/Page2 多个寄存器调整无效果 | [源:实测] |
| NV3052C GIP 起始 | ❌ 无效 | Page3[0x00] 调整无效果 | [源:实测] |
| vendor BSP init 序列对比 | ⚠️ 未做 | 需反汇编 p4 boot.img 提取 SPI 序列 | — |

**结论**([源:实测+推断]):

内存→DE33→TCON 全链路验证通过(§1-4)。水平 1px 左移回绕已通过已知图案拍屏确认([源:实测])。垂直方向无问题。问题锁定在 **NV3052C 面板内部源输出时序**。

最可能的原因([源:推断]):
1. **`wl_355608_a8_panel_regs[]` 与 SP 面板不完全匹配**:GIP 信号映射(page 3, 寄存器 0xb1-0xd0)或源输出时序(page 1/2)与 SP 面板的玻璃基板走线不完全匹配
2. **需对比 vendor BSP init 序列**:反汇编 p4 boot.img 中 `lcd_panel_fog_fj035fhd05_v1_init` 函数,提取 SPI 命令序列,逐条比对

**已排除的寄存器修改**([源:实测] 2026-06-24):

| 寄存器 | 原值 | 试值 | 结果 |
|---|---|---|---|
| TCON DCLK_PHASE | 0 | 1, 2 | 1=无效, 2=黑屏 |
| TCON HBP | 55 | 47 | 无效 |
| CASET/RASET | 默认 | 0-639, 0-479 | 无效 |
| Page1[0x03] | 0x40 | 0x3f, 0x41 | 无效 |
| Page1[0x04] | 0x00 | 0x01, 0xff | 无效 |
| Page1[0x05] | 0x03 | 0x02, 0x04 | 无效 |
| Page1[0x08] | 0x00 | 0x01 | 无效 |
| Page1[0x09] | 0x07 | 0x06, 0x08 | 无效 |
| Page1[0x0a] | 0x01 | 0x00, 0x02 | 无效 |
| Page2[0x01] | 0x01 | 0x00 | 无效 |
| Page3[0x00] | 0x2a | 0x2b | 无效 |

> **[勘误 2026-06-27 #13]**:上表「TCON DCLK_PHASE 0→1,2 结果 1=无效,2=黑屏」一条**部分作废**。#13 重新实测发现:
> - `DCLK_PHASE=1`(0x10000000,即 `dclk_sel=1`)本次**未测**,需补测
> - `DCLK_PHASE=2`(0x20000000,即 `dclk_sel=2`,BSP phase 1)实际**亮屏且回绕消失**,仅引入 1px 右移,**非黑屏**
> - 原「2=黑屏」判断系位域混淆误判。详见 §99 #13。
> 本表其余 NV3052C SPI 寄存器(Page1/Page2/Page3)的「无效」结论仍然有效(#12 datasheet 已证 panel 侧无水平位置寄存器)。

**结论**:

TCON 侧 + NV3052C 已知寄存器均无法修正 1px 偏移。问题可能在:
1. NV3052C 未文档化的源输出时序寄存器(需 datasheet)
2. `wl_355608_a8_panel_regs[]` 与 SP 面板不匹配(需 vendor BSP SPI 序列对比)
3. 面板硬件本身的设计特性(可能无法通过软件修复)

> **[勘误 2026-06-27 #13]**:上述结论第 1、3 条**部分作废**。#13 实测证明 TCON 侧 `dclk_sel`(bits 30:28)是有效旋钮,`dclk_sel=2` 可消除回绕(引入 1px 右移待补偿)。NV3052C 侧无寄存器的判断(#12 datasheet 佐证)仍成立。问题非「无法通过软件修复」,而是「需 dclk_sel=2 + 1px 补偿」。

**下一步**:
1. **反汇编 vendor BSP 提取 SPI init 序列**:与 `wl_355608_a8_panel_regs[]` 逐条对比(需找到 SPI 写入函数的代码路径)
2. **获取 NV3052C datasheet**:查找源输出时序相关的未文档化寄存器
3. **逻辑分析仪**:抓 vendor BSP 的 SPI 初始化序列,与 mainline 对比
4. **软件 workaround**:在 framebuffer 写入时右移 1 像素补偿(不修复根因,但消除可见问题)

更新章节:§99(本条)

### 2026-05-22 #8 — 深入代码静态分析:component bind 链路全审计(本次研究)

**分析目的**:在设备已处于 刷卡流程 刷新状态时，通过源码审计确定为什么 `sun4i_drv_bind` 没有被调用([源:推断])。

**审查范围**:
- `drivers/gpu/drm/sun4i/sun4i_drv.c` — OF graph 遍历 + component master 注册
- `drivers/bus/sun50i-de2.c` — DE33 bus 驱动(SRAM claim → `of_platform_populate`)
- `drivers/base/component.c` — Linux component 框架核心逻辑
- `drivers/gpu/drm/sun4i/sun4i_tcon.c` — TCON 驱动兼容表
- `drivers/gpu/drm/sun4i/sun8i_mixer.c` — Mixer 驱动 probe/component_add
- `drivers/gpu/drm/sun4i/sun8i_tcon_top.c` — TCON TOP 驱动 probe/component_add
- `drivers/of/property.c` — fw_devlink supplier bindings 表(确定哪些 DT 属性触发 device link)
- `arch/arm64/boot/dts/allwinner/sun50i-h616.dtsi` — 显示管线 DT 节点
- `arch/arm64/boot/dts/allwinner/sun50i-h700-anbernic-rg35xx-2024.dts` — 板级 DT enable

**关键结论:静态分析未发现明显 bug，component 框架逻辑自洽。**

具体发现:

**1. Component 收集数量** [源:推断]:
- `allwinner,pipelines` → mixer0, mixer1 (2 个 phandle)
- mixer0 port@1 → tcon_top; mixer1 port@1 → tcon_top (重复)
- tcon_top port@1 → tcon_lcd0 (endpoint@0) + tcon_tv0 (endpoint@2)
- tcon_top port@3 → tcon_lcd0 (endpoint@0) + tcon_tv0 (endpoint@2)
- tcon_top port@5 → hdmi
- tcon_tv0、hdmi 均为 disabled → `sun4i_drv_add_endpoints` 返回 0，不加入 match
- tcon_lcd0 port@1 → panel → 被 TCON channel-0 逻辑跳过(`endpoint.id==0`)
- **最终 match->num = 8**(含重复),**唯一组件 4 个**:mixer0, mixer1, tcon_top, tcon_lcd0
- 4 个组件的 driver **全部有** `component_add` 调用(mixer → sun8i_mixer_probe, tcon_top → sun8i_tcon_top_probe, tcon_lcd0 → sun4i_tcon_probe)

**2. Component 匹配** [源:推断]:
- `component_compare_of` 用 `device_match_of_node`(指针比较 `dev->of_node == data`)
- 重复 entry(match->num=8 vs 4 个唯一设备)通过 `c->adev` 去重，组件框架正确处理
- `try_to_bring_up_aggregate_device` → `find_components` → 找到全部 4 个唯一组件 → 调用 `adev->ops->bind` (= `sun4i_drv_bind`)

**3. fw_devlink 影响** [源:实测]:
- `allwinner,pipelines` 属性**不在** `of_supplier_bindings[]` 表中(该表仅含 clocks/resets/gpios/pwms 等标准属性)
- `display-engine` 节点没有标准 supplier 属性(clocks/resets/gpios 等)
- **fw_devlink 不会基于 `allwinner,pipelines` 创建 device link**
- `waiting_for_supplier = 1` 的来源需在设备上确认(可能是 sync_state link 或父节点链路)

**4. DE33 bus / SRAM-C** [源:推断]:
- `drivers/bus/sun50i-de2.c` 匹配 `allwinner,sun50i-a64-de2`(DT fallback)
- `sunxi_sram_claim()` → 从 `allwinner,sram = <&de3_sram 1>` claim SRAM-C
- SRAM-C 走 `allwinner,sun50i-a64-sram-c` fallback(7.0.9 的 `sunxi_sram.c` 无 h616 专属条目)
- claim 成功后 `of_platform_populate` 创建子设备(mixer/planes/clock)
- 设备上 mixer/planes/clock 均已 probe → SRAM claim 成功

**5. TCON 兼容表缺口** [源:实测]:
- `sun50i-h616-tcon-tv` **不在** `sun4i_tcon_of_table` → tcon_tv0 无 driver 可绑
- 但 tcon_tv0 在 DT 中 disabled → 不影响 component 收集
- `sun50i-h616-tcon-lcd` 也不在表，但通过 `sun8i-r40-tcon-lcd` fallback 成功绑定
- `sun8i_r40_lcd_quirks.has_channel_0 = true` → panel 端点被正确跳过

**6. 新增 debug printk** [源:推断/工程]:
- `sun4i_drv_probe`:入口 + pipeline 数量 + 最终 count + match 指针
- `sun4i_drv_bind`:入口 + `component_bind_all` 前后 + `drm_dev_register` 后
- `sun4i_drv_add_endpoints`:已有的节点名/状态日志

**尚不能排除的根因** [源:推断]:
1. `sun4i_drv_bind` 被调用但 `component_bind_all` 失败(如某个组件的 `.bind` 回调返回错误)→ 需 dmesg 确认
2. module 加载顺序导致 probe 时序微妙 → debug printk 可确认 probe 是否被调用
3. `sun4i_drv_probe` 中 `count==0`(极低概率)→ debug printk 可确认
4. `component_match_add_release` 的 devres 分配失败 → page_alloc WARNING 提示可能

**验证步骤**(下一次 rebuild + deploy 后执行):
1. `dmesg | grep '\[sun4i-drv-debug\]'` — 确认 probe 和 bind 调用链
2. `dmesg | grep -E 'Adding component|No output to bind'` — 确认 component 收集
3. 若 bind 被调用但失败:错误码会指示具体失败点
4. 若 probe 完全未被调用:在 extlinux.conf 加 `fw_devlink=off` 重试
5. 若 count==0:DT 解析异常，需在设备上 dump dtb 对比

**2026-05-22 追加**([源:实测]):用户确认 TTY 登录界面在物理 LCD 上可见。同时验证:
- `wl_355608_a8_panel_regs[]`(RG35XX-Plus SPI init 序列)对 SP 的 `fog_fj035fhd05_v1` 模组**完全兼容** — §4.4 风险已消除
- `fog_fj035fhd05_v1_mode[]`(BSP 实测时序,htotal 720 / vtotal 560 / 59.52 Hz)工作正常 — §4.8 时序差异问题已消除

更新章节:§0(状态)、§4.4、§99(本条)

### 2026-05-22 #9 — 根因定位: tcon_tv0 缺失 `status = "disabled"` 阻塞 component bind ✅ 修复

**根因**([源:实测] dmesg 确认):
- Batocera 0002 patch 08 添加 `tcon_tv0` 节点时**遗漏了 `status = "disabled"`**
- DTS 规范: 节点无 `status` 属性 → 默认 `"okay"`
- OF graph 遍历发现 `tcon_tv0 avail=1` → 加入 component match 列表(第 5 个组件)
- 但 `sun50i-h616-tcon-tv` 不在 `sun4i_tcon_of_table` → 无 driver 绑定 → `component_add` 永远不会被调用
- Component 框架等 5 个组件(含 tcon_tv0)，只收到 4 个(mixer0/mixer1/tcon_top/tcon_lcd0)
- `find_components` 永远返回 `-ENXIO` → `sun4i_drv_bind` 永远不会被调用 → **DRM card0 永远不会创建**

**dmesg 实证**:
```
[sun4i-drv-debug] add_endpoints: node=/soc/lcd-controller@6515000 frontend=0 avail=1 connector=0
[drm:sun4i_drv_probe [sun4i_drm]] Adding component /soc/lcd-controller@6515000
```
tcon_tv0@6515000 被加入 component list，但它在设备上永远不会 probe。

**修复**:
- `sun50i-h616.dtsi` 的 `tcon_tv0` 节点加 `status = "disabled";`(+1 行)
- 与其他 display 节点(display-engine/hdmi/tcon_lcd0)的约定一致
- Component 匹配从 5 个缩减到正确的 4 个 → `sun4i_drv_bind` 应能正常调用

**额外提交**:
- `sun4i_drv.c` + `sun8i_mixer.c` 加 9 个 `pr_info` tracepoint
- 覆盖 probe 入口/管线数/count/bind 入口/component_bind_all/drm_dev_register

**验证**(rebuild + deploy 后):
1. `dmesg | grep '\[sun4i-drv-debug\]'` — 期望看到完整的 probe → bind → DRM card created 链路
2. `ls /sys/class/drm/` — 期望看到 `card0*`
3. `dmesg | grep -i 'Adding component.*6515000'` — 期望**不再**出现 tcon_tv0

更新章节: §0(状态)、§99(本条)

### 2026-05-22 #10 — 🎉 屏亮！DRM card0 创建成功

**验证结果**([源:实测] dmesg + sysfs):

修复 `8dfd74be3` (tcon_tv0 加 `status = "disabled"`) 后 rebuild + deploy，完整链路成功:

```
[12.17] probe: starting for display-engine
[12.19] probe: 2 pipelines from allwinner,pipelines
       add_endpoints: mixer0 avail=1 → 加入
       add_endpoints: mixer1 avail=1 → 加入
       add_endpoints: tcon_top avail=1 → 加入 ×2(重复)
       add_endpoints: tcon_lcd0 avail=1 → 加入 ×4(重复)
       add_endpoints: tcon_tv0 avail=0 → ✅ 跳过(disabled)
       add_endpoints: hdmi avail=0 → ✅ 跳过(disabled)
[12.57] probe: count=8 match=0x4d28c3b5 → component_master_add_with_match
[13.53] bind: called for dev=display-engine     🎉
[13.59] bind: component_bind_all OK             🎉
[13.62] bind: drm_dev_register OK, DRM card created 🎉
```

`/sys/class/drm/card0` + `card0-Unknown-1` 已存在。

**关键数据**:
- count=8(mixer0 + mixer1 + tcon_top×2 + tcon_lcd0×4 = 8 match entry, 4 个唯一组件)
- probe→bind 间隔 ~1.3s(module 加载 + component_add 时序)
- tcon_tv0@6515000 全部 `avail=0` 且被正确跳过
- hdmi@6000000 全部 `avail=0` 且被正确跳过

更新章节:§0(状态)、§99(本条)

### 2026-05-22 #7 — 设备首次上电验证(屏未亮)

**操作**:用户已推到设备(远程部署并重启),kernel 7.0.9 启动成功。ssh root@DEVICE_HOST 检查显示子系统状态。

**设备环境**([源:实测]):
- `uname -r` = `7.0.9`,uptime 6 min
- panfrost 仍报 `-110`(IOMMU 修了但 panfrost deferred probe timeout 未消 — 独立问题,不影响 display)

**已确认加载的模块**([源:实测] `lsmod`):
- `panel_newvision_nv3052c` ✓(probe 成功,panel@0 enforce active low on GPIO handle)
- `sun4i_drm` ✓,`sun4i_tcon` ✓,`sun8i_tcon_top` ✓,`sun8i_mixer` ✓
- `spi_gpio` ✓,`gpio_backlight` ✓,`drm_kms_helper` ✓,`drm` ✓,`backlight` ✓
- `drm_mipi_dbi` ✓(panel 依赖)

**已确认绑定的 platform driver**([源:实测] `readlink .../driver`):

| DT 节点 | platform device | 绑定 driver |
|---|---|---|
| `bus@1000000` | `1000000.bus` | `sun50i-de2-bus` ✓ |
| `mixer@280000` | `1280000.mixer` | `sun8i-mixer` ✓ |
| `mixer@2a0000` | `12a0000.mixer` | `sun8i-mixer` ✓ |
| `planes@100000` | `1100000.planes` | `sun50i-planes` ✓ |
| `clock@8000` | `1008000.clock` | `sunxi-de2-clks` ✓ |
| `tcon-top@6510000` | `6510000.tcon-top` | `sun8i-tcon-top` ✓ |
| `tcon_lcd0@6511000` | `6511000.lcd-controller` | `sun4i-tcon` ✓ |
| `display-engine` | `display-engine` | `sun4i-drm` ✓ |

**Panel SPI 设备**([源:实测]):
- `spi0.0` modalias = `spi:rg35xx-sp-panel`,compatible = `anbernic,rg35xx-sp-panel` ✓
- `panel@0 enforce active low on GPIO handle` — reset GPIO 极性正确 ✓

**背光**([源:实测]):
- `/sys/class/backlight/backlight/{brightness,actual_brightness}` = 1,max_brightness = 1
- GPIO backlight 工作 ✓

**IOMMU**([源:实测]):
- `sun50i-iommu` builtin,`30f0000.iommu` driver 绑定 ✓(P0-4 修好)
- panfrost 仍 `-110` — deferred probe timeout,可能是时序问题(独立于 display)

**DT compatible 串验证**([源:实测] `cat .../of_node/compatible | cat -v`):
- `display-engine`: `allwinner,sun50i-h616-display-engine` ✓
- `bus@1000000`: `allwinner,sun50i-h616-de33,allwinner,sun50i-a64-de2` ✓
- `tcon_lcd0`: `allwinner,sun50i-h616-tcon-lcd,allwinner,sun8i-r40-tcon-lcd` ✓
- `tcon_tv0`: `allwinner,sun50i-h616-tcon-tv` ✓
- `tcon-top`: `allwinner,sun50i-h616-tcon-top,allwinner,sun50i-h6-tcon-top` ✓

**DRM endpoint 连线**([源:实测] phandle 追踪):
- TCON LCD0 port@0/endpoint@0 → tcon-top@6510000 port@1/endpoint@0(phandle 0x30) — 输入端
- TCON LCD0 port@0/endpoint@1 → tcon-top@6510000 port@3/endpoint@0(phandle 0x31) — LVDS 备份
- TCON LCD0 port@1/endpoint@0 → spi/panel@0/port/endpoint(phandle 0x32) — 输出端接 panel ✓

**❌ 屏未亮 — DRM card 未创建**([源:实测]):
- `/sys/class/drm/` 只有 `version`,无 `card0`
- `/sys/class/graphics/` 只有 `fbcon`,无 `fb0`
- `/sys/class/vtconsole/vtcon0` = `dummy device`(无 framebuffer vtconsole)
- `display-engine` 设备存在且 `sun4i-drm` driver 已绑定,但 `waiting_for_supplier = 1`

**深入分析**([源:实测+推断]):

OF graph 连线已逐 phandle 验证,结构正确:
- mixer@280000 port@1/endpoint → tcon-top@6510000 port@0(phandle 0x0f) ✓
- mixer@2a0000 port@1/endpoint → tcon-top@6510000 port@2(phandle 0x10) ✓
- tcon-top port@1 → tcon_lcd@6511000(phandle 0x29) ✓
- tcon-top port@3 → tcon_tv@6515000(phandle 0x2c) ✓
- tcon_lcd port@1/endpoint@0 → panel@0(phandle 0x32) ✓

Component 框架应该收集到 5 个组件(mixer0/mixer1/tcon-top/tcon_lcd0/tcon_tv0)。所有组件 driver 均有 `bind`/`unbind`/`uevent`(注册正常),但 `sun4i-drm` driver 目录**缺少这些文件** — 这说明 driver kobject 初始化不完整。

`display-engine` 的 `waiting_for_supplier = 1`([源:实测] sysfs 目录可见但 cat 会阻塞)表明 fw_devlink 创建了一个未满足的 supplier device link。最可能的 supplier 是 `bus@1000000`(DE33 bus — 通过 `allwinner,pipelines` phandle 间接引用)。如果 DE33 bus 本身有未满足的 supplier(如 SRAM-C claim 失败),会级联阻塞 display-engine。

page_alloc WARNING([源:实测] `mm/page_alloc.c:5234` @ `__alloc_frozen_pages_noprof+0x24c`)来自 `component_match_realloc` 分配内存,是 high-order allocation 警告,非 fatal 但说明 match 数组分配走了 slowpath。

**根因方向**(按可能性排序):
1. **fw_devlink 阻塞 component bind** — `display-engine` 的 `waiting_for_supplier=1` 意味着某个 supplier 未 probe,导致 component master 的 bind 回调 `sun4i_drv_bind` 从未执行。`sun4i-drm` driver 目录缺 `bind`/`unbind` 文件佐证此点
2. **DE33 bus SRAM-C claim 问题** — `sun50i-de2-bus` probe 调 `sunxi_sram_claim()`,如果 H616 SRAM-C 走 a64 fallback 但寄存器 layout 不完全兼容,claim 可能静默失败(无 dmesg 输出),导致 bus 子设备虽 populate 但 fw_devlink 认为 bus 不 ready
3. **page alloc WARNING 导致 match 失败** — 虽然 WARNING 本身非 fatal,但如果 component_match_realloc 的 kmalloc 返回 NULL(极低概率),count 可能为 0

**验证方式**(待执行):
1. **最关键**: `rg35xxsp.config` 加 `CONFIG_DRM_DEBUG_DRIVER=y` rebuild — 看 `Adding component %pOF` 输出确认 component 列表
2. **fw_devlink 排除**: cmdline 加 `fw_devlink=off` 测试(在 extlinux.conf append),如屏亮则确认是 fw_devlink 问题
3. **component debug**: `CONFIG_DEBUG_DRIVER=y` + `CONFIG_DEBUG_DEVRES=y` 看 component bind 细节
4. **SRAM-C 排查**: `dmesg | grep sram` 检查是否有 claim 失败;检查 `/sys/firmware/devicetree/base/soc/bus@1000000/allwinner,sram` 引用的 phandle 0x0c (sram-section@0) 是否被正确 claim

**⚠️ SSH 注意**:读 `waiting_for_supplier` sysfs 文件会导致 SSH 会话阻塞(内核可能持有 mutex 不释放),需避免直接 cat 该文件。

更新章节:§0 状态、§99(本条)

### 2026-05-22 #6 — 方案 A 落地 + 两个 build break 修复

**Batocera 11 子 patch 应用**([源:实测]):
- `git mailsplit` 把 0002 大 patch 切成 21 个 mbox
- `git am -3` 顺序应用 `01/02/03/04/07/08/16/17/18/19/21` 共 11 个 — **全部干净 apply,无 conflict**(印证 §4.9)

**自家 patch 3 个**([源:推断/工程]):
- `36df6ca2b [local] drm/panel: nv3052c: add Anbernic RG35XX-SP variant`
  - 加 `fog_fj035fhd05_v1_mode[]` + `fog_fj035fhd05_v1_panel_info` 用 BSP 实测时序(§8.2)
  - 加 compatible `anbernic,rg35xx-sp-panel` 到 of_match 表
  - sp.dts 加 `&panel { compatible = "anbernic,rg35xx-sp-panel"; }` override
- `5e33755d1 [local] drm/sun4i: export sun8i_(vi|ui)_layer_init_one`
  - 修 build break #1:`modpost ERROR: ... [sun50i_planes.ko] undefined!`
  - Batocera patch 03 引入新 sun50i_planes.ko 跨 module 调用,但没 EXPORT 原符号
- `7f7708164 [local] drm/sun4i: merge sun50i_planes into sun8i-mixer.ko`
  - 修 build break #2:`depmod: ERROR: Cycle detected: sun8i_mixer -> sun50i_planes -> sun8i_mixer`
  - 改 Makefile 把 planes.o 加进 sun8i-mixer-y;sun50i_planes.c 删 `module_platform_driver()` + MODULE_*;sun8i_mixer.c 用 `platform_register_drivers()` 数组一次注册两个 driver

**Kernel release 干净度**([源:实测]):
- 起 git 后 setlocalversion 探到仓库 → KERNELRELEASE 变 `7.0.9-g7f77...`,modules 路径错乱
- 修法两点 combine:
  - `rg35xxsp.config` 加 `# CONFIG_LOCALVERSION_AUTO is not set`
  - `build_kernel.sh` 给 make 加 `LOCALVERSION=""`(压住 setlocalversion 的 `+` 追加)
- 最终 `uname -r` = `7.0.9`,modules 装到 `/lib/modules/7.0.9/`（安装到与内核版本匹配的目录）

**`rg35xxsp.config` 新增 4 行 Kconfig**:
- `CONFIG_DRM_PANEL_NEWVISION_NV3052C=m`
- `CONFIG_SPI_GPIO=m`
- `CONFIG_BACKLIGHT_GPIO=m`
- `# CONFIG_LOCALVERSION_AUTO is not set`

**Build 产物**([源:实测]):
- `p2-payload/Image.gz` 14.3 MB,`sun50i-h700-anbernic-rg35xx-sp.dtb` 25.5 KB
- p2 占用 14 MB / 30 MB 上限 ✓
- `modules.tar.gz` 20 MB(含 `sun8i-mixer.ko / sun4i-tcon.ko / panel-newvision-nv3052c.ko / gpio_backlight.ko / pwm_bl.ko`)
- `SUN50I_IOMMU` builtin(modules 内只有 apple-dart.ko,符合预期)
- dtb 内 `strings` 验证:含 `anbernic,rg35xx-sp-panel`、`sun50i-h616-display-engine`、`sun50i-h616-de33-mixer-0`、`sun50i-h616-tcon-lcd` 等关键 compatible


更新章节:§0 状态、§99(本条)

### 2026-05-22 #5 — 预检 + git baseline + 4 项新发现

应用 Batocera patch 前的预检,在 `linux-7.0.9` 源码 grep + ssh root@DEVICE_HOST 实测:

- ✅ [源:实测] H616 CCU clock 全齐:`CLK_TCON_LCD0=130`、`CLK_BUS_TCON_LCD0=131`、`pll-video0/1/2` + 4x 变体、`tcon_tv_parents`(共享给 tcon_lcd)— 见 §4.6
- ⚠️ [源:实测] H616 SRAM-C 走 a64 fallback:7.0.9 `sunxi_sram.c` 没注册 `sun50i-h616-sram-c`,Batocera dtsi 用 fallback 兼容串 → driver 走 a64,**待上电验证**(§4.7)
- ❌ [源:实测+厂商] **mainline `wl_355608_a8_mode[0]` 时序与 BSP 不匹配**:htotal 770 vs 720、vtotal 520 vs 560、hfp 64 vs 24 等。必须在 NV3052C 驱动新增 `fog_fj035fhd05_v1_panel_info`(§4.8、§10.2)
- ⚠️ [源:实测] Batocera 0002 用 `patch -p1` 在 7.0.9 上 dry-run 1 hunk FAIL(`sun50i-h616.dtsi:1154`,子 patch 18 依赖子 patch 08 已应用)→ 必须 `git mailsplit + git am -3` 顺序应用(§4.9)
- ✅ [源:实测] `vcc-cldo3` 实测 3.3V enabled(§4.10)
- ✅ [源:厂商+实测] Panel reset 极性 BSP / mainline / Batocera 全部 active-low,一致(§4.10)

工程基础设施:
- 顺手把 `CONFIG_SUN50I_IOMMU=y` 加进 `rg35xxsp.config` 修 P0-4 panfrost(独立问题,详 issues.md P0-4)

更新章节:§0(状态) / §4.6-4.10(预检) / §10(落地清单,加 git am 流程 + panel info fork) / §11 / §12(新增 git 工作流)

### 2026-05-22 #4 — 来源等级约定 + 启动 BSP 决策

- 新增 §00「来源等级约定」(五档:厂商 / 实测 / 社区 / 推断 / 假设)
- 给 §0 / §1 / §2 / §4 / §8 关键论断补来源标
- 新增 §9「决策:是否启动原厂 BSP 镜像?」结论:**不启动**,详见 §9.6
  - 最高价值数据(panel SPI 序列)启 BSP 也拿不到(§9.4)
  - p5 原厂 rootfs 无备份,实际启动需重刷一张原厂 TF 卡(§9.3)
  - 次要数据等方案 A 落地后跑 mainline 同样可读,对比更直接(§9.5)
- 备用方案三选一记在 §9.7
- 章节重排:§8 BSP 数据 → §9 决策 → §10 落地清单 → §11 引用 → §99 日志

### 2026-05-22 #3 — 原厂 init 序列调研封顶(接受残留风险)

试图从 p4 boot.img 抠 `lcd_panel_fog_fj035fhd05_v1_init` 寄存器序列:
- ✅ 确认符号在 BSP 4.9.170 kernel ARM64 Image 中(字符串偏移 `0x974a3a`)
- ✅ 确认 `__lcd_panel.name[32]` 缓冲在 `0xf8c580`(`fog_fj035fhd05_v1\0` + 全 0 填充)
- ❌ 该处后续的 8 个函数指针槽位全 0 — 不是 ops 表(可能是 driver_name 字段,真正的 ops 表在别处通过 panel_array[] 间接索引)
- ❌ 本地 BSP 源码三棵树都没该 panel `.c` 文件(用户澄清:`linux-allwinner-4.9.170` 是自建混合,**不可作 ground truth**)
- ❌ GitHub 公开搜无 `fog_fj035fhd05_v1` / `fj035fhd05_v1_init` 命中 — Anbernic vendor 私有 patch,未公开

**判断**:反汇编 17 MB ARM64 Image 工作量 > 一上午,**不划算**。同 NV3052C controller、同 640×480、同模组家族,mainline 驱动大概率直接可用。**风险延迟到首次上电试**,失败再回头深挖(§4.4 给出三级升级路径)。

更新章节:§0 / §4.4

### 2026-05-22 #2 — 从原厂 BSP 抠出全部硬件数据

通过 ssh root@DEVICE_HOST 直连设备 + 反编译 p4 Android boot image 内三个候选 dtb,在不上电试错的前提下消除了 §4 大部分不确定项:

- ✅ **Panel 版本确认 v1**:p4 三个候选 dtb 中,实测生效的 dtb 0 `lcd_driver_name = fog_fj035fhd05_v1`,640×480。不存在 v2/rev6 字符串
- ✅ **GPIO 走线 6 个全部确认与 Batocera 0002 patch 19/21 一致**(PI8/9/10/14/15 + PD28)
- ✅ **LCD 时序参数全部抠出**(24 MHz dclk / hbp 48 / ht 720 / vbp 48 / vt 560 / 60 Hz)— 见 §8.2
- ✅ **PWM 背光参数**:PWM0,50 kHz,active-high,默认亮度 50/255 — Batocera 0007 用的 40 kHz,需对齐成 50 kHz

更新章节:§0 / §4.1-4.4 / §8(新增)

### 2026-05-22 #1 — 调研收尾

- 完成驱动 / dtsi / DTS / defconfig 四层缺失审计(§1)
- 整理 Batocera `linux_patches/0002` 21 个子 patch 的 LCD-only / +HDMI 划分(§2)
- 用户选定方案 A
- 阻塞项确定:panel 是 v1 / v2(§4.1)— 给出 3 条设备侧验证命令(§5)
- 创建本文档
