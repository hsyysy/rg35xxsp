# RG35XX-SP 设备实际情况汇总

> **来源可信度规则（适用于本文档全部条目）**
>
> 仅以下三类来源被视为权威，可直接作为事实依据：
> 1. **设备实测**：`ssh root@SP_HOST`（TF1 卡槽内 SD 卡，非 eMMC）通过 `/proc`、`/sys`、`dmesg`、`fdisk`、`strings /dev/mmcblk0p*` 等手段读出的运行时数据
> 2. **设备串口日志**：`scripts/serial.log`（实际重启抓取的 BOOT0/ATF/U-Boot/Kernel printk）
> 3. **官方 PDF 文档**：`doc/*.pdf` —— Allwinner H616 Datasheet / H616 User Manual / H700 brief / H616 USB module manual / H616 Android Q OTA Guide（已转文本到 `pdf-text/`）
>
> **其他来源默认不可信**，包括本仓库内的 `README.md`、`mainline-support.md`、`verification-report.md`、第三方 BSP 树（HandsomeMod、batocera-h700、alpine-h700 等）的注释/文档。若必须引用，须显式标注来源并降级为"推测/未验证"。
>
> 本文档原始内容抽取自 `README.md` 中与 BSP 无关、仅描述设备硬件/固件/运行时状态的条目；**2026-05-20 验证**：经 `ssh sp` 实测 + serial.log 重启 + 官方 PDF 对照，绝大多数条目正确。下表为发现的偏差，已就地修正具体条目；总结见文末"十一、2026-05-20 验证摘要" 和 "十二、H616 PDF 文档补强"。

---

## 一、SoC 与硬件规格

### 1. CPU：Quad Cortex-A53

- **事实**：`/proc/cpuinfo` 报 4 cores
- **验证方式**：`ssh root@SP_HOST cat /proc/cpuinfo`

### 2. GPU：Mali-G31 MP1（非 brief 标称的 MP2）

- **事实**：`/sys/devices/platform/gpu/gpuinfo` 明确报 1 cores，r0p0，ID 0x7093。H700 Brief 标称 MP2，**实测为 MP1**
- **验证方式**：`ssh root@SP_HOST cat /sys/devices/platform/gpu/gpuinfo`
- **备注**：可能是 H700 SKU 在 RG35XX-SP 上 fuse 掉了第 2 个 shader core，或 brief 描述的是上限

### 3. GPU compatible 字符串：`arm,mali-midgard`（BSP DTS）

- **事实**（设备实测 `/proc/device-tree/gpu@0x01800000/compatible`）：BSP DTS 写 `arm,mali-midgard`
- **架构归属**：ARM 官方将 Mali-G31 归类为 **Bifrost** 架构（Midgard 是上一代，对应 Mali-T8xx 系列）。BSP 用 `arm,mali-midgard` 是因为底层 mali_kbase r20p0 驱动复用 Midgard 通信框架，并非 GPU 真的属于 Midgard 架构
- **mainline 写法**：mainline DTS 写 `arm,mali-bifrost`（Panfrost 驱动），与 ARM 官方架构分类一致
- **备注**：常见非官方称呼 "Bifrost Gen2"/"2nd-gen Bifrost" 来自社区/技术博客，ARM 官方产品页面只用 "Bifrost"，不做世代细分——本文档不采用未经官方背书的世代说法

### 4. GPU OPP：6 档频率

- **事实**：420 / 456 / 504 / 552 / 600 / 648 MHz
- **验证方式**：`ssh root@SP_HOST cat /sys/devices/platform/gpu/devfreq/gpu/available_frequencies`

### 5. DRAM：LPDDR4 @ 672 MHz，32-bit 总线，1 GiB

- **事实**：`dram_type=8`（LPDDR4，BSP 编号），32-bit 总线带宽约 5.4 GB/s 峰值
- **验证方式**：
  - 串口 BOOT0 日志（`scripts/serial.log` 2026-05-20 实测重启，锚点搜索）：
    - `DRAM CLK =672 MHZ`
    - `DRAM Type =8 (3:DDR3,4:DDR4,7:LPDDR3,8:LPDDR4)`
    - `Actual DRAM SIZE =1024 M`
    - `[AUTO DEBUG]32bit,1 ranks training success!`
    - `DRAM SIZE =1024 MBytes, para1 = 30fa, para2 = 4000000, dram_tpr13 = 2008c60`
    - DRAM_VCC = 1100 mV，芯片 ID = 0x6c00
  - `dram_type` 来自 BSP 编号定义（8 = LPDDR4）
- **备注**：32-bit 总线带宽是 H6 一半，解释了同 SoC 掌机感知性能 < TV box。LPDDR4 训练首轮 `read_calibration error ×10` → `retraining final error` 后 fallback 仍成功，正常。

### 6. IOMMU：存在

- **事实**：`0x030f0000-030f0fff : /iommu@030f0000`
- **验证方式**：`ssh root@SP_HOST cat /proc/iomem | grep iommu`
- **备注**：mainline DRM/VPU 必需

### 7. LCD 接口：RGB888，640×480 24-bit IPS

- **事实**（设备实测）：SP 用 RGB888 接口（lcd_if=0x0），LCD 控制器输出 640×480，驱动名 `fog_fj035fhd05_v1`
- **屏型号"WL-355608-A8" 提示**：mainline DTS（linux-sunxi 社区树）将此 panel 标称为 WL-355608-A8（3.5" 640×480 24-bit IPS）。**此型号字符串不在设备 `/proc/device-tree` 或 boot.img 二进制中出现**（设备侧只有厂商代号 `fog_fj035fhd05_v1`），属本文档来源规则中的"第三方/社区来源"，未通过设备或官方 PDF 独立验证；仅作交叉参考
- **验证方式**：
  - 设备 DTS 中 `lcd_driver_name` 和 `lcd_x` 属性（实测 `lcd_x=0x280=640, lcd_y=0x1e0=480`）
  - `ssh root@SP_HOST cat /sys/class/disp/disp/attr/sys` → `mgr0: 640x480 fmt[rgb] ... lcd output 640x480`（看 LCD 实际输出而不是 fb0/virtual_size——后者是运行时可变的 fb 配置，不等同于 LCD 物理分辨率，见 §8）
- **备注**：SP 实物无 HDMI 输出口

### 8. 帧缓冲参数

- **事实**：`fb0` 由 sunxi disp framebuffer 提供，**当前运行态实测**：virtual_size=1280×1024、bits_per_pixel=16、stride=2560（2026-05-20）。fb 虚拟尺寸与 bpp 由用户态启动时 ioctl 设置，**不等同于 LCD 物理分辨率**；LCD 实际仍为 640×480。
- **保留物理帧缓冲**：cmdline 中 `disp_reserve=1228800,0x7bf24800` —— 1,228,800 字节 = 640×480×4，从 `0x7BF24800` 起预留一帧 RGBA8888 给 sunxi disp（位于 DRAM 顶部 ~1 GiB 边界附近）
- **验证方式**：
  - `ssh root@SP_HOST cat /sys/class/graphics/fb0/{virtual_size,bits_per_pixel,stride}`
  - `ssh root@SP_HOST cat /proc/cmdline | tr ' ' '\n' | grep disp_reserve`
  - LCD 真实输出看 `cat /sys/class/disp/disp/attr/sys`：`mgr0: 640x480 fmt[rgb] ... lcd output backlight(140) fps:60.2 640x480`
- **备注**：先前记录的 "640×960 双缓冲 32bpp" 是另一时刻的 fb 配置，受运行时 fbset 影响会变；视为非稳定事实。

### 9. 音频：codec @ 0x05096000，扬声器 PA 走 PMIC cldo1

- **事实**：
  - codec 节点 `/proc/device-tree/soc@03000000/codec@0x05096000`，compatible = `allwinner,sunxi-snd-codec`
  - regulator_summary 显示 `5096000.codec` 同时消费 `axp2202-aldo4`（1.8V）与 `axp2202-cldo1`（3.3V）——推断 aldo4 为 codec 模拟/数字供电，cldo1 为扬声器 PA 供电
- **验证方式**：
  - `ssh root@SP_HOST cat /proc/device-tree/soc@03000000/codec@0x05096000/compatible`
  - `ssh root@SP_HOST cat /sys/kernel/debug/regulator/regulator_summary | grep -A1 'aldo4\|cldo1'`
- **备注（2026-05-20）**：设备运行态 DTS 中**没有** `allen_amp_power` 属性，也没有 `allen*` 命名节点；该字符串可能只存在于源码 DTS/驱动中（见 §10.4）。

### 10. PMIC：X-Powers AXP2202（BSP 名）≈ mainline AXP717（同一颗芯片，但驱动栈不同）

- **事实**：物理同一颗 PMIC IC（i2c name=`axp2202`，DTS compatible=`x-powers,axp2202`）。BSP 内核（HandsomeMod / Anbernic 2025 树）使用 **私有 `axp2202-regulator` + `axp2101-regulator` 驱动栈**；mainline 内核使用 `axp20x` family 驱动并把该芯片识别为 **AXP717**。
- **完整 regulator**：4×DCDC + 4×ALDO + 4×BLDO + 4×CLDO + CPUSLDO + RTCLDO + drivevbus + vmid
- **验证方式**：
  - `ssh root@SP_HOST cat /sys/bus/i2c/devices/5-0034/name`
  - 设备 DTS 中 `x-powers,axp2202` compatible
- **重要修订（2026-05-20，对原"BSP 叫 AXP2202 = mainline 叫 AXP717"表述的纠正）**：
  - 两个驱动栈**不是简单的名字映射**：在 `scripts/serial.log` 中搜索锚点 `axp2101-regulator` 可见 `Setting DCDC frequency for unsupported AXP variant` / `Error setting dcdc frequency: -22`。
  - 此错误来自 BSP 自带的 `axp2101-regulator` 驱动在初始化阶段尝试设 DCDC 工作频率，但发现 chip variant 不在它支持表里——说明 BSP 内同时挂载了 `axp2101` 与 `axp2202` 两套驱动，且 variant ID/寄存器布局存在差异，最终由 `axp2202-regulator` 接管。
  - 含义：mainline porting 时，仅靠 `compatible = "x-powers,axp717"` 不一定能复现 BSP 完整功能集（如 fuel-gauge 校准值、private regulator constraints），需要对照 BSP 私有驱动的寄存器写入序列。

### 11. 连接性：实际引出的外设

- **事实**：SP 实际接出：USB OTG、SDIO（RTL8821CS WiFi，驱动 `8821cs.ko` 厂商 RTW 版，**非** mainline `rtl8xxxu`/`rtw88_8821cs`，详见 §18）、UART1（RTL8821CS BT，配合 `rtl_btlpm`）、LRADC（按键）。大部分 brief 项 SP 未引出
- **验证方式**：
  - `ssh root@SP_HOST ls /sys/class/net/`（实测有 **wlan0 与 wlan1** 两个 netdev，均挂在同一 SDIO 设备 mmc2:0001:1 下，对应 RTL8821CS STA + P2P/虚拟接口）
  - `ssh root@SP_HOST ls /sys/class/bluetooth/`（hci0 存在 = BT）
  - `ssh root@SP_HOST lsmod | grep -iE "8821|rtl_bt"`（确认现役驱动模块）
  - 设备 DTS 中各节点 status

### 12. 安全：未启用 OPTEE

- **事实**：cmdline `secure_os_exist=0`，未启用 OPTEE，可自由换内核
- **验证方式**：`ssh root@SP_HOST cat /proc/cmdline | grep secure_os_exist`

### 13. Thermal：5 个 zone

- **事实**：实测 zone 名为 `cpu_thermal_zone` / `gpu_thermal_zone` / `ve_thermal_zone`（视频引擎） / `ddr_thermal_zone` / `axp2202-battery`
- **验证方式**：`ssh root@SP_HOST 'for f in /sys/class/thermal/thermal_zone*/type; do echo "$f: $(cat $f)"; done'`
- **备注**：mainline DTS 只声明 cpu_thermal + gpu_thermal，ve/ddr zone 是 BSP 私有
- **硬件来源（H616 User Manual §2.2.6.7 + §3.10，逐字引用）**：
  - §2.2.6.7 Thermal Sensor Controller，第 4 项 bullet 原文：
    > "Four thermal sensors: sensor0 located in the GPU, sensor1 located in the VE, sensor2 located in the CPU and sensor3 located in the DDR"
  - §3.10.1 Overview 重述（`pdf-text/H616_User_Manual_V1.0_cleaned.txt` 行 9904-9906）：
    > "The Thermal Sensor Controller (THS) embeds four thermal sensors, sensor0 is located in GPU, sensor1 is located in VE, sensor2 is located in CPU, sensor3 is located in DDR."
  - 寄存器（§3.10.5 Register List）：`THS{0,1,2,3}_DATA` @ 0x00C0/C4/C8/CC，`THS{0,1,2,3}_ALARM_CTRL` @ 0x0040-0x004C，全部相对 THS 基址 `0x0507 0400`
  - THS 控制器 @ 0x05070400, 1K（§3.1 内存映射）
  - 精度 ±3°C (0-100°C) / ±5°C (-25~125°C)，**测的是 die 内部温度，不是外壳温度**；die 温度通常比掌机外壳高 10-30°C（受 PCB 散热、塑料壳厚度、内部气流影响），勿用此数据校准用户感知温度或判断设备是否过热
- **mainline DTS porting 指引**：THS 通道号必须按 GPU=0/VE=1/CPU=2/DDR=3 映射，与 BSP zone 编号一致。axp2202-battery 是 PMIC 内独立的电池温度传感器，与 SoC THS 无关。

---

## 二、CPU 频率（OPP 表）

### 14. 实测 9 档 OPP（2026-05-20 重测修订）

- **事实**（`scaling_available_frequencies` 实测）：
  480 / 720 / 936 / 1008 / 1104 / 1200 / 1320 / 1416 / 1512 MHz，共 **9 档**。
  与 `online_log/sys_kernel_debug_cpufreq_table` 中记录的 8 档表（含 1032/1296，缺 936/1008/1320）不一致——以设备实测为准。

  （电压值需通过 `/sys/kernel/debug/opp/...` 或 debugfs cpufreq_table 抓取，本次未单独验证；原表中的电压列保留供参考但已知频率列与现况不符。）

- **验证方式**：
  - `ssh root@SP_HOST cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies`
  - 旧来源：`online_log/sys_kernel_debug_cpufreq_table`（与现役不同，可能是别一 build 的 OPP 表）

---

## 三、PMIC Regulator 路由（22 路）

### 15. AXP2202 完整 regulator → consumer 映射

- **事实**（2026-05-20 实测 `/sys/kernel/debug/regulator/regulator_summary`）：

  | Regulator | min (V) | max (V) | 当前 (mV) | 实测 consumer | 用途推断 |
  |-----------|---------|---------|-----------|---------------|----------|
  | dcdc1 | 0.5 | 1.54 | 1160 | cpu0 | **CPU 核心**（动态调压） |
  | dcdc2 | 0.5 | 3.4 | 940 | — | （无 consumer 绑定；推断 GPU/DRAM 之一，但未在 regulator_summary 上挂） |
  | dcdc3 | 0.5 | 1.84 | 1100 | — | （无 consumer 绑定） |
  | dcdc4 | 1.0 | 3.7 | 1000 | — | — |
  | rtcldo | 1.8 | 1.8 | 1800 | — | RTC（always-on） |
  | aldo1 | 0.5 | 3.5 | 1800 | sdc2 | SDIO（WiFi） |
  | aldo2 | 0.5 | 3.5 | 1800 | — | — |
  | aldo3 | 0.5 | 3.5 | 1800 | — | — |
  | aldo4 | 0.5 | 3.5 | 1800 | 5096000.codec | 音频 codec（数字/模拟 1.8V） |
  | bldo1 | 0.5 | 3.5 | 1800 | — | — |
  | bldo2 | 0.5 | 3.5 | 1800 | — | — |
  | bldo3 | 0.5 | 3.5 | 2800 | — | — |
  | bldo4 | 0.5 | 3.5 | 1200 | — | — |
  | cldo1 | 0.5 | 3.5 | 3300 | 5096000.codec | 推断为扬声器 PA 3.3V（codec 节点亦挂 aldo4） |
  | cldo2 | 0.5 | 3.5 | 3300 | — | — |
  | cldo3 | 0.5 | 3.5 | 3300 | — | — |
  | cldo4 | 0.5 | 3.5 | 3300 | — | WiFi（DTS `wlan_io_regulator=axp2202-cldo4`，但 regulator_summary 无 consumer 绑定，可能通过其他路径间接控制） |
  | cpusldo | 0.5 | 1.4 | 900 | — | CPU SRAM/PLL |
  | vmid | — | — | 0 | — | use=0、电压报 0V，但 LPDDR4 系统正常运行——存在两种可能：(a) VMID 实际由 PMIC 内部固定电路提供（不暴露给 regulator framework），(b) 此板 LPDDR4 用内部自端接 (on-die termination) 而非外部 VMID。**未独立验证**，但功能上不影响 |
  | drivevbus | — | — | 0 | — | USB OTG VBUS（H616 USB 模块手册 §5.1 `usb_drv_vbus_gpio="axp_ctrl"` 路径，由 OTG 状态机按需开关） |

- **验证方式**：
  - `ssh root@SP_HOST cat /sys/kernel/debug/regulator/regulator_summary`
- **关键修正（对原条目）**：
  - 原表声称 dcdc1=核心、dcdc2=GPU、dcdc3=??? 全部 always-on——实测 dcdc1 确实绑 cpu0，但 dcdc2/dcdc3 在 regulator_summary 上**没有 consumer**（use=0），不能直接确认其用途
  - aldo1=sdc2 ✅；aldo4=5096000.codec ✅；cldo1=5096000.codec ✅；
  - cldo4 表中标注 "WiFi（wlan_power）"——DTS 端 `wlan_io_regulator` 字符串属性确为 `axp2202-cldo4`，但运行时 regulator framework 未建立 consumer 绑定。
- **dcdc2/dcdc3 缺 consumer 的合理解释（推断，非实测）**：
  - H616/H700 die 的 GPU 与 DRAM PHY 很可能**不走 AXP2202 的 DCDC 路径**，而是由片内独立电源域或硬连在 dcdc4/其他路上——理由：
    - GPU 在 H616 die 内只占 256K 寄存器区 (0x01800000)，PLL_GPU0 在 CCU 内统一管理；H700 brief 也未单列 GPU 外部供电脚
    - DRAM 通过 `DRAM_VCC set to 1100 mv`（BOOT0 阶段）单独配置，路径上更可能挂 dcdc4（1.0-3.7 V 范围匹配 LPDDR4 VDD）而非 dcdc2
    - dcdc2/dcdc3 当前电压 (940 mV / 1100 mV) 看上去像两个备用辅助电源域，可能给 SoC 内部 PLL/SRAM 或外部 sensor，但 BSP DTS 没声明 consumer
  - **含义**：mainline porting 时如果照搬"dcdc2=GPU"会写错。需以实测 sysfs + 板级原理图为准；mainline 应只为有真实 consumer 的 rail 建 fixed-regulator/regulator-fixed 节点。

### 16. 电池信息

- **事实**：设计容量 3000 mAh，健康 Good，温度 30.0°C。状态/电压/电流为瞬时值，会随充放电变化（实测 2026-05-20：充电中、4.043V、CHARGE_FULL=3000 mAh、容量 65%、CC=1286 mA）。
- **验证方式**：`ssh root@SP_HOST cat /sys/class/power_supply/axp2202-battery/uevent`

### 17. USB 电源信息

- **事实**：在线，输入电流限制 2500 mA，最小电压 4200 mV
- **验证方式**：`ssh root@SP_HOST cat /sys/class/power_supply/axp2202-usb/uevent`

---

## 四、WiFi/BT 固件文件

### 18. RTL8821CS WiFi/BT 固件清单（设备 rootfs 自带）

- **实际加载情况（2026-05-20 `lsmod` + `dmesg` 实测）**：
  - **现役驱动只有一个**：`8821cs.ko`（Realtek 厂商版 RTW 驱动，版本 `v5.5.1_32758.20190403_COEX20180712-3232`，build time `Aug 30 2024 00:07:18`，WiFi + BT 二合一，**非** mainline `rtl8xxxu` 或 `rtl8821ae`）。`modinfo` 确认 `alias: sdio:c*v024CdC821*`（Realtek vendor 0x024C, device 0xC821 = RTL8821CS）
  - **驱动 init 流程**（两段式，dmesg 实测）：
    1. `[6.077] RTW: module init start` → 打印版本
    2. `[6.213] rtl8821cs: probe of mmc2:0001:1 failed with error -123`（**-ENOMEDIUM**，首次 sdio probe 时 SDIO 卡尚未 enumerate 完毕的非致命中间状态）
    3. `[6.215] RTW: module init ret=0`（模块加载本身成功）
    4. `[6.327] RTW: == SDIO Card Info ==` clock 50 MHz / DDR50 / sd3_bus_mode TRUE → 后续 HW EFUSE dump、HAL 初始化、chplan 0x7F、wlan0/wlan1 起来
  - 这是 RTW 厂商驱动的常见 init 模式：模块 init 不依赖 SDIO probe 成功，待 SDIO 子系统完成卡枚举后再由 `mmc_attach_sdio` 触发真正绑定；error -123 应被理解为"未就绪"而非真失败
  - BT 低功耗辅助：`rtl_btlpm`（独立模块，与 8821cs 配合做电源管理）
  - 厂商驱动 `8821cs` 走自己的 RAM patch / EFUSE 读取流程（dmesg 中可见 `HW EFUSE 0x000-0x1F0` 完整 dump），**不依赖 `/lib/firmware/rtlwifi/` 或 `/lib/firmware/rtl_bt/` 下的 Linux upstream 风格固件**
- **`/lib/firmware/` 下的固件文件**（rootfs 自带，与现役驱动多数无关）：

  | 文件 | 路径 | 实际归属 |
  |------|------|----------|
  | `rtl8821c_fw` / `rtl8821c_config` | `/lib/firmware/rtlbt/` | RTL8821C BT 部分（rtl_btlpm 或厂商 BT 路径可能用到） |
  | `rtl8821aefw.bin` / `rtl8821aefw_wowlan.bin` | `/lib/firmware/rtlwifi/` | **mainline `rtl8821ae` 驱动**用——本机不用，Ubuntu rootfs 自带 |
  | `rtl8723b_fw.bin` / `rtl8723b_config.bin` | `/lib/firmware/rtl_bt/` | RTL8723B 固件，**与 RTL8821CS 无关**——Ubuntu 22.04 标准 firmware-realtek 包带的，纯 rootfs 残留 |
  | `rtl8723bs_*` / `rtl8723bu_*` / `rtl8188*` / `rtl8192*` / `rtl8712u.bin` 等 ~28 个 | `/lib/firmware/rtlwifi/` | 同上，Ubuntu firmware 包默认全装，与本机硬件无关 |

- **验证方式**：
  - `ssh root@SP_HOST lsmod | grep -iE "8821|8723|rtl|btusb|hci"`
  - `ssh root@SP_HOST dmesg | grep -iE "rtl8821|rtlwifi|rtlbt"`
  - `ssh root@SP_HOST ls /lib/firmware/rtlbt/ /lib/firmware/rtlwifi/ /lib/firmware/rtl_bt/`
- **mainline porting 含义**：替换为 mainline `rtw88_8821cs` 或 `rtl8xxxu` 驱动时，需要的固件是 `rtw88/rtl8821c_fw.bin` 系列（不在当前 firmware 目录中），需另行安装。

---

## 五、UART 调试串口

### 19. UART0 走 PH0/PH1

- **事实**：`pin 224 (PH0): uart0 function uart0 group PH0` + `pin 225 (PH1): uart0 function uart0 group PH1`。焊接 PH0/PH1 测试点 = 拿到 debug 串口（115200 baud）
- **验证方式**：
  - 来源：`online_log/sys_kernel_debug_pinctrl_pio_pinmux-pins`
  - 串口 BOOT0/U-Boot 日志中 `ttyS0` console 输出
  - 物理验证：焊接后 USB-TTL 接 GND/TX/RX，minicom 115200

---

## 六、Mali kbase 与 libmali 版本绑定

### 20. 设备 mali_kbase = r20p0-01rel0，libmali = r20p0

- **事实**（设备实测）：
  - `/sys/module/mali_kbase/version` = `r20p0-01rel0 (UK version 11.17)`
  - `/usr/lib/libmali.so.0.20.0`（".20" 即 r20p0；`libmali.so.1` 也软链到此 blob）
  - 文件时间戳：2023-04-07（libmali blob）vs 2024-06-11（软链）
- **版本绑定要求（kbase ↔ libmali UK 协议）**：
  - kbase 通过暴露 `UK version` 数字（实测 11.17）声明它支持的 user-kernel 通信协议版本；libmali 在 dlopen 时通过 ioctl 协商，若版本不匹配则 fail
  - **r18p0 kbase 不能驱动 r20p0 libmali** —— **来源说明**：此断言在 ARM 公开的 Mali GPU driver 文档中**没有正式 release notes 列表**（ARM 历来不公开 Mali kernel driver 的逐版本变更细节，r10-r28 系列只发 source tarball）。结论来自社区/嵌入式厂商经验：rockchip/sunxi BSP 开发者在论坛和 git commit message 中多次报告 r18→r20 之间 `UK version` 跳变导致 libmali ioctl 协商失败。**严格按本文档来源规则归类：此条为"社区经验性约束"，未通过官方文档独立验证；实践上 Anbernic 用 r20p0 kbase 配 r20p0 libmali（实测可工作）即可规避**
- **验证方式**：
  - `ssh root@SP_HOST cat /sys/module/mali_kbase/version`
  - `ssh root@SP_HOST ls -la /usr/lib/libmali*`
- **mainline porting 含义**：换用 mainline `panfrost` 驱动后整个 kbase + libmali 栈被替换为 Panfrost (kernel DRM) + Mesa (userspace OpenGL ES/Vulkan)，UK 协议问题不再存在；但 Panfrost 性能/兼容性是否能匹配 libmali blob 需另行评估

---

## 七、分区表（mmcblk0，14.5 GB TF1 SD 卡）

### 21. 完整分区布局

- **事实**（fdisk -l 实测扇区数，2026-05-20 复核）：

  | 分区 | 用途 | 起始扇区 | 大小 (字节) | 大小 | FS |
  |---|---|---|---|---|---|
  | （前导） | BOOT0 / U-Boot / DTB 等 SPL 区 | 0 – 73727 | 37,748,736 | 36 MiB | raw（dd 烧录） |
  | p1 | Roms | 73728 | 2,147,483,648 | 2 GiB | vfat → /mnt/mmc |
  | p2 | boot-resource | 4,268,032 | 33,554,432 | 32 MiB | vfat（logo / charge png） |
  | p3 | env | 4,333,568 | 16,777,216 | 16 MiB | u-boot env raw |
  | p4 | boot | 4,366,336 | 67,108,864 | 64 MiB | Android boot.img |
  | p5 | rootfs | 4,497,408 | 7,516,192,768 | 7 GiB | ext4（Ubuntu 22.04，root=/dev/mmcblk0p5） |
  | p6 | appfs | 19,177,472 | 4,294,967,296 | 4 GiB | ext4 → /mnt/vendor |
  | p7 | UDISK | 27,566,080 | 1,438,644,736 | 1.34 GiB | ext4 → /mnt/data |

- **容量核对**：分区累计 = 2+0.03125+0.015625+0.0625+7+4+1.34 = **14.45 GiB**；卡总容量 `blockdev --getsize64 /dev/mmcblk0` = 15,552,479,232 B = **14.48 GiB**。差额 ≈ 36 MiB 前导（sunxi 标准 BOOT0/U-Boot raw 区）+ 末尾几 MiB GPT 备份头未分配，数学吻合。
- **验证方式**：`ssh root@SP_HOST fdisk -l /dev/mmcblk0`、`blockdev --getsize64 /dev/mmcblk0p{1..7}`
- **备注**：
  - mmcblk0 是 TF1 卡槽（可拆卸热备份），mmcblk1 是 TF2（实测 59.5 GiB exFAT 用户 ROM 卡），**没有 eMMC**
  - p1 (2 GiB) 内并非游戏 ROM——Anbernic 的"Roms"分区只放系统级 BIOS/字体/语言包；用户 ROM 在 mmcblk1 上
  - 前 36 MiB 是 Allwinner sunxi BSP 的标准布局：BOOT0 在 8KB 偏移开始 + U-Boot/ATF/DTB 紧随其后，详见 boot.img v0 (§22)
  - **路线 C mainline 实测**（2026-05-22）：p2 (boot-resource FAT16) 已被改作 mainline boot 分区（Image.gz/dtb/extlinux.conf），原厂 logo/charge 资源可从 `device-backup/p2-orig.tar` 恢复

### 22. Boot 格式：Android boot image v2

- **事实**：page=2048，name=`sun50i_arm64`，header version **2**（V1.1.5 固件实测，offset 0x2C = `02 00 00 00`；早期固件为 v0）
  - kernel load `0x40080000`（= DRAM 基址 `0x40000000` + ARM64 标准 `TEXT_OFFSET 0x00080000`）
  - ramdisk load `0x42000000`
  - DTB load `0x44000000`
  - tags `0x40000100`
- **U-Boot DRM 密钥**：`keybox_list=hdcpkey,widevine`（Widevine/HDCP 密钥支持，SP 无 HDMI 口，此配置可能为 H700 平台通用模板）
- **验证方式**：
  - `boot 分区前 8 字节 = "ANDROID!"`（已实测，见 §十一）
  - `hexdump -C /dev/mmcblk0p4 | head -5` 读 header：offset 0x08 = kernel_addr, 0x0C = kernel_size, 0x10 = ramdisk_addr, 0x18 = tags_addr, 0x2C = header version
- **路线 C 备注**：p4 仍保留原厂内核作回滚锚点，但 p5 rootfs 已不可逆改动（`/lib/modules/4.9.170/` 已被清理，即使 p4 rollback 原厂 modules 也需从备份恢复）

### 23. DTB 三选一逻辑

- **事实**（设备实测 + boot.img 字符串）：
  - U-Boot 根据 cmdline `lcd_type=<字符串>` 在三份 DTB 中挑选
  - 设备当前 `cat /proc/cmdline | tr ' ' '\n' | grep lcd_type` → `lcd_type=old`，对应选择 dtb 0（`0_1334800.dtb`）
  - boot.img 二进制中可见以下字符串证据（`strings /dev/mmcblk0p4`）：
    - 函数名 `allen_get_lcd_type`
    - 格式串 `===%s:L%d,lcd_type= %d`（U-Boot 阶段在 serial.log 中可见 `===lcd_panel_uboot_fj035fhd05_v1_init:L276,lcd_type = 0`）
    - 格式串 `allen_boe_lcd=%d, str=%s`（Kernel earlycon 在 serial.log 中可见 `allen_boe_lcd=0, str=old`）
    - LCD 驱动名字符串 `fog_fj035fhd05_v1` 与 `rg34xxsp_v1` 同时出现
  - 三份 DTB 内容差异（通过反编译各自 `.dtb` 看 `lcd_driver_name`/`lcd_x` 属性确认）：

    | 文件 | lcd_driver_name | 分辨率 (lcd_x) | 当前实测设备使用情况 |
    |---|---|---|---|
    | `0_1334800.dtb` | `fog_fj035fhd05_v1` | 640 | ✅ `lcd_type=old` 实测选用 |
    | `1_135b800.dtb` | `rg34xxsp_v1` | 720 | 未实测（推测 SP 检测到 RG34XX-SP 屏时选用） |
    | `2_13b8800.dtb` | `fog_fj035fhd05_v1` | 640 | 未实测（时序变体 + USB/电池 wakeup 多了几项） |
- **未确定**：`lcd_type` 取什么字符串值会选 dtb 1 或 dtb 2——选择逻辑在 `allen_get_lcd_type` 函数内（闭源 U-Boot 二进制），设备和官方 PDF 都没给出映射表。当前仅能确认 `lcd_type=old → dtb 0`。
- **验证方式**：
  - `ssh root@SP_HOST cat /proc/cmdline | tr ' ' '\n' | grep lcd_type`
  - `ssh root@SP_HOST strings /dev/mmcblk0p4 | grep -E 'lcd_type|allen.*lcd|fj035fhd05|rg34xxsp'`
  - 反编译各 dtb 看 `lcd_driver_name`/`lcd_x` 属性

---

## 八、运行时观测（串口 / online_log）

### 24. BOOT0 / ATF / U-Boot 版本与 DRAM 训练

- **事实**（2026-05-20 `scripts/serial.log` 重启实测）：
  - **BOOT0**：commit `ac58e911-dirty`，DRAM BOOT DRIVE INFO V0.651
  - **ATF (BL3-1)**：`v1.0(debug):5a77824`，Built `15:20:22, 2024-08-02`，commit 8 ——**注意 `(debug)` 标记**：这是 ATF 的 debug build（启用额外的 register dump、exception backtrace、verbose NOTICE/ERROR 输出），通常生产固件会用 `release` build。Anbernic 在出货固件中保留 debug ATF 对安全评估有意义：例如后续若考虑 secure boot/OPTEE，debug 构建会暴露更多调试信息
  - **U-Boot**：`2018.05 (Dec 25 2025 - 12:00:24 +0800) Allwinner Technology`（与 cmdline `uboot_message=2018.05(12/25/2025-12:00:24)` 一致）
  - **LPDDR4 训练**：两轮（R_2d 与 R_2st 各一轮）read_calibration 各失败 10 次后 fallback 走 retraining，最终 `Dram DST Success` + 32-bit 1 ranks 训练成功
  - **U-Boot 阶段 CPU/总线时钟**（来自同一行 U-Boot 直接 printf：`CPU=1008 MHz,PLL6=600 Mhz,AHB=200 Mhz, APB1=100Mhz  MBus=400Mhz`）：1008 / 600 / 200 / 100 / 400 MHz
- **online_log build 历史值**：BOOT0 `749c1f9a-dirty`（与现役 `ac58e911-dirty` 不同 build）
- **路线 C mainline 实测**（2026-05-22）：`SPL/BL31 v2.14.2` → `U-Boot 2026.04` → `Linux 7.0.9` 完整启动到 login（替换原厂 BOOT0/ATF/U-Boot 链路，SPL 写在 8KB 偏移，与原厂布局兼容）
- **验证方式**：串口抓 BOOT0/ATF/U-Boot 日志（PH0/PH1 UART 115200）；在 `scripts/serial.log` 中搜索锚点 `BOOT0 commit`、`BL3-1`、`U-Boot 2018.05`、`DRAM CLK =`、`Starting kernel ...` 各阶段

### 25. 内核版本字符串

- **事实**：
  - 原厂 #11：`Linux version 4.9.170 (flower@flower-B85M-D2V) (gcc version 5.3.1 20160412 (Linaro GCC 5.3-2016.05)) #11 SMP PREEMPT Wed Dec 24 23:14:11 CST 2025`
  - online_log #241：`Linux version 4.9.170 (cc@cc-H81M-S1) (gcc version 5.3.1 20160412 (Linaro GCC 5.3-2016.05)) #241 SMP PREEMPT Thu Apr 11 12:49:34 CST 2024`
- **验证方式**：`ssh root@SP_HOST uname -a` 或串口 bootlog

### 26. 串口日志中"原厂就有、可忽略"的错误

- **事实**：以下错误在原厂内核正常启动时都出现，**不是自编内核引入的**：

  **BOOT0/SPL 阶段**：
  - `[pmu]: bus read error`（试探多家 PMIC，下面探到 AXP2202 即可）
  - `read_calibration error` × 10 + `retraining final error`（LPDDR4 训练正常重试）

  **ATF/U-Boot 阶段**：
  - `ERROR: tsp_ep_info->pc is NULL`（`secure_os_exist=0` 预期无 secure OS）
  - `partno erro : can't find partition Reserve0` 等（Android 分区表中不存在的分区）
  - `[mmc]: MMC Device 2 not found`（SP 没 eMMC，只有 mmcblk0/1）
  - `pmu_axp152_probe pmic_bus_read fail` / `pmu_axp1530_probe ...`（PMIC 探测序列正常）
  - `uart uart1: get regulator failed`（uart1 接 BT，regulator 缺失只影响 BT）

  **Kernel driver probe 阶段**：
  - `sunxi:i2c_sunxi@twi3[ERR]: get supply failed!`（twi3/twi5 未引出）
  - `axp2101-regulator ... unsupported AXP variant ... Error setting dcdc frequency: -22`（**说明 BSP 同时挂载了 `axp2101-regulator` 与 `axp2202-regulator` 两套驱动**：前者尝试匹配 AXP2101 变体失败，后者真正接管 AXP2202——印证 §10 中"BSP 私有驱动栈与 mainline `axp20x`/AXP717 驱动并非简单等价"。regulator 仍正常工作）
  - `sunxi-wlan ... get gpio wlan_regon/chip_en/power_en failed`（DTS 中 wlan 节点声明了 chip_en/power_en 属性但**没指 GPIO**——只有 `wlan_io_regulator=axp2202-cldo4` 字符串属性。RTW 厂商驱动 `8821cs` 不依赖 sunxi-wlan 的 GPIO 控制路径，它通过 SDIO 总线直接访问 RTL8821CS 自身的 power management，WiFi 仍工作）
  - `VE: sunxi ve debug register driver failed`（仅影响 VE 调试接口，不影响视频解码）
  - `mmc:failed to get gpios`（某 MMC 控制器 GPIO 缺失）
  - `sunxi-mmc sdc1: smc 2 p1 err, cmd 52, RTO !!`（WiFi SDIO 初始化前正常探测）
  - `ERROR: pinctrl_get for HDMI2.0 DDC fail`（SP 无 HDMI 口）
  - `cpu cpu1/2/3: Failed to register opp debugfs (-12)`（-ENOMEM，不阻断 OPP 设置）
  - `e2fsck: /dev/mmcblk0p5 has unsupported feature(s): metadata_csum`（e2fsck 太老，fs 仍正常 mount）
  - `cgroup: cgroup2: unknown option "nsdelegate,memory_recursiveprot"`（4.9 不识别，回退）
  - `proc: unrecognized mount option "hidepid=invisible"`（Linux 5.8+ 选项，4.9 不识别，回退）

- **验证方式**：串口 bootlog 对照（原厂 #11 serial.log + online_log/bootlog）
- **2026-05-20 重启实测对照**（`scripts/serial.log`，原厂 #11 build `ac58e911-dirty`）：
  - 上表所有条目均在本次重启日志中再现。新增观察：
    - PMIC 探测序列中本次日志只看到 2 个 `[pmu]: bus read error` 然后直接 `PMU: AXP2202`，**未列出** `pmu_axp152_probe` / `pmu_axp1530_probe` 具体名字（这些只在更早的 build 或 BSP 详细日志里出现，与功能无关）
    - 新增可忽略错误：
      - `[NAND][NE] Not found valid nand node on dts`（SP 无 NAND）
      - `sunxi-wlan ... get gpio wlan_hostwake failed`（与 wlan_regon/chip_en/power_en 同源）
      - `axp2202_usb_power: axp2202-acin device is not configed, not use vbus-det`（SP 走 USB OTG，无独立 AC 输入）
      - `hci: request ohci0-controller gpio:272` / `hci: request ohci1-controller gpio:147`（USB HCI GPIO 请求信息，非错误）
      - `[asoc_simple_probe, 432]` 反复出现（asoc-simple-card 探测多条声卡，是 print 不是错误）
      - U-Boot 阶段：`FDT ERROR:fdt_get_regulator_name:get property handle twi-supply error:FDT_ERR_INTERNAL`、`[axp][err]: b12_mode: 0`（PMIC 初始化阶段非致命）
      - U-Boot 阶段：`Item0 (Map) magic is bad` / `the secure storage item0 copy0/1 magic is bad`（secure_storage 未初始化，secure_os_exist=0 预期）

---

## 九、DTB 来源确认

### 27. online_log DTS 与本地 DTB 一致

- **事实**：`online_log/sun50i-h700-rg35xx-sp.dts`（6638 行）与本地 `dtb/0_1334800.dts`（6632 行）差异 911 行**全部是 dtc 反编译风格差异**（`"a\0b"` vs `"a","b"`），实质内容完全相同
- **验证方式**：`diff <(sed 's/"\([^"]*\)\\0\([^"]*\)"/"\1","\2"/g' dtb/0_1334800.dts) online_log/sun50i-h700-rg35xx-sp.dts | head`

---

## 十、验证方式不明确的条目

以下条目的验证方式存在不确定性：

### 10.1 GPU 实测为 MP1 而非 MP2 的根因

- **问题**：无法确认是 fuse 掉了第 2 个 core，还是 brief 描述的是上限 SKU
- **现有验证**：`/sys/devices/platform/gpu/gpuinfo` 只能报当前状态，无法区分 fuse vs SKU
- **可能途径**：读 GPU 寄存器 `GPU_ID`（偏移 0xFC8）的 `SCALER_COUNT` 字段，或查 efuse 中 GPU 配置位——但寄存器映射未公开

### 10.2 DRAM 总线宽度（32-bit）的确认

- **2026-05-20 已确认**：见 §5。`scripts/serial.log` 中 `[AUTO DEBUG]32bit,1 ranks training success!`、`Actual DRAM SIZE =1024 M`、`DRAM Type =8 (LPDDR4)` 三个锚点交叉确认 LPDDR4 / 32-bit / 1 GiB。
- **进一步硬件来源**：H700 brief（`2021070513595227.pdf`）明确写 `32-bit DDR4/DDR3/DDR3L/LPDDR3/LPDDR4 interface, supporting maximum capacity of 4GB`——即 H700/H616 die 本身只有 32-bit DRAM PHY，**不存在 64-bit 配成 32-bit 模式**的可能性。H616 User Manual §3.1 内存映射表中 `DRAM 0x40000000-0x13FFFFFFF (4G)` 也确认地址空间上限为 4 GiB（受 32-bit PHY × 32 Gbit 限制）。
- **原问题已封闭**：物理 32-bit。

### 10.3 H700 与 H616 的 die 关系

- **问题**：文档说"几乎同一 die"，差异在 G31 GPU + SmartColor 3.3 + fuse 配置——但无法独立验证 die 内部差异
- **现有验证**：mainline DTS 直接 `#include "sun50i-h616.dtsi"` 而不另写 h700 dtsi，且功能正常
- **2026-05-20 文档补强（H616 Datasheet §2.11 + H700 brief 封装节）**：
  - **H616 package**：TFBGA284 balls, 14×12 mm, 0.65 mm ball pitch
  - **H700 package**：TFBGA 421 balls, 15×15 mm, 0.65 mm ball pitch（多出 137 balls）
  - H700 brief 直接说明"another package variant, **exposing the RGB LCD pins**, and reportedly the NMI pin"
  - 旁证：H616 User Manual §3.1 显示区只有 `TCON_TV0/TV1` + HDMI + TVE，**没有 TCON_LCD**——die 内部应有 LCD 控制器但 H616 package 未引出 pad；H700 通过更大封装将 LCD pin 接出
  - 设备 dt 中存在 `intc-nmi@07010320` 节点（落在 PRCM 区 0x07010000-0x070103FF 内），与 H700 brief 提到 "exposing NMI pin" 一致
- **结论**：H700 = H616 同 die，仅 package + (fuse + ?G31 是否多 1 shader) 不同。同 die 假说有原始文档证据。
- **未解开的子问题**：G31 是否物理同片（H700 brief 写 MP2，设备实测 MP1）——仍无法独立验证 fuse 还是 SKU 差，见 §10.1。

### 10.4 `allen_amp_power` / `allen_boe_lcd` / `[allen]` 标记的确切来源

- **问题**：推断为 Anbernic 内部工程师 "Allen"，但无法确认是个人签名还是方案商/公司代号
- **现有验证**：源码 DTS + bootlog 中多处出现，与其他 Anbernic 内部代码证据（`cc@cc-H81M-S1` build host——见 §10.5 注：本机实测为 `flower@flower-B85M-D2V`，`cc@cc-H81M-S1` 仅出现在互联网公开的他人日志中）交叉印证
- **2026-05-20 实测补充**：
  - 在设备 `/proc/device-tree` 中**未发现**任何 `allen*` 节点或属性
  - 但 `scripts/serial.log`（实际重启日志）中确认 allen 字符串以 **U-Boot/Kernel printk** 形式存在：
    - U-Boot：`--allen--PE_data[31:0] = 0x40,lcd_type = 0`
    - U-Boot：`===allen==: download_normal_boot0 L289`
    - Kernel earlycon：`allen_boe_lcd=0, str=old`
    - Kernel driver：`---[allen] axp20x_pek_probe: 907 wakup_irq=120`
  - 结论：allen 标记是 **C 源码中的 printk/调试串**，不是 DT 节点也不是设备运行时可读属性；与 Anbernic 内部工程师签名假说一致（贯穿 U-Boot LCD 探测、boot0 下载、kernel earlycon LCD 选择、AXP PEK 探测四处）。
- **可能途径**：无公开途径确认

### 10.5 `deeplay` hostname 的确切含义

- **2026-05-20 更新（基于本机串口实测）**：本机 `scripts/serial.log` 中 hostname 实测为 **`ANBERNIC`**（行 2: `Ubuntu 22.04 LTS ANBERNIC ttyS0`），**不出现 `deeplay`**。`deeplay` 字符串只见于互联网公开渠道（GitHub Gist 等）的**其他 H700 设备**串口日志——按本文档来源规则属"互联网搜索来源、默认不可信"，仅记录其存在性，不作为本机事实。
- **互联网渠道推测（未独立验证）**：网上有人将 `deeplay` 与 `Deeplay-Keys` 输入映射 / 菜单中间件关联，但 `Deeplay-Keys` 未找到独立官网或公开信息；该实体与 Anbernic 的确切关系无法确认
- **本机可能途径**：无公开途径独立验证

### 10.6 原厂内核 29 个孤儿符号对应驱动的完整功能

- **问题**：`AXP2202_POWER` / `NVMEM_AXP` / `SUNXI_THERMAL_NG` / `SND_SOC_SUNXI_*`（Audio HUB v2）等 29 个符号在 HandsomeMod 树中完全缺失，无法确认其完整功能边界
- **"29" 的来源**：在 HandsomeMod 4.9 树中执行 `cp config-4.9.170 .config && make oldconfig`，记录所有提示 `(NEW)` 的符号，再交叉过滤掉 mainline 已有但默认 N 的——剩下既不在源码树、也不在 Kconfig 选项中的"完全缺失"项 = 29 个。详细枚举见 `verification-report.md` §1（PMIC 相关、平台/架构相关、WiFi/蓝牙、Audio 等子表）。
- **现有验证**：设备 `config-4.9.170` 中有这些符号，`/sys/` 运行时可见对应平台设备
- **可能途径**：需要 Anbernic 内部 BSP 2025 源码（公网不存在，GPL 违规）

---

## 十一、2026-05-20 ssh sp 验证摘要

通过 `ssh root@SP_HOST`（hostname=ANBERNIC，原厂 #11 内核 build Wed Dec 24 23:14:11 CST 2025）对 §一-§九 共 27 条事实做了实测对照。结果：

### 完全核实（无修改）

#1 CPU 4×Cortex-A53、#2 GPU Mali-G31 MP1 r0p0 0x7093、#3 GPU compatible=`arm,mali-midgard`、
#4 GPU 6 档频率（420/456/504/552/600/648 MHz）、#6 IOMMU 0x030f0000、
#7 LCD `fog_fj035fhd05_v1` lcd_x=0x280(640) lcd_y=0x1e0(480)、
#10 PMIC i2c name=`axp2202`、compatible 含 `x-powers,axp2202`、
#12 cmdline `secure_os_exist=0`、
#17 USB power online=1 / current_limit=2500 / voltage_min_design=4200、
#18 现役驱动 = `8821cs` + `rtl_btlpm`（厂商 RTW，详见 §18 重写说明；/lib/firmware/ 下大部分 rtl* 文件为 Ubuntu rootfs 自带，与现役驱动无关）、
#19 PH0/PH1=uart0 ✅、
#20 mali_kbase r20p0-01rel0 + libmali.so.0.20.0、
#21 mmcblk0 分区表（p1=2G/p2=32M/p3=16M/p4=64M/p5=7G/p6=4G/p7=1.3G）大小完全一致（前 36 MiB 留 BOOT0/U-Boot）、
#22 boot 分区前 8 字节 = "ANDROID!"、
#25 内核版本字符串。

另外确认 mmcblk1（TF2 卡槽）= 59.5 GiB exFAT 卡（用户 ROM 卡），符合 §21 "没有 eMMC、只有 mmcblk0/1" 的备注。

### 已修订条目

- **#8 帧缓冲**：实测 fb0 当前是 1280×1024@16bpp（而非 "640×960×32bpp 双缓冲"）；LCD 实际仍 640×480（取自 `/sys/class/disp/disp/attr/sys`）。fb 虚拟尺寸非稳定事实，受运行时配置影响。
- **#9 音频**：codec 节点 compatible=`allwinner,sunxi-snd-codec`；regulator_summary 显示 `5096000.codec` 同时挂 aldo4(1.8V) + cldo1(3.3V)；运行时 DT 中**无 `allen_amp_power` 属性**。
- **#11 连接性**：实测 `/sys/class/net/` 有 `wlan0` + `wlan1` 两个 netdev（同一 SDIO 设备）。
- **#13 Thermal**：实测 zone 名是 `cpu_thermal_zone`/`gpu_thermal_zone`/`ve_thermal_zone`/`ddr_thermal_zone`/`axp2202-battery`（带 `_thermal_zone` 后缀）。
- **#14 CPU OPP**：实测 **9 档**（480/720/936/1008/1104/1200/1320/1416/1512），不是表中的 8 档（缺 1032/1296，多 936/1008/1320）。原表来源 `online_log/sys_kernel_debug_cpufreq_table` 与现役 #11 内核不一致。
- **#15 PMIC regulator → consumer 映射**：实测 dcdc2/dcdc3 **未挂 consumer**（与原表 "dcdc2=GPU 供电、dcdc3=always-on" 不符）；cldo4 在 regulator framework 上亦未绑定 consumer，仅 DTS 端有 `wlan_io_regulator` 字符串属性。dcdc1=cpu0、aldo1=sdc2、aldo4/cldo1=5096000.codec 三处确认。
- **#16 电池**：状态/电压/电流是瞬时值（实测 4.043V / 充电中 / 65%）；设计容量 3000 mAh、温度 30.0°C、Health=Good 仍正确。
- **§10.4 `allen_*` 标记**：补充实测 `/proc/device-tree` 中无 `allen*` 节点/属性；但 `scripts/serial.log`（实际重启日志）确认 allen 字符串以 U-Boot/Kernel printk 形式存在四处（LCD 探测、boot0 下载、earlycon LCD 选择、AXP PEK 探测），属于源码字符串而非 DT。

### 2026-05-20 串口日志（`scripts/serial.log`）补充

通过实际触发一次 `reboot` 抓到完整的 BOOT0 → ATF → U-Boot → Kernel 启动日志（262 行），对原先标注"需串口"的条目补充确认：

- **#5 DRAM**：从 `[2561]DRAM CLK =672 MHZ` + `[2563]DRAM Type =8 (LPDDR4)` + `[2575]Actual DRAM SIZE =1024 M` + `[351]32bit,1 ranks training success` 四点交叉确认（同时升级 §10.2 状态）。
- **#24 引导链版本**：
  - BOOT0 commit `ac58e911-dirty`（与 #11 build 一致，原 fact 表已记录）
  - **新增 ATF**：`BL3-1: v1.0(debug):5a77824, Built 15:20:22 2024-08-02, commit 8`
  - **新增 U-Boot**：`2018.05 (Dec 25 2025 - 12:00:24 +0800)`
  - DRAM 训练实测两轮 read_calibration 各 ×10 失败后 fallback retraining，最终 `Dram DST Success`
  - U-Boot 阶段 CPU 跑 1008 MHz（同行 U-Boot 输出还报 PLL6=600/AHB=200/APB1=100/MBus=400 MHz）
- **#26 启动错误清单**：上表罗列的 BOOT0/ATF/U-Boot/Kernel 各阶段错误**几乎全部再现**；本次新增可忽略错误条目：`[NAND][NE] Not found valid nand node on dts`、`sunxi-wlan ... get gpio wlan_hostwake failed`、`axp2202_usb_power: axp2202-acin device is not configed`、U-Boot 阶段 `FDT ERROR:fdt_get_regulator_name:get property handle twi-supply error:FDT_ERR_INTERNAL`、`[axp][err]: b12_mode: 0`、`Item0 (Map) magic is bad`（secure_storage 未初始化）。本次未见原表列出的 `pmu_axp152_probe`/`pmu_axp1530_probe` 具体名字——只有 2 个 `[pmu]: bus read error` 然后直接 `PMU: AXP2202`，说明 PMIC 探测序列在不同 build 上行为略异，但功能等价。
- **#8 帧缓冲（新增物理保留信息）**：cmdline `disp_reserve=1228800,0x7bf24800` = 640×480×4 字节单帧 RGBA 缓冲，物理基址 `0x7BF24800`，位于 DRAM 顶部。

### 未在本次验证范围内

#22-#23 boot.img/DTB 内部结构（需 dd 出镜像反编译）、#27 online_log DTS 与本地 DTB 对比（在主机做即可，与设备无关）。这些条目维持原状。

---

## 十二、2026-05-20 H616 PDF 文档补强

将 `doc/*.pdf`（H616 Datasheet V1.0、H616 User Manual V1.0、H700 brief、H616 USB module manual、H616 Android Q OTA Guide）全部转为文本（`pdf-text/*.txt`），针对 device-facts 中"硬件来源"层级的条目做了文档侧确认。

### H700 ↔ H616 die 共用关系（首次拿到 SoC 厂商原始文档证据）

| 项目 | H616 | H700 |
|------|------|------|
| 封装 | TFBGA284 / 14×12 mm / 0.65 mm pitch | TFBGA 421 / 15×15 mm / 0.65 mm pitch |
| 多出的 pin | — | +137 ball，用于引出 RGB LCD pads（die 上本有 LCD 控制器，H616 package 未引出）+ NMI pin |
| GPU | Mali-G31 MP2（datasheet 默认上限） | Mali-G31 MP2（brief 标称，本机实测 MP1，fuse 差异） |
| die 内部其他差异 | — | SmartColor 3.3 picture enhancement engine（brief 强调） |

**关键含义**：mainline kernel `#include "sun50i-h616.dtsi"` 是合理的——die 完全同源，DTS 主要差别仅在 LCD/RGB pinmux 上。

### 设备 `/proc/device-tree` 节点 ↔ H616 User Manual §3.1 内存映射逐项对照

| 设备 DT 节点 | H616 User Manual 基址 | 大小 | 状态 |
|--------------|----------------------|------|------|
| `gpu@0x01800000` | GPU 0x0180 0000 | 256K | ✅（Mali-G31 落在 GPU 槽位） |
| `deinterlace@0x01420000` | DI0 0x0142 0000 | 256K | ✅ |
| `g2d@01480000` | G2D 0x0148 0000 | 256K | ✅ |
| VE（设备未独立暴露） | VE 0x01C0 E000 + VE SRAM 0x01A0 0000 | 8K + 2M | （由 cedrus 驱动用） |
| `interrupt-controller@03020000` | GIC 0x0302 0000 | 64K | ✅ |
| `iommu@030f0000` | IOMMU 0x030F 0000 | 64K | ✅（实测 /proc/iomem 一致） |
| `sunxi-sid@03006000` / `sunxi-sid-ng@03006000` | SID 0x0300 6000 | 4K | ✅ |
| `sunxi-chipid@03006200` | SID 区内偏移（chip id 子节点） | — | ✅ |
| `twi@0x07081400` | S_TWI0 0x0708 1400 | 1K | ✅（PMIC AXP2202 挂这条总线） |
| `codec@0x05096000` | Audio Codec 0x0509 6000 | 4K | ✅ |
| `ehci0/ohci0-controller@0x05101000/05101400` | USB0(OTG) 0x0510 0000 | 1M | ✅ |
| `ehci1/ohci1-controller@0x05200000/05200400` | USB1(Host) 0x0520 0000 | 1M | ✅ |
| `ehci2/ohci2-controller@0x05310000/05310400` | USB2(Host) 0x0531 0000 | 4K | ✅ |
| `ehci3/ohci3-controller@0x05311000/05311400` | USB3(Host) 0x0531 1000 | 4K | ✅ |
| `eth@05020000` / `eth@05030000` | EMAC0 0x0502 0000 / EMAC1 0x0503 0000 | 64K each | ✅（H700 brief 提 2 个 MAC，SP 未引出） |
| `hdmi@06000000` | HDMI_TX0(1.4/2.0) 0x0600 0000 | 1M | ✅（die 上有控制器；SP 物理无 HDMI 口，启动会报 `pinctrl_get for HDMI2.0 DDC fail`） |
| `intc-nmi@07010320` | PRCM 0x0701 0000-0x0701 03FF 区内 | — | ✅（H700 brief 提"exposing NMI pin"） |
| `memory@40000000` | DRAM 0x4000 0000-0x13FFF FFFF | 4 GiB 上限 | ✅（H700 brief 写最大 4GB；实际 1 GiB） |
| THS（设备暴露为 thermal_zone0-3） | THS 0x0507 0400 | 1K | ✅（4 sensor 见 §13 补强） |
| UART0（cmdline `0x05000000`） | UART0 0x0500 0000 | 1K | ✅ |

**所有设备 dt 节点的物理地址与 H616 User Manual 内存映射完全吻合**——首次拿到完整厂商寄存器表对照，可作为后续 mainline porting/调试的权威参考。

### USB drivevbus = "axp_ctrl"（§17 补强）

H616 USB 模块手册 §5.1 明确写：`usb_drv_vbus_gpio = "axp_ctrl"` 表示 USB OTG VBUS 由 AXP PMIC 驱动。设备 dt 中 `pmu/x-powers,drive-vbus-en` + regulator `axp2202-drivevbus` 即此机制的具体实现，与"未引出 VBUS GPIO 而是走 PMIC"的硬件设计一致。
