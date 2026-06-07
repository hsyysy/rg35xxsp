# HandsomeMod 4.9 内核编译验证报告

> **来源等级声明**：本文档属"二手分析"——记录的是在主机端对 HandsomeMod 4.9 内核源码做 `make oldconfig` / `make Image` 的过程与结论，**不是设备实测、不是官方 PDF**。所有与硬件实际行为相关的断言以 `device-facts.md` 为准；本文档仅作"内核编译可行性"参考。
>
> 设备硬件/运行时事实见 `device-facts.md`（已采用三类权威来源：设备实测 / 串口日志 / 官方 PDF）。

验证日期: 2026-05-16

## 1. `make oldconfig` — 新增符号分析

**共发现 91 个 `(NEW)` 符号**（即 HandsomeMod 2021 快照 Kconfig 中没有、但设备 BSP `config-4.9.170` 需要的选项），其中 **29 个在 HandsomeMod 源码树中完全没有对应驱动文件**——该数字即 `device-facts.md §10.6` 引用的"29 个孤儿符号"。

### PMIC 相关（关键）

| 符号 | 状态 | 说明 |
|------|------|------|
| `AXP2101_POWER` | NEW, 默认 N | AXP2101 电源驱动（与设备实际 AXP2202 不同，但 BSP 同时挂载——见 `device-facts.md §10` 关于 BSP 双驱动栈的说明） |
| `AXP152_VBUS_POWER` | NEW, 默认 N | AXP152 VBUS 检测（设备 PMIC 探测序列残留，非现役驱动） |
| **AXP2202 驱动** | **HandsomeMod 树中不存在** | DTS 引用 `x-powers,axp2202` 但 HandsomeMod 内核源码树中**完全没有对应驱动文件**（设备实跑的内核来自 Anbernic 内部 BSP 2025 私有树） |

### 平台/架构相关

| 符号 | 说明 |
|------|------|
| `ARCH_SUN50IW11` | 新增 SoC 平台（IW9 已有，IW11 是新增选项） |
| `HZ_15` | 15Hz 时钟频率选项 |
| `ARM_SUNXI_CPUFREQ_PWM` | PWM CPUFreq 支持 |

### WiFi/蓝牙

- `XR_WLAN`, `XR829_BT`, `XR_BT_FDI` — XRadio 无线
- `RTL8189ES`, `RTL8189FS`, `RTL8723DS` — Realtek SDIO WiFi 系列（设备现役驱动是 `8821cs`，见 `device-facts.md §18`）
- `BT_HCIUART_RTL3WIRE` — Realtek 三线 UART 蓝牙

### 显示/LCD（约 30+ 个 NEW）

- 大量 LCD panel 驱动（`LCD_SUPPORT_*`，设备实际 panel 见 `device-facts.md §7`）
- `SUNXI_DISP2_FB_DISABLE_ROTATE` / `ROTATION_SUPPORT` — 旋转支持选择
- `HDMI_EP952_DISP2_SUNXI` — EP952 HDMI 驱动

### 其他

- `SUNXI_NNA`, `SUNXI_ISE`, `SUNXI_EISE` — NPU/ISE 加速器
- `CSI_VIN` — 视频输入（默认 y）
- `CRYPTO_SPECK` — Speck 加密算法

## 2. DTS 头文件检查

`dtb/0_1334800.dts` 是**反编译的 DTS**（dtb2dts 输出），**没有 `#include` 指令**——所有引用已展开为原始值（phandle 数字、硬编码地址等）。

- 设备树 `model = "sun50iw9"`, `compatible = "allwinner,h616", "arm,sun50iw9p1"`（与设备实测 `/proc/device-tree/{model,compatible}` 一致——见 `device-facts.md` §10.3 / §十二内存映射对照表）
- 包含 AXP2202 PMIC 节点（`compatible = "x-powers,axp2202"`），完整 regulator 见 `device-facts.md §15`
- **不需要额外头文件**，但此 DTS 无法直接参与内核编译（需要原始 `.dts` 源文件带 include）

## 3. `make Image` 编译结果

### 最终结果：编译成功

```
arch/arm64/boot/Image: Linux kernel ARM64 boot executable Image, little-endian, 4K pages
大小: 18MB
```

### 编译过程中需要修复的问题（共 5 处）

| # | 问题 | 修复方式 |
|---|------|----------|
| 1 | `CONFIG_CC_STACKPROTECTOR_STRONG` 与 GCC 15 不兼容 | `.config` 中禁用该选项 |
| 2 | `dtc` 工具 `yylloc` 多重定义（GCC 10+ `-fno-common`） | `dtc-lexer.lex.c` 中改为 `extern YYLTYPE yylloc` |
| 3 | `ccm0`/`ccm1` 变量多重定义（`-fno-common`） | `sensor_helper.c` 中改为 `extern`，移除重复 `module_param_string` |
| 4 | Makefile `-fno-common` 与 `-Werror=implicit-function-declaration` 等严格标志 | 改为 `-fcommon` 和 `-Wno-implicit-function-declaration` 等 |
| 5 | 显示驱动缺少 SUN50IW9 平台分发 | `disp_features.h`, `disp_private.h`, `de_feat.h`, `de_feat.c`, `de/Makefile` 中添加 SUN50IW9 -> lowlevel_v3x 映射 |

### 仍存在的潜在问题

- **AXP2202 PMIC 驱动 HandsomeMod 树完全缺失** — 不影响 `make Image`，但烧入设备后电源管理/充电/USB 供电不工作（设备现役内核走 Anbernic 私有 BSP 驱动栈，见 `device-facts.md §10`）
- **显示驱动是基于 H6 (SUN50IW6) 特征集硬凑的** — H616 的 DE 硬件细节可能有差异，需实际测试（LCD 参数见 `device-facts.md §7`）
- 大量 `-Wzero-length-bounds` 警告（vin 视频驱动），不影响编译但代码质量有问题

## 结论

**HandsomeMod 4.9 内核可以编出 Image，但需要 5 处源码补丁才能用 GCC 15 编译。编出的内核缺少 AXP2202 PMIC 驱动，实际能否启动板子存疑——电源管理不会工作。** 设备现役内核（`uname -a` 见 `device-facts.md §25`）来自 Anbernic 内部 BSP 2025 私有树，HandsomeMod 树不能直接替换。
