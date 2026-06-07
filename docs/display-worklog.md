# RG35XX-SP 主线点屏(P0-3)工作记录

> 配套 `issues.md` P0-3「屏黑」。跟踪从调研 → 落地 → 点亮的全过程。
> 启动:2026-05-22。文档为活文档,有进展或发现先追加 §99「进展日志」。

---

## 00. 来源等级约定(2026-05-22 加)

文档中事实/判断按以下五档标注来源,可靠度由高到低:

| 标 | 含义 | 可靠度 | 典型 |
|---|---|---|---|
| `[源:厂商]` | 一手 — Allwinner / Anbernic 官方文档、原厂二进制(boot.img / DTB / U-Boot 字符串)、SoC datasheet、原厂 BSP kernel image 内含符号 | 最高 | H616 Datasheet、p4 dtb 抠出的 `lcd_driver_name` |
| `[源:实测]` | 一手 — `ssh sp` 直读 `/sys /proc /dev/mem`、dd raw 分区、dmesg、串口日志、设备物理测量 | 高 | 当前 cmdline、`/sys/kernel/debug/pinctrl/...` 输出 |
| `[源:社区]` | 二手 — linux-sunxi wiki、Batocera / ROCKNIX、kernel 上游 commit、GitHub 公开仓 | 中 | Batocera 0002 patch、linux-sunxi H616 wiki |
| `[源:推断]` | 二手 — 我们综合多个来源推出的结论 / 跨设备类比 | 低 | 不同信息合并、未单独验证的因果 |
| `[源:假设]` | 二手 — 未经任何证据支持,仅作工作前提 | 待证 | 上电前的「应该可用」类判断 |

混合来源用 `[源:厂商+社区]` 等组合。**关键论断必须在结论后括号内标源**;描述性 / 项目工程性内容(任务清单、命令示例)不标。

---

## 0. 当前状态(快照,每次更新都覆盖此节)

| 项 | 状态 |
|---|---|
| 阶段 | ✅ **屏亮！TTY 登录界面可见(§99 #10). P0-3 完成。** |
| 选定方案 | A(LCD-only + GPIO 背光,后续再升 PWM) |
| Panel 版本 | ✅ **v1**(`fog_fj035fhd05_v1`,640×480 — [源:厂商] p4 BSP dtb 实测;[源:实测] 用户 2026-05-22 二次确认物理分辨率) |
| GPIO 走线 | ✅ 5+1 个 GPIO 与 Batocera DTS 一致([源:厂商] p4 dtb `lcd_gpio_N` × [源:社区] Batocera 0002,见 §8) |
| LCD 时序(dtsi 侧)| ✅ 全部抠出([源:厂商] p4 dtb,见 §8.2) |
| H616 CCU clocks | ✅ 全齐(§4.6) |
| H616 SRAM-C 控制器 | ⚠️ 走 a64 fallback,待上电验证(§4.7) |
| Mainline panel 时序 | ✅ 已 fork `fog_fj035fhd05_v1_panel_info` + 新 compatible `anbernic,rg35xx-sp-panel`(commit `36df6ca2b`,§4.8) |
| Batocera patch 应用 | ✅ `git mailsplit + git am -3` 应用 11 个子 patch 全部干净(§99 #5) |
| Build break #1 | ✅ 修复 — sun8i_(vi|ui)_layer_init_one 跨模块调用需 EXPORT_SYMBOL_GPL(commit `5e33755d1`) |
| Build break #2 | ✅ 修复 — `sun8i_mixer ↔ sun50i_planes` 模块循环 → 合并到 sun8i-mixer.ko(commit `7f7708164`) |
| KERNELRELEASE 干净度 | ✅ `7.0.9`(关 LOCALVERSION_AUTO + `LOCALVERSION=""` 压 `+`;commit `22f398040` + rg35xxsp.config + build_kernel.sh) |
| AXP717 cldo3 | ✅ enabled 3.3V(§4.10) |
| Panel reset 极性 | ✅ active-low,一致(§4.10) |
| NV3052C SPI init 序列 | ✅ 兼容 — `wl_355608_a8_panel_regs[]` 直接可用,TTY 显示正常(§99 #10 实测) |
| **产物** | `out/{Image.gz,sun50i-h700-anbernic-rg35xx-sp.dtb}` + `modules.tar.gz`(20 MB,路径 `/lib/modules/7.0.9/`)|
| 已落地的 Kconfig | `CONFIG_SUN50I_IOMMU=y`(P0-4)+ `CONFIG_DRM_PANEL_NEWVISION_NV3052C=m` + `CONFIG_SPI_GPIO=m` + `CONFIG_BACKLIGHT_GPIO=m` |
| Git baseline | ✅ `linux-7.0.9` 16 commits(baseline → 11 batocera → 3 local → .gitignore)(§12) |
| 启动 BSP? | ❌ 不启动(决策见 §9) |
| 调试设施 | ✅ `CONFIG_DRM_DEBUG_DRIVER=y` + `CONFIG_DRM_DEBUG=y` + 9 个 `[sun4i-drv-debug]` printk(commit `12e89e8e4`) |
| Git tag | ✅ `lcd-stage-1-lit`(首屏成功里程碑,commit `12e89e8e4`) |
| 下一步 | 1)✅ 屏亮 + TTY 可见 — P0-3 完成 2)后续可选:PWM 背光(方案 B) 3)GPU panfrost 仍 -110(P0-4) |

---

## 1. 三层缺失清单(根因审计,2026-05-22)

issues.md 写「~40 行 SP DTS overlay」是**严重低估**([源:实测] 检查 7.0.9 源码得到的反驳证据)。实际是 driver + dtsi + 板级 DTS + defconfig **四层**都缺。

### 1.1 驱动层(`src/linux-7.0.9/drivers/gpu/drm/sun4i/` + `panel/` + `pwm/`)

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

### 1.2 dtsi 层(`src/linux-7.0.9/arch/arm64/boot/dts/allwinner/sun50i-h616.dtsi`)

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

## 2. Batocera 可复用 patch 清单(`thirdparty/batocera-h700/board-h700/linux_patches/`)

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

通过 ssh sp 拉 p4 boot 镜像反编译三个候选 dtb(`lcd_type=0/1/2`):

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

| 参数 | mainline | BSP dtb([源:厂商]) | diff |
|---|---|---|---|
| clock (kHz) | 24000 | 24000 | ✅ |
| hdisplay | 640 | 640 | ✅ |
| hfp(hsync_start − hdisplay) | **64** | **24** | ❌ |
| hsync(hsync_end − hsync_start) | **20** | **8** | ❌ |
| hbp(htotal − hsync_end) | **46** | **48** | 接近 |
| htotal | **770** | **720** | ❌ -50 |
| vdisplay | 480 | 480 | ✅ |
| vfp | **21** | **28** | ❌ |
| vsync | 4 | 4 | ✅ |
| vbp | **15** | **48** | ❌ |
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

[源:推断] 解决方案:**用 `git am` 把 21 个子 patch 拆成 21 个独立 commit 顺序 apply**。这正好契合用户 2026-05-22 提议的"linux-7.0.9 做 git 仓库"。Patch 已是 mailbox 格式(每子 patch 含 `From ... Subject:` 头),`git mailsplit + git am -3` 直接可用。

### 4.10 Panel reset 极性 + AXP717 cldo3 实测 — ✅ [源:厂商+实测]

- **Reset 极性**:[源:厂商] BSP dtb `lcd_gpio_3 = <0x08 0x0e 0x01 ... 0x01>` 末位 `data=0x01`(默认 high = deassert)→ active-low;[源:实测] mainline NV3052C `gpiod_set_value(reset, 1)` 后跟 `set_value(reset, 0)` + Batocera DTS `GPIO_ACTIVE_LOW` → 一致
- **`reg_cldo3`(vcc-io,给 PD bank 供电)**:[源:实测] `ssh sp 'cat /sys/class/regulator/.../name,state,microvolts'` → `vcc-io enabled 3300000` ✅(P0-7 AXP DMA mask 问题不影响 cldo3 输出)

---

## 8. 原厂 BSP 关键数据汇总(2026-05-22 抠出)

[源:厂商] 一手数据。来源:`ssh sp 'dd if=/dev/mmcblk0p4 ...'` + `dtc -I dtb -O dts`,见 §99 日志 2026-05-22 #2。

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
lcd_width = 0x96     # 物理宽度 ~150mm
lcd_height = 0x5e    # 物理高度 ~94mm  (→ 3.5" 4:3 IPS)
lcd_dclk_freq = 24   # MHz
lcd_hbp = 48         # h back porch
lcd_ht = 720         # h total = active(640) + hbp(48) + hsync(8) + hfp(24)
lcd_hspw = 8         # h sync pulse width
lcd_vbp = 48         # v back porch
lcd_vt = 560         # v total = active(480) + vbp(48) + vsync(4) + vfp(28)
lcd_vspw = 4         # v sync pulse width
lcd_hv_clk_phase = 2
lcd_hv_sync_polarity = 3   # HSYNC + VSYNC both active-low
lcd_frm = 1                # frame buffer dithering(BSP 特性,mainline 无)
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

[源:实测] `dmesg | grep SMCCC` 当前 mainline 7.0.9 启动日志直接读到。

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

### 9.4 关键数据(panel SPI 命令序列)BSP 也拿不到

[源:推断] 即便能启动 BSP:
- BSP 内核无源 → 不能加 dynamic_debug
- disp2 框架内 SPI 调用([源:实测] 见 §99 #3 — `__lcd_panel.name[32]` 后续 8 个槽位全 0)不暴露 byte log,真正的 ops 表通过 `panel_array[]` 间接索引
- ftrace 函数级跟踪能跟到 `lcd_panel_fog_fj035fhd05_v1_init`,但参数(SPI command byte)不在 ARM64 ftrace 调用栈
- **唯一可靠路径:物理逻辑分析仪抓 PI9/PI10**(BSP 启动与否对此无影响)

### 9.5 mainline 7.0.9 上能拿到什么(2026-05-22 实测)

[源:实测] `ssh sp` 在当前运行的 mainline 7.0.9 上跑同样命令的结果:

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

## 10. 方案 A 落地清单(2026-05-22 更新,等执行)

[源:推断] 工程步骤,基于 §2 patch 清单 + §1.4 Kconfig + §4.6/4.7/4.8/4.9 预检结论。

### 10.1 在 `linux-7.0.9` 仓库内顺序 apply Batocera 子 patch

```bash
cd src/linux-7.0.9
mkdir /tmp/batocera-split
cd /tmp/batocera-split
git mailsplit -o. /path/to/0002-rg35xx-enable-HDMI-LCD.patch
# 得 0001..0021 21 个独立 mbox 文件
cd src/linux-7.0.9
# 按方案 A 应用 11 个子 patch(跳过 05/06/09/10/11/12/13/14/15/20 — HDMI 或 bindings YAML)
for n in 0001 0002 0003 0004 0007 0008 0016 0017 0018 0019 0021; do
    git am -3 /tmp/batocera-split/$n
done
```

### 10.2 NV3052C 驱动 fork `fog_fj035fhd05_v1_panel_info`(§4.8)

新增到 `drivers/gpu/drm/panel/panel-newvision-nv3052c.c`(估 ~40 行):
- `fog_fj035fhd05_v1_mode[]`:用 [源:厂商] §8.2 时序值(htotal 720 / vtotal 560 / hsync_start 664 等)
- `fog_fj035fhd05_v1_panel_info`:`width_mm=70 height_mm=53`(从 BSP `lcd_width=0x96 lcd_height=0x5e` 折算 — 注意 BSP 单位是 0.1 mm),`bus_format=MEDIA_BUS_FMT_RGB888_1X24`,`panel_regs=wl_355608_a8_panel_regs`(暂复用,§4.4)
- `nv3052c_of_match[]` 新加一条 `{ .compatible = "anbernic,rg35xx-sp-panel", .data = &fog_fj035fhd05_v1_panel_info }`
- commit 信息标 `[本地] anbernic: add rg35xx-sp panel timing for fog_fj035fhd05_v1`

板级 DTS(由 Batocera 子 patch 19/21 改 2024.dts)的 panel compatible 改为 `"anbernic,rg35xx-sp-panel"`(SP 专用,**不复用 plus-panel**)。

### 10.3 `rg35xxsp.config` 追加 LCD Kconfig

```
# === Panel / LCD(P0-3)===
CONFIG_DRM_PANEL_NEWVISION_NV3052C=m
CONFIG_SPI_GPIO=m
CONFIG_BACKLIGHT_GPIO=m
```

### 10.4 调整 `build_kernel.sh`(可选,等真出问题再改)

当前用 `patch -p1` 吃 `patches/*.patch`。git 工作流后建议改成 `git am -3`,但当前 11 个 patch 已在 linux-7.0.9 git 里,build script 不必再次 apply。要么:
- (a) 把 git 内 patches 用 `git format-patch` 导回 `bsp/build/c/patches/`,build script 保留 `patch -p1` 流程,但去掉 git 跟踪(失去 git 优势)
- (b) 改 build script 跳过 patches 应用,假设 linux-7.0.9 已是含 patch 的目标状态(更纯净)

[源:推断] 推荐 (b)。但**先 Task #3 跑通再决定**。

### 10.5 验证(上电)

```
dmesg | grep -iE "panel|sun4i-tcon|sun8i-mixer|backlight|sun50i-iommu"
ls /sys/class/drm/             # 期望见 card0-DPI-1 (LCD)
ls /sys/class/backlight/       # 期望见一个 backlight 设备
cat /sys/class/backlight/*/brightness
# 物理:LCD 背光亮 + framebuffer console (tty1 输出)
```

若白屏 / 花屏(panel probe 但无图像)→ 见 §4.4 三级升级路径、§9.7 备用方案。

---

## 11. 相关引用

- `issues.md` P0-3 — 原始问题描述
- `issues.md` P0-4 — 同期 IOMMU/panfrost 修(独立问题,共享 rebuild)
- `mainline-support.md` 第 212/263/292 行 — Panel 驱动状态、DE3.3 高级特性差距
- `device-facts.md` §7 / §10.4 / §23 — LCD 物理事实、lcd_type 三选一
- `thirdparty/batocera-h700/board-h700/linux_patches/0002-rg35xx-enable-HDMI-LCD.patch` — 主要参考 [源:社区]
- `thirdparty/batocera-h700/board-h700/linux_patches/0007,0013,0099-*` — 后续升级 / panel 变体 [源:社区]

---

## 12. Git 工作流(2026-05-22 新增)

[源:推断] 工程基础设施。

### 12.1 仓库

- 位置:`bsp/src/linux-7.0.9/.git/`(2026-05-22 新建)
- baseline commit:`a8b03ff baseline: Linux 7.0.9 + build/c/patches/0001-mmc-pwrseq-tolerate-ENOENT applied`
- .gitignore:kernel-aware(忽略 `.config / *.o / *.cmd / vmlinux / Image* / *.dtb / scripts/dtc/* / include/config|generated/...`)
- distclean 后 tree 1.8 GB,`.git` 689 MB

### 12.2 后续提交策略

- 每个 Batocera 子 patch → 一个 commit(`git am -3` 自动以原作者身份创建)
- 我们的本地 patch(panel info fork、build 系统调整)→ 单独 commit,标 `[local]` 前缀
- 标签:Task #3 完成时打 `lcd-stage-1-pre-boot`,首屏成功后打 `lcd-stage-1-lit`

### 12.3 与 `build_kernel.sh` 的兼容性

[源:推断] 当前 build script 用 `patch -p1` 应用 `build/c/patches/*.patch`,**和 git am 应用的 commit 序列会重复**。处理方案见 §10.4。

---

## 99. 进展日志(倒序,最新在上)

### 2026-05-22 #8 — 深入代码静态分析:component bind 链路全审计(本次研究)

**分析目的**:在设备已处于 `flash_sd.sh` 刷新状态时，通过源码审计确定为什么 `sun4i_drv_bind` 没有被调用([源:推断])。

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

**修复**(commit `8dfd74be3`):
- `sun50i-h616.dtsi` 的 `tcon_tv0` 节点加 `status = "disabled";`(+1 行)
- 与其他 display 节点(display-engine/hdmi/tcon_lcd0)的约定一致
- Component 匹配从 5 个缩减到正确的 4 个 → `sun4i_drv_bind` 应能正常调用

**额外提交**(commit `12e89e8e4`):
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

**操作**:用户已推到设备(`deploy_ssh.sh --reboot`),kernel 7.0.9 启动成功。ssh sp 检查显示子系统状态。

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
- 最终 `uname -r` = `7.0.9`,modules 装到 `/lib/modules/7.0.9/`(`deploy_ssh.sh` 内 KVER=7.0.9 hardcode 不用改)

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

**git 状态**:`linux-7.0.9` 16 commits — baseline + 11 batocera + 3 local + 1 .gitignore 修补

更新章节:§0 状态、§99(本条)

### 2026-05-22 #5 — 预检 + git baseline + 4 项新发现

应用 Batocera patch 前的预检,在 `linux-7.0.9` 源码 grep + ssh sp 实测:

- ✅ [源:实测] H616 CCU clock 全齐:`CLK_TCON_LCD0=130`、`CLK_BUS_TCON_LCD0=131`、`pll-video0/1/2` + 4x 变体、`tcon_tv_parents`(共享给 tcon_lcd)— 见 §4.6
- ⚠️ [源:实测] H616 SRAM-C 走 a64 fallback:7.0.9 `sunxi_sram.c` 没注册 `sun50i-h616-sram-c`,Batocera dtsi 用 fallback 兼容串 → driver 走 a64,**待上电验证**(§4.7)
- ❌ [源:实测+厂商] **mainline `wl_355608_a8_mode[0]` 时序与 BSP 不匹配**:htotal 770 vs 720、vtotal 520 vs 560、hfp 64 vs 24 等。必须在 NV3052C 驱动新增 `fog_fj035fhd05_v1_panel_info`(§4.8、§10.2)
- ⚠️ [源:实测] Batocera 0002 用 `patch -p1` 在 7.0.9 上 dry-run 1 hunk FAIL(`sun50i-h616.dtsi:1154`,子 patch 18 依赖子 patch 08 已应用)→ 必须 `git mailsplit + git am -3` 顺序应用(§4.9)
- ✅ [源:实测] `vcc-cldo3` 实测 3.3V enabled(§4.10)
- ✅ [源:厂商+实测] Panel reset 极性 BSP / mainline / Batocera 全部 active-low,一致(§4.10)

工程基础设施:
- 完成 `linux-7.0.9` git init + kernel-aware .gitignore + baseline commit `a8b03ff`(§12)
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

通过 ssh sp 直连设备 + 反编译 p4 Android boot image 内三个候选 dtb,在不上电试错的前提下消除了 §4 大部分不确定项:

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
