# RG35XX-SP 路线 C(完全主线)二期问题清单

> 一期完成时间:2026-05-20 22:xx,串口登录成功(`Linux ANBERNIC 7.0.9 #1 SMP PREEMPT aarch64`)。
> 本文件按"阻塞优先级 × 子系统"组织,每项给出 **证据 / 假设原因 / 验证方式**。

---

## P0 — 缺核心功能(影响日常可用)

### P0-1 网络全无:mmc1 没 probe,WiFi RTL8821CS 起不来 — **2026-05-21 已解决**

- **修复路径**:
  1. patch `drivers/mmc/core/pwrseq_simple.c` 容忍 `devm_reset_control_get_optional_shared` 返回 -ENOENT(主线 7.0.9 回归 bug)
  2. defconfig 加 `CONFIG_RTW88_8821CS=m`(+ SDIO/8821C 依赖链)
  3. firmware:`/lib/firmware/rtw88/rtw8821c_fw.bin` from linux-firmware
- **验证**(2026-05-21):`wlan0` 起来,真 MAC `XX:XX:XX:XX:XX:XX` 从 eFuse 读;`wlan0: associated` 已连 AP
- 详见 `内核 git 中` 和 `src/linux-7.0.9/kernel/configs/rg35xxsp.config`

### P0-2 蓝牙 — **2026-05-21 已弃用,不再编译**

> 之前(同日早些时候)标注 "完全解决" 是错误的:只验证到 `hci0 sysfs 节点出现` 和 `BD_ADDR 可读`,没验证业务命令(inquiry/scan)。深入排查后发现 mainline 7.0.9 对 RTL8821CS BT 支持不完整,任何业务操作都会让 chip 自挂,**连带 WiFi 一起挂**(`rtw88_8821cs: failed to send h2c command`)。掌机不依赖 BT,决定直接弃用。

#### 弃用动作(2026-05-21)
- `src/linux-7.0.9/kernel/configs/rg35xxsp.config`:`CONFIG_BT=n`,所有 `BT_HCIUART*` / `BT_RTL` / `BT_BNEP` 注释掉(默认 disable 跟着走)
- 设备 ssh 操作:
  1. `systemctl disable rtl8821cs-bt-rebind.service` + `rm /etc/systemd/system/rtl8821cs-bt-rebind.service`(此 unit 是临时 workaround,见下方"诊断过程"小节)
  2. `rm -rf /lib/firmware/rtl_bt/`(`rtl8821cs_fw.bin` + `rtl8821cs_config.bin` + vendor 遗留 `rtlbt_config`)
  3. `systemctl mask bluetooth.service`

#### 诊断过程(为何弃用,留存便于后续复跑或换 kernel 时参考)

**当前症状(ssh 实测)**:
- boot 后 `/sys/class/bluetooth/` 是空,hci0 不存在
- 手动 `unbind` + `bind` 一次后,hci0 创建,firmware 加载成功(`fw version 0x75b8f098`),DOWN
- 立即 `hciconfig hci0 up`(抢在 udev/bluez 之前)→ `UP RUNNING`,Features/Local Version 都读到
- **但 inquiry / scan / piscan 全部 `Connection timed out`**
- 任何 `down/up` 第二轮 setup → chip 主动 reset(`Peer device has reset`)→ 连带 `rtw88_8821cs h2c failed` **WiFi 也挂**

**根因三层**:
1. **Boot 时序**:RTL8821CS 是 WiFi+BT combo,WiFi 主控 firmware load 完(`[13.5s] rtw88 fw 24.11.0`)之前,BT 子系统硬件层不响应。但 `hci_uart_h5` 在 `[9.4s]` 就 register 触发 probe,H5 sync 期间 chip 不响应(UART tx 持续涨 / rx≈2 / `pe:1 brk:1`),sync 永远不进 `H5_ACTIVE`,`hci_register_dev` 永不调用,hci0 永不出现。
2. **mainline 7.0.9 `hci_h5.c` 不支持 8821CS**:`rtl_bluetooth_of_match[]` 只含 `8822cs/8723bs/8723cs/8723ds`。我们 DTS 写 `realtek,rtl8821cs-bt, realtek,rtl8723bs-bt` fallback 到 8723bs → 用 `h5_data_rtl8723bs` 配置(带 `H5_INFO_WAKEUP_DISABLE`)。8821CS 实际是 8822CS 同代,应该用 8822cs vnd。
3. **缺 `HCI_QUIRK_NON_PERSISTENT_SETUP`**:`btrtl_set_quirks` 没给 8821/8723 系列 set。导致 `hci_uart_close` 不真正关 serdev,chip 状态保留。第二次 `hdev->setup` 重做 `btrtl_initialize + download_firmware` → chip 已 fw loaded 不接受 → 主动 reset → 连 WiFi 一起拉死。

**临时 workaround(测试过,部分可用但仍受根因 3 限制)**:
`/etc/systemd/system/rtl8821cs-bt-rebind.service` 在 wlan0 起来后做 `unbind + bind + hciconfig hci0 up`,boot 后能让 hci0 `UP RUNNING`,但任何业务操作仍会挂 chip → 整体可用性≈0。已删。

**真正修复方向(若未来需要 BT)**:
- A. patch `hci_h5.c` 加 `realtek,rtl8821cs-bt` 到 `rtl_bluetooth_of_match[]`,指向 `h5_data_rtl8822cs`
- B. patch `btrtl_set_quirks` 给 8821CS 系列 set `HCI_QUIRK_NON_PERSISTENT_SETUP`
- C. 排查 firmware load 后 inquiry 不工作的原因(baudrate 1.5M 是否真达到 / config bin 字段)
- D. 升级到 mainline 6.13+ 原生支持 8821CS,或 backport

#### 现状
路线 C 二期不再花精力在 BT 上。`rg35xxsp.config` 已显式 `CONFIG_BT=n`,下次 rebuild kernel BT 整个子系统不编译。

### P0-3 屏 — ✅ **DRM card0 已创建,屏已点亮(2026-05-22)**

> 详细研究/调试/决策过程见 `display-worklog.md`(完整活文档,~700 行)。本条目只保留最终摘要。

**证据**(更新于 2026-05-22):

原假设"~40 行 DTS overlay"是**严重低估**。实际四层全缺:

| 层 | 缺失内容 | 实际工作量 |
|---|---|---|
| 驱动层 | DE33 planes/mixer/TCON 不支持 H616 + `sun50i-h616-tcon-*` 无 driver | Batocera 0002 子 patch 01/02/03/04/07/16 共 6 个 |
| dtsi 层 | `sun50i-h616.dtsi` 完全无 display 节点(display-engine/bus@1000000/mixer/tcon/hdmi/SRAM-C/pinctrl) | Batocera 0002 子 patch 08/17/18 共 3 个 |
| 板级 DTS | panel/backlight/spi_lcd/reg_lcd 节点 + display 管线 enable 全缺 | Batocera 0002 子 patch 19/21 共 2 个 |
| Kconfig | 缺 `DRM_PANEL_NEWVISION_NV3052C`/`SPI_GPIO`/`BACKLIGHT_GPIO` | rg35xxsp.config +3 行 |

**已完成**(commit 序列见 `display-worklog.md §12`):

| 里程碑 | 内容 |
|---|---|
| Batocera 11 子 patch | `git mailsplit` + `git am -3` 全部干净 apply 到 linux-7.0.9 |
| 自家 panel fork | NV3052C 驱动加 `fog_fj035fhd05_v1` 时序(640×480,BSP 实测值)+ compatible `anbernic,rg35xx-sp-panel` |
| Build break #1 修复 | `sun8i_(vi|ui)_layer_init_one` 跨模块调用加 `EXPORT_SYMBOL_GPL` |
| Build break #2 修复 | sun50i_planes 合并到 sun8i-mixer.ko,消除模块循环依赖 |
| 关键 bug 修复 | tcon_tv0 DT 节点缺 `status = "disabled"` → component bind 被阻塞(根因定位:2026-05-22 dmesg debug) |
| 调试设施 | `sun4i_drv.c` 加 7 个 `pr_info` tracepoint + `CONFIG_DRM_DEBUG_DRIVER=y` |
| 最终产物 | `out/{Image.gz,sun50i-h700-anbernic-rg35xx-sp.dtb}` + modules.tar.gz(20 MB) |

**当前状态**([源:实测] dmesg + sysfs):
- `panel_newvision_nv3052c` / `sun4i_drm` / `sun4i_tcon` / `sun8i_tcon_top` / `sun8i_mixer` 全部 probe ✓
- `sun4i_drv_bind` → `component_bind_all` → `drm_dev_register` 完整成功链路 ✓
- `/sys/class/drm/card0` + `card0-Unknown-1` 已创建 ✓
- 背光:GPIO backlight `/sys/class/backlight/backlight/` ✓
- Git tag:`lcd-stage-1-lit`(首屏里程碑,commit `12e89e8e4`)
- **2026-05-22 物理确认**:TTY 登录界面正常显示,SPI init 序列和 BSP 时序均兼容 ✓ **P0-3 完成**

**后续升级**(可选):
- 方案 B:PWM 背光(亮度可调)— Batocera 0005/0006/0007 三个 patch
- 方案 C:HDMI 双路 — Batocera 0002 子 patch 05/06/09/20 四个 patch

**关联文档**:
- `display-worklog.md` — 完整活文档(四层审计/Batocera patch 清单/方案对比/GPIO 走线/LCD 时序/BSP 数据/启动 BSP 决策/component bind 调试/根因分析/进展日志)
- `device-facts.md` §7 — LCD 硬件事实(lcd_type 三选一)
- `mainline-support.md` 第 212/263/292 行 — Panel 驱动状态

### P0-4 GPU — ✅ **已解决(2026-05-23)**

- **原始证据**:`panfrost 1800000.gpu: probe failed with error -110`
- **两个根因**:
  1. IOMMU driver 未编译 → `CONFIG_SUN50I_IOMMU=y`(2026-05-22)
  2. PRCM PPU 电源域驱动未编译 → `CONFIG_SUN50I_H6_PRCM_PPU=y`(2026-05-23)
- **验证**([源:实测] dmesg):
  - `panfrost 1800000.gpu: clock rate = 432000000` ✓
  - `panfrost 1800000.gpu: mali-g31 id 0x7093` ✓
  - `[drm] Initialized panfrost 1.6.0 for 1800000.gpu on minor 0` ✓
  - `/dev/dri/renderD128` 已创建 ✓

### P0-5 cpufreq — ✅ **已解决(2026-05-23 实测)**

- **2026-05-23 实测**([源:实测] ssh sp):`/sys/devices/system/cpu/cpu0/cpufreq/` 全部存在
  - `scaling_available_frequencies`:480000 720000 1008000 1032000 1104000 1200000 1416000 1512000(8 档 OPP)
  - `scaling_governor`:schedutil(默认)
- **已修复**:`rg35xxsp.config` 的 `CONFIG_ARM_ALLWINNER_SUN50I_CPUFREQ_NVMEM=y`(builtin)生效

### P0-6 audio — ✅ **声卡已创建(2026-05-23 实测)**

- **2026-05-23 实测**([源:实测] serial.log):声卡已 probe,且 headphone jack input 已注册
  - `input: H616 Audio Codec Headphone Jack as /devices/platform/soc/5096000.codec/sound/card0/input3`
  - systemd `Save/Restore Sound Card State` 服务正常启动
- **仍待验证**:`aplay -l` 确认用户态可见;实际播放声音
- **已修复**:`rg35xxsp.config` 的 `CONFIG_SND_SUN4I_CODEC=m` + 模块自动加载生效

### P0-7 电池 / USB power — ✅ **已工作(2026-05-23 实测)**

- **2026-05-23 实测**([源:实测] ssh sp):
  - `axp20x-battery`:type=Battery, status=Discharging, capacity=90%, voltage=3.989V, current=430mA ✓
  - `axp20x-usb`:type=USB, present=0, online=0(USB 未插入,正常) ✓
  - `/sys/class/power_supply/axp20x-{battery,usb}/` 全部 sysfs 属性可读
- **关于 DMA mask warning**:dmesg 仍有 `DMA mask not set`(P1-2),但 **不影响功能** — 旧假设"probe 失败"被实测推翻
- **已修复**:`rg35xxsp.config` 的 `CONFIG_AXP20X_POWER=m` + `CONFIG_BATTERY_AXP20X=m` 模块自动加载生效

### P0-8 ~~`poweroff` 后无法用 RESET / 电源键唤醒~~ — **2026-05-20 实测:电源键长按能起,正常硬件行为,降级为文档项 P2-7**

补充(2026-05-21 用户澄清):**RESET 按钮的设计本就是"开机状态下的 SoC 复位",不是"关机状态下的唤醒"**。掌机 RG35XX-SP 关机后必须用电源键的 PWRON 路径唤醒 AXP717,这是正确硬件设计,完全不是 bug。

- **证据**(2026-05-20 实测):`poweroff` 后,按 RESET 按钮无反应,设备无法重启
- **分析**:
  - TF-A v2.14.2 `plat/allwinner/sun50i_h616/sunxi_power.c` 有 AXP717 支持(`x-powers,axp717` 检测 + I2C 路径 + `axp_setbits(0x27, BIT(0))` 关机命令),所以 PSCI SYSTEM_OFF 本身**应该**走通
  - 但 RESET 按钮通常只复位 SoC,**不能唤醒 AXP717**。AXP717 进 standby 后,wake source 应为 PWRON pin(电源键)或 vbus 上电
  - 待确认:RG35XX-SP 电源按键 GPIO 是否真接到 AXP717 PWRON,还是仅作为 GPIO 给 kernel 用(若后者,kernel 关了就死锁)
- **当前救援**:长按电源 ≥ 10s + 插 USB-C;最终拆机断电池 JST 5s 再插
- **一期临时绕开**:**避免用 `poweroff`,用 `reboot`**(走 PSCI SYSTEM_RESET → watchdog,不动 PMIC)
- **二期工作**:
  1. 用万用表/原理图确认电源键 → AXP717 PWRON 走线
  2. 若没接 PWRON,kernel 实现电源键 long-press → 写 AXP717 0x27(避开 TF-A 关机路径)
  3. 或加 RTC alarm wake / vbus wake 作 fallback

---

## P1 — 可见 warning/error(不阻塞登录,但要清理)

### P1-1 ~~`module (missing .modinfo section)` 2 次~~ — **2026-05-21 已解决**

- **根因**:p5 上残留两个 BSP 4.9.170 的 vermagic-mismatch 模块
  - `/lib/modules/mali_kbase.ko`(17.5 MB,Mali Bifrost r20p0-01rel0,udev 按 `arm,mali-midgard` alias 自动尝试加载)
  - `/etc/modules` 显式加载 `rtl_btlpm`(vendor BT 低功耗 helper)
- **修复**(2026-05-21 ssh 操作):
  1. `rm /lib/modules/mali_kbase.ko`
  2. `rm -rf /lib/modules/4.9.170/`(11 MB,整个 BSP 模块树)
  3. `sed -i 's/^rtl_btlpm/#rtl_btlpm/' /etc/modules`
  4. 清理 `/lib/firmware/rtl_bt/` 冗余 vendor blob:删 `rtl8723b_*`(2 个,8723B chip 用,我们没有)+ `rtlbt_fw` / `rtlbt_fw_new`(已被 mainline `rtl8821cs_fw.bin` 替代)。**保留 `rtlbt_config`** 作为 vendor MAC config 备份
- **验证**(2026-05-21 重启后):`dmesg | grep "missing .modinfo"` 空,WiFi/BT/cpufreq/audio/battery 全保留
- **释放空间**:~30 MB

### P1-2 AXP 子驱动 `DMA mask not set` warnings

- **证据**:
  - `platform axp717-adc: DMA mask not set`
  - `platform axp20x-usb-power-supply: DMA mask not set`
  - `platform axp20x-battery-power-supply: DMA mask not set`
- **假设原因**:mainline 这几个 platform device 没在 driver init 里调 `dma_coerce_mask_and_coherent()`,但实际不走 DMA。属上游小毛病,不影响功能。
- **验证方式**:确认电池 SOC / USB power 在 `/sys/class/power_supply/` 下有读数。

### P1-3 `pwrseq_simple` probe 失败 `-ENOENT` — ✅ **已解决(2026-05-23 实测)**

- **验证**(2026-05-23 serial.log):pwrseq_simple error 不再出现。随 P0-1 WiFi 修复自动消失。

### P1-4 `Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1` — ✅ **已解决(2026-05-23 实测)**

- **修复**:U-Boot defconfig 加 `CONFIG_ENV_IS_NOWHERE=y` + `# CONFIG_ENV_IS_IN_FAT is not set`
- **验证**(2026-05-23 serial.log):启动日志中 `Unable to read "uboot.env"` 已消失

### P1-5 binfmt_misc — ✅ **修复入队(2026-05-23)**

- **根因**:`CONFIG_BINFMT_MISC` 未设置
- **修复**:`rg35xxsp.config` 加 `CONFIG_BINFMT_MISC=y`
- **验证**(下次 rebuild):`ls /proc/sys/fs/binfmt_misc/` 应存在;`systemctl status systemd-binfmt.service` 应正常

### P1-6 SO_PASSSEC — ✅ **修复入队(2026-05-23)**

- **根因**:`CONFIG_SECURITY_NETWORK` 未设置,`net/core/sock.c` 中 `SO_PASSSEC` 被 `IS_ENABLED(CONFIG_SECURITY_NETWORK)` gate
- **修复**:`rg35xxsp.config` 加 `CONFIG_SECURITY_NETWORK=y`
- **验证**(下次 rebuild):`dmesg | grep "SO_PASSSEC failed"` 应消失

### P1-7 `aldo3: disabling` — ✅ **非 bug,已关闭(2026-05-23)**

- **分析**:DTS 中 `reg_aldo3` 明确标注 `/* 1.8v - unused */`(sun50i-h700-anbernic-rg35xx-2024.dts:341)。aldo3 在此板上无消费者,regulator 框架自动关闭以省电——**这是正确行为**,不是 bug。
- **决定**:关闭,无需修复

### P1-8 `vcc-pl not found, using dummy regulator`

- **证据**:`sun50i-h616-r-pinctrl 7022000.pinctrl: supply vcc-pl not found, using dummy regulator`
- **尝试修复失败**(2026-06-06): 启用 `reg_aldo1` + 添加 `&r_pio { vcc-pl-supply = <&reg_aldo1>; };` → 导致 17 个 pinctrl "Error applying setting" → MMC 全部 probe 失败 → 启动挂死。已回滚。
- **原因推测**: ALDO1 可能实际给其他关键供电域供电,不能随便启用;或需确认 ALDO1 在此板的真实走线
- **影响**: dummy regulator 够用,PL 引脚(RSB/I2C)功能正常。暂不修复。

### P1-9 `usb_phy_generic: dummy supplies not allowed for exclusive requests (id=vbus)`

- **证据**:`[ 1.313] usb_phy_generic usb_phy_generic.1.auto: dummy supplies not allowed for exclusive requests (id=vbus)`
- **原因**:USB OTG vbus regulator 在 DTS 没 wire(原厂 BSP 是 AXP `drivevbus` GPIO 控制 vbus,见 `device-facts.md §17`,mainline DTS 可能未对接)。
- **影响**:USB host 模式可能没 vbus → 插 U 盘可能不上电。一期不要 USB,记下。

### P1-10 DT dependency cycle warnings(CCU ↔ RTC)

- **证据**:`/soc/clock@3001000: Fixed dependency cycle(s) with /soc/rtc@7000000`(及 7010000 互相,共 12 行)
- **原因**:mainline sun50i-h616.dtsi 中 `clock@3001000` (CCU) 与 `rtc@7000000` (sun6i-rtc) 和 `clock@7010000` (R-CCU) 之间引用环。kernel `Fixed` 自动解,但上游可以更整洁。
- **影响**:无功能影响,只是 dmesg 噪声。

### P1-11 `phy phy-5100400.phy.0: Changing dr_mode to 1`

- **证据**:`[ 1.309] phy phy-5100400.phy.0: Changing dr_mode to 1`(切到 OTG/host)
- **原因**:DTS 默认 `dr_mode = "otg"` 被运行时改成 host。注意一致性。
- **配对问题**:跟 P1-9 是一对——dr_mode 切了,但 vbus regulator 没接。

### P1-12 ~~用户态有"每 30 秒 drop_caches"后台脚本~~ — ✅ **已解决(2026-05-23 rebuild rootfs 时自动消失)**

- **原始证据**:`[ 32.327] sh (336): drop_caches: 3`...,每 30 秒一次
- **根因**:BSP 原厂 rootfs 有 systemd service 定期 `echo 3 > /proc/sys/vm/drop_caches`
- **修复**:`rootfs_mk.sh` 重建 rootfs 后该 service 不再存在,dmesg 无 `drop_caches` 输出,`systemctl list-timers` 无相关条目

### P1-13 KASLR 未启用 — ❌ **Won't fix(2026-05-23)**

- **分析**:需为 U-Boot 写新的 sunxi CE TRNG 驱动(H616 Cortex-A53 无 ARMv8.5 RNDR),工作量很大。掌机场景安全性收益极小。
- **决定**:降级为已知限制,不修复。

### P1-16 `sun4i-usb-phy 5100400.phy: failed to get reset usb0_reset` — ✅ **已修复(2026-05-23 实测)**

- **验证**(2026-05-23 serial.log):`failed to get reset usb0` 不再出现。commit `6a291bb29` 生效。

### P1-17 `dw-apb-uart ...: Error applying setting, reverse things back` — **已知 cosmetic,暂不修复(2026-05-23)**

- **证据**([源:实测] serial.log):uart0/uart1 两处报错,pinctrl 框架回退
- **分析**:sunxi pinctrl 驱动不支持 DW UART 请求的某个 generic pin config 参数。pinmux-pins debug 确认 PH0/PH1 correctly muxed to uart0, PG6-9 correctly muxed to uart1。UART 功能完全正常(ttyS0 console, ttyS1 注册)
- **影响**:纯 cosmetic,无功能影响
- **决定**:暂不修复。需上游 `drivers/pinctrl/sunxi/pinctrl-sunxi.c` 增加对缺失 pin config 参数的支持

### P1-18 `cfg80211: loaded regulatory.db is malformed` — **已知,接受(2026-05-23)**

- **尝试修复失败**:关闭 `CONFIG_CFG80211_REQUIRE_SIGNED_REGDB` 会导致 `verify_pkcs7_signature` 符号从内核消失,cfg80211 模块编译失败
- **原因**:Kconfig 有 `default y` 且无 prompt(除非 `CFG80211_CERTIFICATION_ONUS=y`),无法通过 rg35xxsp.config 关闭
- **影响**:WiFi 关联正常(`wlan0: associated`),仅使用 world regulatory domain(可能限制发射功率/信道)
- **决定**:接受此 warning,不修复。对掌机 WiFi 使用无实质影响

### P1-19 `gpiochip*: Static allocation of GPIO base is deprecated` (×4) — **2026-05-23 新发现**

- **证据**([源:实测] serial.log L223/224/243/245):
  - `gpio gpiochip0: Static allocation of GPIO base is deprecated, use dynamic allocation.` ×3
  - `gpio gpiochip1: Static allocation of GPIO base is deprecated, use dynamic allocation.` ×1
- **原因**:内核 GPIO 子系统已废弃静态 GPIO base 分配,但 H616 pinctrl 驱动(`sun50i-h616-pinctrl`/`sun50i-h616-r-pinctrl`)仍在使用旧 API
- **影响**:纯 cosmetic,无功能影响
- **修复**:上游 `drivers/pinctrl/sunxi/pinctrl-sunxi.c` 需改为动态分配。非阻塞,等上游合入

### P1-20 TF2 卡槽(第二 SD 卡)未识别 — ✅ **已解决(2026-05-23 实测)**

- **修复**:SP DTS 加 `reg_tf2_vdd` (PE4 GPIO 电源) + `&mmc2` enable (PE22 CD, reg_cldo2 vqmmc)
- **验证**([源:实测]):`mmcblk2` 59.5 GB SDXC 卡识别,`lsblk` 可见 `mmcblk2p1`

### P1-14 模拟摇杆 — ❌ **不适用(2026-05-23)**

- **用户确认**:RG35XX-SP 无模拟摇杆,只有 D-Pad + ABXY 按键。此问题不适用。

### P1-15 电源按键无 KEY_POWER event — ✅ **已解决(2026-05-23 实测)**

- **2026-05-23 实测**([源:实测] serial.log):`input: axp20x-pek as .../input4` — power key 已注册
- **原因**:`rg35xxsp.config` 的 `CONFIG_INPUT_AXP20X_PEK=m` 已生效,模块自动加载
- **验证**:`evtest /dev/input/event4` 或按电源键看 dmesg 有 KEY_POWER event

---

## P2 — 项目卫生 / 文档 / 流程

### P2-1 ~~README 缺路线 C 完整章节~~ — ✅ **已解决(2026-06-06)**

- 路线 C 已单列为推荐路线
- 目录结构更新: `scripts/` 替代 `build/c/`, 脚本名已同步
- Mainline 支持表已更新(Display/VPU 标 ✅, BT 标 ⚠️ 弃用)

### P2-2 ~~device-facts 应补充 mainline 实测条目~~ — ✅ **已解决(2026-06-06)**

- §24: 已补充 `SPL/BL31 v2.14.2 → U-Boot 2026.04 → Linux 7.0.9` mainline 实测
- §22: 已补充 p4 回滚锚点备注 + p5 modules 已不可逆
- §21: 已补充 p2 改作 mainline boot 分区

### P2-3 ~~P5 模块清理~~ — ✅ **已解决(2026-06-06)**

- **修复**: `rootfs_update.sh` 清理阶段自动删除旧 BSP 4.9.x 模块目录 + `mali_kbase.ko`
- 见 P1-1（2026-05-21 手动清理过设备，现纳入脚本自动化）

### P2-4 `flash_sd.sh` 补 `--uboot-only` / `--p2-only` 子模式

- 调试 U-Boot 改动时,目前要么手动 dd,要么跑全 flow 浪费时间(每次都把 modules.tar.gz 重解一次,20 MB I/O)。增量模式:
  - `--uboot-only`:只 dd `u-boot-sunxi-with-spl.bin` 到 sector 16
  - `--p2-only`:只重写 p2(摆 Image.gz / dtb / extlinux)
  - `--modules-only`:只装 modules 到 p5

### P2-5 备份策略漏洞

- 当前 `device-backup/` 在 `out/` 下,首次执行 `flash_sd.sh` 才生成。
- 一旦误删 `device-backup/`,后续 `--rollback` 不可用且无法重新生成(原始数据已被覆盖)。
- 建议:把 `device-backup/prehead-36m.bin` + `p2-orig.tar` **拷一份到仓库版本控制位置**(如 `device/c-route-baseline/`)或 git-annex。

### P2-6 ~~RTC 双时源问题~~ — ✅ **已解决(2026-06-06)**

- **根因**: sun6i-rtc(rtc0,无电池) vs PCF8563(rtc1,有纽扣电池),默认 hwclock 写 rtc0 → 掉电归零
- **修复**:
  1. udev 规则 `/etc/udev/rules.d/99-rtc-pcf8563.rules`: `SYMLINK+="rtc"` 让 `/dev/rtc → rtc1`
  2. `/etc/adjtime` 指定 `/dev/rtc1`
- **验证**: `/dev/rtc → rtc1` ✓, `hwclock --systohc` 写入 rtc1 ✓
- 已写入 `rootfs_config.sh` 自动部署

---

### P1-21 H616 Cedrus VPU 硬解 — ✅ **HEVC + H.264 均已通(2026-06-04)**

- **目标**:启用 H616 VPU (video-codec@1c0e000) 实现 H.264/H.265 硬解
- **内核层**(2026-05-24 完成):
  - Batocera patch `012-arm64-dts-Add-VPU-node` + `033-cedrus-add-H616-variant` 已合入 dtsi 和 cedrus.c
  - `CONFIG_VIDEO_SUNXI=y` + `CONFIG_VIDEO_SUNXI_CEDRUS=m` 已启用
  - `sunxi-cedrus.ko` 编译成功 (399 KB)
  - SRAM-C1 补丁解决 `Failed to claim SRAM` 问题
  - dmesg: `cedrus 1c0e000.video-codec: Device registered as /dev/video0`
- **DMA-buf heaps**(2026-06-02 修复):
  - 缺 `CONFIG_DMABUF_HEAPS=y/CMA/SYSTEM` → FFmpeg 回退 MMap → cedrus capture queue 不支持 → segfault
  - 已加入 `rg35xxsp.config`,内核已重编部署
- **FFmpeg v4l2-request HEVC segfault 修复**(2026-06-02):
  - RPi 补丁 `ffmpeg-7.1.2-rpi_28.patch` 的 `offsets_add()` 函数有 bug:
    ```c
    void * p2;
    // ...
    av_reallocp_array(&rd->offsets, n2, sizeof(*rd->offsets));  // 已更新 rd->offsets
    rd->offsets = p2;  // ← BUG: p2 未初始化,覆盖刚分配的指针 → 野指针 → segfault
    ```
  - 修复:PKGBUILD 中 `sed -i '/void \* p2;/d;/rd->offsets = p2;/d'` 删掉两行
  - 已加 `options=(!strip)` + `-g` 保留调试符号
- **HEVC 硬解验证**(2026-06-02):
  - `mpv --vo=gpu --gpu-context=drm --hwdec=auto` → `Using hardware decoding (drm)` + `drm_prime[nv12]`
  - CPU 37%(零拷贝),vs `--vo=drm` drm-copy 模式 120%
  - 720p30 视频轻微软解可播放,少量掉帧(vsync/缩放相关,待优化)
- **rg35xxsp.config 改进**:
  - `build_kernel.sh` 改为每次构建都合并 `rg35xxsp.config`(原逻辑 `.config` 存在就跳过)
- **H.264 v4l2-request 硬解**(2026-06-04 完成):
  - 从 RPi HEVC 实现派生,完整实现 H.264 V4L2-request hwaccel (~1200 行)
  - 关键修复:VLD bitstream 起始位置(skip start code + NAL header)、DPB reference_ts ×1000 纳秒转换、SPS/PPS flags 补齐、slice_type/qp_delta/deblock 修正
  - 性能:720p30 全片 7066 帧,硬件 224fps/7.47x vs 软件 194fps/6.45x,CPU 占用从 88.5% 降至 34.8%(2.5 倍)
  - 前 8 帧 framemd5 硬件/软件完全一致(MATCH)
  - 详见 `v4l2-request-h264.md`
- **Cedrus 支持但 FFmpeg 未支持的编解码器**:
  | 编解码器 | Cedrus H616 | FFmpeg V4L2-request | 内核 UAPI 控件 |
  |---|---|---|---|
  | MPEG-2 | ✅ | ❌ 缺失 | `V4L2_CID_STATELESS_MPEG2_SEQUENCE/PICTURE/QUANTISATION` |
  | VP8 | ✅ | ❌ 缺失 | `V4L2_CID_STATELESS_VP8_FRAME` |
  - MPEG-2:老游戏视频/DVD 等复古内容,内核控件最简单(3 个),实现量最小
  - VP8:WebM 视频/部分模拟器场景,内核只有 1 个控件但结构体较大
  - 两者内核 UAPI + cedrus 驱动均已就绪,只差 FFmpeg 端 hwaccel 对接
- **待完成**:
  1. 掉帧优化 — 可能需要调显示刷新率或 vsync 策略
  2. mpv 启动参数优化 — `--vo=gpu --gpu-context=drm --hwdec=auto` 写入 menu.conf
  3. MPEG-2 / VP8 v4l2-request hwaccel(可选,优先级较低)

---

### P1-22 音量按键不调节 ALSA 音量 — ✅ **已修复(2026-05-24)**

- **现象**: `gpio-keys-volume` 已注册,但按音量键不会改变实际播放音量。
- **根因**: 当前路线 C 是最小 console/systemd rootfs,没有桌面/session 音量守护进程消费 `KEY_VOLUMEUP/DOWN` 事件。
- **设备状态**:
  - `/proc/bus/input/devices`: `gpio-keys-volume` 注册为 `event1`,带 `kbd` handler
  - ALSA mixer:可调输出控件为 `DAC`(0-63)和 `Line Out`(0-31);当前使用 `DAC` 做用户音量
- **修复**:
  - 新增 `/usr/local/bin/volume-buttond`:自动查找名为 `gpio-keys-volume` 的 event 节点,用 `evtest` 监听 `KEY_VOLUMEUP/DOWN`,调用 `amixer -c 0 set DAC 3%+/- unmute`
  - 新增 `volume-buttond.service`,已在设备 `sp` 上 `enable --now`
  - 已同步写入 `rootfs_mk.sh`,后续重建 rootfs 会自动带上该服务
- **验证**:
  - `systemctl status volume-buttond.service`:active/running
  - `ps`:服务正在运行 `evtest /dev/input/event1`
  - 待人工按键确认:按音量 +/- 后 `amixer -c 0 sget DAC` 应变化

---

## 待调查 — 不确定但应该早看

### I-1 USB-OTG / Host 模式 — **已查(2026-05-20)**

- 结果:`/sys/class/udc/*/state = not attached`,只有 EHCI/OHCI 自带 root hub。**当前没插任何 USB 设备**,无法判断 host 模式实际是否工作。
- 后续:插 U 盘试一次。若不上电 → 确认 P1-9 vbus regulator 缺失。

### I-2 输入设备 — **已查(2026-05-20)**

- 结果:3 个 input(gamepad/volume/lid),全工作。摇杆 / 电源键 缺失 → 升级为 P1-14 / P1-15。

### I-3 cpufreq — **已查 → 升 P0-5**

### I-4 audio — **已查 → 升 P0-6**

### I-5 电池 — **已查 → 升 P0-7**

### I-6 thermal — **已查(2026-05-20)**

- 结果:4 zone(gpu/ve/cpu/ddr)全在线,温度 43°C ±1。**这一项 mainline 工作正常**,与 device-facts §13 完全一致(只缺 axp battery zone,因为 P0-7 没起来)。
- 无后续。

### I-7 mmcblk1 — **已查 → 已并入 P0-1**

- 设备只有 mmcblk0,mmc1 完全没初始化。U-Boot 看到 `mmc@4021000`,kernel 没。归到 P0-1。

---

## 一期完成确认

| 子系统 | 状态 |
|---|---|
| TF-A v2.14.2 → U-Boot 2026.04 → Linux 7.0.9 全链路 | ✅ |
| earlycon + serial console ttyS0 | ✅ |
| systemd init + getty | ✅ |
| EXT4 rootfs mount (mmcblk0p5) | ✅ |
| AXP717 PMIC probe | ✅ |
| 4 核 SMP + PSCI(无路线 A 的 RCU stall) | ✅ |
| root 登录、shell 可用 | ✅ |

**已完成 P0 (7/7)**:P0-1(WiFi)✅ P0-2(BT/弃用)✅ P0-3(屏)✅ P0-4(GPU)✅ P0-5(cpufreq)✅ P0-6(audio)✅ P0-7(电池)✅

**P1 已修复 (11 项)**:P1-1(.modinfo)✅ P1-3(pwrseq)✅ P1-4(uboot.env)✅ P1-5(binfmt)✅ P1-6(SO_PASSSEC)✅ P1-7(aldo3)✅ P1-12(drop_caches)✅ P1-16(USB PHY)✅ P1-20(TF2)✅ P1-21(Cedrus VPU)✅ P1-22(volume keys)✅

**P1 cosmetic (7 项)**:P1-2/8/9/11/17/18/19 👁️ (P1-10 已被 fw_devlink=off 抑制)

**P1 won't fix (2 项)**:P1-13(KASLR)❌ P1-14(摇杆/SP无此硬件)❌

**P2 已解决 (6/6)**:P2-1(README)✅ P2-2(device-facts)✅ P2-3(模块清理)✅ P2-4(--incremental)✅ P2-5(备份)✅ P2-6(RTC)✅

**本期新增成就**:
- USB gadget CDC ECM (10.1.1.3, systemd 持久化, SSH 双通道)
- TF2 卡槽 (PE4/PE22, 59.5 GB SDXC)
- 音频默认设备修复 (asound.conf audiocodec→Codec)
- 音量按键直控 ALSA DAC 音量(volume-buttond)
- SSH config USB/WiFi 自动切换
- flash_sd.sh / deploy_ssh.sh 均改为 rsync 增量
