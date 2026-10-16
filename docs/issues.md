# RG35XX-SP 主线问题清单与历史记录

> 一期完成时间:2026-05-20 22:xx,串口登录成功(`Linux DEVICE 7.0.9 #1 SMP PREEMPT aarch64`)。
> 本文件按"阻塞优先级 × 子系统"组织,每项给出 **证据 / 假设原因 / 验证方式**。

---

## 当前待办与验收范围（2026-10-11）

当前源码和设备均为 Linux 7.2.9。下文保留日期明确的历史修复过程；“实现完成”“人工验收待补”“已接受限制”分别记录，不再用总完成率代替验收范围。文档整理后另完成启动警告清理、部署和整机重启复验，见[清理记录](boot-warning-cleanup.md)。

| 项目 | 当前状态 | 下一步 / 证据 |
|---|---|---|
| LCD 点亮、左右边缘回绕 | **已关闭** | 7.2.9 原生路由修复，重启和用户视觉验收通过；P0-3 |
| LCD 测试与 menu 交接 | **已验收** | 三平面、64 个格式组合，Start 返回 menu；[显示记录](display-worklog.md) |
| LCD console 日志 | **已处理，重启复验通过** | fbcon bind=0，menu stdout 为 journal socket；[主线指南](mainline-guide.md#lcd-控制台输出2026-10-11) |
| H.264 640×480 P 帧错误 | **历史未解决，待 7.2.9 复测** | 720p 成功不能覆盖全部分辨率；[解码记录](v4l2-request-h264.md#38-mpv-2026-06-04) |
| 视频 DMA-BUF 导入、掉帧和同步 | **历史待调查 / 优化** | 720×540 pitch 导入失败及部分播放问题未记录最终修复；[H.264 记录](v4l2-request-h264.md) |
| DE33 NV12 直扫、硬件缩放 | **当前能力缺口** | 7.2.9 未公布 YUV，scaler_mask=0；六项缩放组合返回 ERANGE。旧实验补丁不代表当前正式驱动能力 |
| USB Host / OTG | **待适配与实机验收** | USB gadget/SSH 已工作；Host 外设供电与 PHY 配置仍待验证；P1-9、I-1 |
| 音频、音量按键 | **驱动 / 服务已实现，人工验收记录待补** | ALSA 枚举与服务运行有记录；本清单未保存 7.2.9 听音及实体音量键改变音量的验收结果；P0-6、P1-22 |
| 刷卡分项更新 | **公开流程待完善** | 当前公开 BSP 提供构建产物，部署流程需按介质布局准备；P2-4 |
| 刷写自动备份与回退 | **流程缺口** | 尚无公开通用自动流程；P2-5 |
| PWM 调光、MPEG-2 / VP8 request 硬解 | **可选增强** | 不阻塞当前 RGB 显示和已验证的 H.264/HEVC 路径 |
| 启动 warnings | **已复核并清理可修项** | 7.2.9 #2 重启后由 16 条降为 5 条；GPIO/DMA/写保护/networkd 已修，剩余限制分类保留；[清理记录](boot-warning-cleanup.md) |
| 蓝牙 / KASLR / 模拟摇杆 | **弃用 / 不修 / 不适用** | 保留原决定，不列入待修列表 |

## P0 — 缺核心功能(影响日常可用)

### P0-1 网络全无:mmc1 没 probe,WiFi RTL8821CS 起不来 — **2026-05-21 已解决**

- **修复路径**:
  1. patch `drivers/mmc/core/pwrseq_simple.c` 容忍 `devm_reset_control_get_optional_shared` 返回 -ENOENT(主线 7.0.9 回归 bug)
  2. defconfig 加 `CONFIG_RTW88_8821CS=m`(+ SDIO/8821C 依赖链)
  3. firmware:`/lib/firmware/rtw88/rtw8821c_fw.bin` from linux-firmware
- **验证**(2026-05-21):`wlan0` 起来,真 MAC `XX:XX:XX:XX:XX:XX` 从 eFuse 读;`wlan0: associated` 已连 AP
- 配置见公开 BSP 的 `configs/rg35xxsp.config`

### P0-2 蓝牙 — **2026-05-21 已弃用,不再编译**

> 之前(同日早些时候)标注 "完全解决" 是错误的:只验证到 `hci0 sysfs 节点出现` 和 `BD_ADDR 可读`,没验证业务命令(inquiry/scan)。深入排查后发现 mainline 7.0.9 对 RTL8821CS BT 支持不完整,任何业务操作都会让 chip 自挂,**连带 WiFi 一起挂**(`rtw88_8821cs: failed to send h2c command`)。掌机不依赖 BT,决定直接弃用。

#### 弃用动作(2026-05-21)
- `configs/rg35xxsp.config`:`CONFIG_BT=n`,所有 `BT_HCIUART*` / `BT_RTL` / `BT_BNEP` 注释掉(默认 disable 跟着走)
- 设备 ssh 操作:
  1. `systemctl disable rtl8821cs-bt-rebind.service` + `rm /etc/systemd/system/rtl8821cs-bt-rebind.service`(此 unit 是临时 workaround,见下方"诊断过程"小节)
  2. `rm -rf /lib/firmware/rtl_bt/`(`rtl8821cs_fw.bin` + `rtl8821cs_config.bin` + vendor 遗留 `rtlbt_config`)
  3. `systemctl mask bluetooth.service`

#### 诊断过程(为何弃用,留存便于后续复跑或换 kernel 时参考)

**2026-05-21 诊断症状（历史实测，蓝牙现已禁用）**:
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
- D. 若未来恢复蓝牙，核对目标内核的 RTL8821CS 支持与业务命令；旧建议“升级到 6.13+”不适用于已运行 7.2.9 的设备，也不是经验证的修复。

#### 现状
路线 C 二期不再花精力在 BT 上。`rg35xxsp.config` 已显式 `CONFIG_BT=n`,下次 rebuild kernel BT 整个子系统不编译。

### P0-3 屏 — ✅ **已关闭：点亮及边缘回绕修复验收通过（2026-10-10）**

**2026-10-10 收尾：根因确认、上游修复、7.2.9 部署与重启视觉验收均已完成，LCD 边缘回绕问题关闭。**

- TCON TOP 两路 mixer 均选中 TCON0（`0x0651001c=0`）触发约 1 像素回绕。仅分离路由为 `0x20` 即恢复，往返对照可复现。
- 正常测试图最终验收通过：最左红、最右白，内侧左绿右蓝。上游修复 `62bac3202042a8cfd4d4969a6d057cebd8b7cc1a` 已含于 `v7.3-rc1`；该缺陷在项目 init commit 中已存在。
- 2026-10-04 经用户随后授权，已替换当时设备正式 `sun8i_tcon_top.ko` 并备份原模块；实际重启后路由自动为 `0x20`，未加载临时模块，2026-10-04 用户确认重启后“一切正常”，视觉验收通过。当时保留 **7.2.3 base** 并计划待 7.3 正式版再迁移；随后按用户决定先迁移至 7.2.9。旧 7.2.3 release 模块包未同步修复，勿用于回退覆盖。
- 2026-10-10：已从 7.2.9 vanilla 新仓库移植 23 个适配提交，原生保留 LCD 路由修复；Image、DTB 与 1633 个模块完整构建通过，已部署并重启到 `7.2.9 #1`。原生路由 `0x20`，未加载临时模块，用户确认正常测试图无边缘回绕。7.2.3 启动文件已备份，旧模块树保留；见 [部署日志](display-worklog.md#deploy-7-2-9-20261010)。
- 详见 [LCD 工作记录](display-worklog.md#lcd-route-fix-20261003)。以下点屏记录为历史状态。

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

**2026-05-22 点亮状态（历史实测）**:
- `panel_newvision_nv3052c` / `sun4i_drm` / `sun4i_tcon` / `sun8i_tcon_top` / `sun8i_mixer` 全部 probe ✓
- `sun4i_drv_bind` → `component_bind_all` → `drm_dev_register` 完整成功链路 ✓
- `/sys/class/drm/card0` + `card0-Unknown-1` 已创建 ✓
- 背光:GPIO backlight `/sys/class/backlight/backlight/` ✓
- Git tag:`lcd-stage-1-lit`(首屏里程碑,commit `12e89e8e4`)
- **2026-05-22 物理确认**:TTY 登录界面正常显示,SPI init 序列和 BSP 时序均兼容 ✓ **P0-3 完成**

**后续升级**(可选):
- 方案 B:PWM 背光(亮度可调)— Batocera 0005/0006/0007 三个 patch
- HDMI 双路属于通用 H700 参考方案；RG35XX-SP 没有 HDMI 输出口，不列为本机待办。

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

- **2026-05-23 实测**([源:实测] ssh root@DEVICE_HOST):`/sys/devices/system/cpu/cpu0/cpufreq/` 全部存在
  - `scaling_available_frequencies`:480000 720000 1008000 1032000 1104000 1200000 1416000 1512000(8 档 OPP)
  - `scaling_governor`:schedutil(默认)
- **已修复**:`rg35xxsp.config` 的 `CONFIG_ARM_ALLWINNER_SUN50I_CPUFREQ_NVMEM=y`(builtin)生效

### P0-6 audio — **驱动与用户态配置已完成，听音验收记录待补**

- **2026-05-23 实测**([源:实测] serial.log):声卡已 probe,且 headphone jack input 已注册
  - `input: H616 Audio Codec Headphone Jack as /devices/platform/soc/5096000.codec/sound/card0/input3`
  - systemd `Save/Restore Sound Card State` 服务正常启动
- **后续记录**：主线指南已记录 ALSA 卡名修正和默认 aplay 配置；7.2.9 部署确认音频驱动及 ALSA 声卡枚举正常。原始“aplay -l 待验证”不再作为驱动缺失；实体扬声器 / 耳机听音的人工验收记录仍待补，未据此认定声音存在故障。
- **已修复**:`rg35xxsp.config` 的 `CONFIG_SND_SUN4I_CODEC=m` + 模块自动加载生效

### P0-7 电池 / USB power — ✅ **已工作(2026-05-23 实测)**

- **2026-05-23 实测**([源:实测] ssh root@DEVICE_HOST):
  - `axp20x-battery`:type=Battery, status=Discharging, capacity=90%, voltage=3.989V, current=430mA ✓
  - `axp20x-usb`:type=USB, present=0, online=0(USB 未插入,正常) ✓
  - `/sys/class/power_supply/axp20x-{battery,usb}/` 全部 sysfs 属性可读
- **关于 DMA mask warning**:dmesg 仍有 `DMA mask not set`(P1-2),但 **不影响功能** — 旧假设"probe 失败"被实测推翻
- **已修复**:`rg35xxsp.config` 的 `CONFIG_AXP20X_POWER=m` + `CONFIG_BATTERY_AXP20X=m` 模块自动加载生效

### P0-8 poweroff 后 RESET 不能开机 — **已关闭：正常硬件行为**

- 2026-05-20 最初将关机后 RESET 无法唤醒误判为故障；随后实测电源键长按可开机，2026-05-21 用户确认 RESET 只用于开机状态下复位 SoC。
- 关机后通过电源键 PWRON 唤醒 PMIC 属正常设计。撤销旧的“避免 poweroff”、拆电池救援及追加唤醒补丁建议，不作为当前待办。

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

### P1-2 AXP 子驱动 `DMA mask not set` warnings — **已修复（2026-10-11 重启实测）**

- **证据**:
  - `platform axp717-adc: DMA mask not set`
  - `platform axp20x-usb-power-supply: DMA mask not set`
  - `platform axp20x-battery-power-supply: DMA mask not set`
- **已确认根因**：MFD core 用 I2C 父设备的 NULL `dma_mask` 覆盖了 `platform_device_alloc()` 初始化的默认 mask；原先猜测子驱动应调用 `dma_coerce_mask_and_coherent()` 不准确。
- **修复**：父设备有 mask 时继承，否则保留 platform 默认值；提交 `90b3fdf`。
- **验证**：7.2.9 #2 实际重启后三条警告均消失，电池 SOC、USB 电源及电压读数正常；详见[清理记录](boot-warning-cleanup.md)。

### P1-3 `pwrseq_simple` probe 失败 `-ENOENT` — ✅ **已解决(2026-05-23 实测)**

- **验证**(2026-05-23 serial.log):pwrseq_simple error 不再出现。随 P0-1 WiFi 修复自动消失。

### P1-4 `Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1` — ✅ **已解决(2026-05-23 实测)**

- **修复**:U-Boot defconfig 加 `CONFIG_ENV_IS_NOWHERE=y` + `# CONFIG_ENV_IS_IN_FAT is not set`
- **验证**(2026-05-23 serial.log):启动日志中 `Unable to read "uboot.env"` 已消失

### P1-5 binfmt_misc — **配置已纳入构建，服务验收记录待补**

- **根因**:`CONFIG_BINFMT_MISC` 未设置
- **修复**:`rg35xxsp.config` 加 `CONFIG_BINFMT_MISC=y`
- **补验收记录**：`ls /proc/sys/fs/binfmt_misc/` 应存在;`systemctl status systemd-binfmt.service` 应正常

### P1-6 SO_PASSSEC — **配置已纳入构建，运行验收记录待补**

- **根因**:`CONFIG_SECURITY_NETWORK` 未设置,`net/core/sock.c` 中 `SO_PASSSEC` 被 `IS_ENABLED(CONFIG_SECURITY_NETWORK)` gate
- **修复**:`rg35xxsp.config` 加 `CONFIG_SECURITY_NETWORK=y`
- **补验收记录**：`dmesg | grep "SO_PASSSEC failed"` 应消失

### P1-7 `aldo3: disabling` — ✅ **非 bug,已关闭(2026-05-23)**

- **分析**:DTS 中 `reg_aldo3` 明确标注 `/* 1.8v - unused */`(sun50i-h700-anbernic-rg35xx-2024.dts:341)。aldo3 在此板上无消费者,regulator 框架自动关闭以省电——**这是正确行为**,不是 bug。
- **决定**:关闭,无需修复

### P1-8 `vcc-pl not found, using dummy regulator`

- **当前状态（2026-10-11）**：7.2.9 #2 仍出现。供电连接未确认，保留现状，不凭猜测绑定电源。
- **证据**:`sun50i-h616-r-pinctrl 7022000.pinctrl: supply vcc-pl not found, using dummy regulator`
- **尝试修复失败**(2026-06-06): 启用 `reg_aldo1` + 添加 `&r_pio { vcc-pl-supply = <&reg_aldo1>; };` → 导致 17 个 pinctrl "Error applying setting" → MMC 全部 probe 失败 → 启动挂死。已回滚。
- **原因推测**: ALDO1 可能实际给其他关键供电域供电,不能随便启用;或需确认 ALDO1 在此板的真实走线
- **影响**: dummy regulator 够用,PL 引脚(RSB/I2C)功能正常。暂不修复。

### P1-9 `usb_phy_generic: dummy supplies not allowed for exclusive requests (id=vbus)`

- **证据**:`[ 1.313] usb_phy_generic usb_phy_generic.1.auto: dummy supplies not allowed for exclusive requests (id=vbus)`
- **当前确认的日志来源（2026-10-11）**：sunxi MUSB 注册的兼容 NOP transceiver 请求可选 vbus；exclusive 请求找不到电源时打印警告并返回 ENODEV，generic 驱动允许该结果。实际 PHY 由独立 sun4i 驱动管理；此前推测此日志等于板级 OTG 电源未接，并未证实。
- **当前范围**：USB gadget CDC ECM 与 SSH 已验证；Host 外设供电未完成验收。旧 vbus warning 不足以单独证明 Host 根因，须结合当前 PHY、dr_mode 与实际供电测量。

### P1-10 DT dependency cycle warnings(CCU ↔ RTC)

- **当前状态（2026-10-11）**：此次 7.2.9 启动日志未重现，以下为历史记录。
- **证据**:`/soc/clock@3001000: Fixed dependency cycle(s) with /soc/rtc@7000000`(及 7010000 互相,共 12 行)
- **原因**:mainline sun50i-h616.dtsi 中 `clock@3001000` (CCU) 与 `rtc@7000000` (sun6i-rtc) 和 `clock@7010000` (R-CCU) 之间引用环。kernel `Fixed` 自动解,但上游可以更整洁。
- **影响**:无功能影响,只是 dmesg 噪声。

### P1-11 `phy phy-5100400.phy.0: Changing dr_mode to 1` — **历史 warning，当前未重现**

- **证据**:`[ 1.309] phy phy-5100400.phy.0: Changing dr_mode to 1`(切到 OTG/host)
- **原因**:DTS 默认 `dr_mode = "otg"` 被运行时改成 host。注意一致性。
- **当前状态（2026-10-11）**：7.2.9 DTB 确认 USB=peripheral，两次启动日志均无此项。Host 待办统一见 P1-9、I-1。

### P1-12 ~~用户态有"每 30 秒 drop_caches"后台脚本~~ — ✅ **已解决(2026-05-23 rebuild rootfs 时自动消失)**

- **原始证据**:`[ 32.327] sh (336): drop_caches: 3`...,每 30 秒一次
- **根因**:BSP 原厂 rootfs 有 systemd service 定期 `echo 3 > /proc/sys/vm/drop_caches`
- **修复**:重建 rootfs 后该 service 不再存在,dmesg 无 `drop_caches` 输出,`systemctl list-timers` 无相关条目

### P1-13 KASLR 未启用 — ❌ **Won't fix(2026-05-23)**

- **分析**:需为 U-Boot 写新的 sunxi CE TRNG 驱动(H616 Cortex-A53 无 ARMv8.5 RNDR),工作量很大。掌机场景安全性收益极小。
- **决定**:降级为已知限制,不修复。

### P1-16 `sun4i-usb-phy 5100400.phy: failed to get reset usb0_reset` — ✅ **已修复(2026-05-23 实测)**

- **验证**(2026-05-23 serial.log):`failed to get reset usb0` 不再出现。commit `6a291bb29` 生效。

### P1-17 `dw-apb-uart ...: Error applying setting, reverse things back` — **历史记录，当前未重现（2026-10-11）**

- **证据**([源:实测] serial.log):uart0/uart1 两处报错,pinctrl 框架回退
- **分析**:sunxi pinctrl 驱动不支持 DW UART 请求的某个 generic pin config 参数。pinmux-pins debug 确认 PH0/PH1 correctly muxed to uart0, PG6-9 correctly muxed to uart1。UART 功能完全正常(ttyS0 console, ttyS1 注册)
- **影响**:纯 cosmetic,无功能影响
- **历史决定**：当时暂不修复。当前 7.2.9 清理前后日志均无此项，不列为现存故障；未为此新增 UART 补丁。

### P1-18 `cfg80211: loaded regulatory.db is malformed` — **历史记录，当前未重现（2026-10-11）**

- **尝试修复失败**:关闭 `CONFIG_CFG80211_REQUIRE_SIGNED_REGDB` 会导致 `verify_pkcs7_signature` 符号从内核消失,cfg80211 模块编译失败
- **原因**:Kconfig 有 `default y` 且无 prompt(除非 `CFG80211_CERTIFICATION_ONUS=y`),无法通过 rg35xxsp.config 关闭
- **影响**:WiFi 关联正常(`wlan0: associated`),仅使用 world regulatory domain(可能限制发射功率/信道)
- **当前复核**：设备已有配对的 `regulatory.db` 与签名；7.2.9 清理前后均无 malformed 警告，WiFi 关联与 DHCP 正常。本次未修改签名校验配置或固件，也未逐信道验收发射功率。

### P1-19 `gpiochip*: Static allocation of GPIO base is deprecated` — **已修复（2026-10-11 重启实测）**

- **证据**([源:实测] serial.log L223/224/243/245):
  - `gpio gpiochip0: Static allocation of GPIO base is deprecated, use dynamic allocation.` ×3
  - `gpio gpiochip1: Static allocation of GPIO base is deprecated, use dynamic allocation.` ×1
- **原因**:内核 GPIO 子系统已废弃静态 GPIO base 分配,但 H616 pinctrl 驱动(`sun50i-h616-pinctrl`/`sun50i-h616-r-pinctrl`)仍在使用旧 API
- **影响**:纯 cosmetic,无功能影响
- **修复**：提交 `90b3fdf` 将 sunxi GPIO base 改为动态分配，同时将四处硬件引脚计算切回描述符 pin_base，避免硬件编号依赖全局 GPIO 编号。
- **当前实测**：7.2.9 原有两条警告在 #2 重启后均消失；GPIO 消费者、TF2、WiFi、背光和 LCD 自检正常。上述 ×4 是旧启动记录，不是当前计数。

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
- 构建入口见公开 BSP 仓库与主线指南
- Mainline 支持表已更新(Display/VPU 标 ✅, BT 标 ⚠️ 弃用)

### P2-2 ~~device-facts 应补充 mainline 实测条目~~ — ✅ **已解决(2026-06-06)**

- §24: 已补充 `SPL/BL31 v2.14.2 → U-Boot 2026.04 → Linux 7.0.9` mainline 实测
- §22: 已补充 p4 回滚锚点备注 + p5 modules 已不可逆
- §21: 已补充 p2 改作 mainline boot 分区

### P2-3 ~~P5 模块清理~~ — ✅ **已解决(2026-06-06)**

- **修复**: rootfs 清理阶段自动删除旧 BSP 4.9.x 模块目录 + `mali_kbase.ko`
- 见 P1-1（2026-05-21 手动清理过设备，现纳入脚本自动化）

### P2-4 刷卡增量 / 分项更新 — **公开流程待完善**

公开 BSP 提供内核、启动固件和模块构建产物。应根据实际卡布局分别部署启动文件与模块；
独立的刷卡子模式不作为当前公开工具能力。构建与布局说明见[主线指南](mainline-guide.md)。

### P2-5 刷写备份与回退 — **尚无公开通用自动流程**

开发阶段有单次升级的备份与回退验证，不能将其当作公开自动化功能或原厂整卡恢复方案。
通用校验、设备外备份和回退流程仍待整理。


### P2-6 RTC 双时源问题 — ✅ **已解决(2026-09-05 实机验证)**

#### 一次修复(2026-06-06)——只改了用户态,不成立

- **当时的根因认识**: sun6i-rtc(rtc0,无电池) vs PCF8563(rtc1,有纽扣电池),默认 hwclock 写 rtc0 → 掉电归零
- **当时的修复**:
  1. udev 规则 `/etc/udev/rules.d/99-rtc-pcf8563.rules`: `SYMLINK+="rtc"` 让 `/dev/rtc → rtc1`
  2. `/etc/adjtime` 第 4 行写 `/dev/rtc1`
- **当时的验证**: `/dev/rtc → rtc1` ✓, `hwclock --systohc` 写入 rtc1 ✓
- **为什么不成立**: 验证只覆盖了用户态。`/dev/rtc` 和 `hwclock` 是用户态路径;**开机时间不由 hwclock 决定**,而是内核 `rtc_hctosys()` 读 `CONFIG_RTC_HCTOSYS_DEVICE`("rtc0" = sun6i)。所以掉电停走的那块芯片依然决定开机时间,改动等于没生效。另外 `/etc/adjtime` 第 4 行 `/dev/rtc1` 是**无效的** —— hwclock 的 adjtime 解析只用前 3 行 + `UTC|LOCAL`,真正起作用的是 udev 符号链接。

#### 暴露方式(2026-09-05)

`ssh root@DEVICE_HOST 'pacman -Syu'` 失败:

```
error: failed retrieving file 'extra.db' from mirrors.sjtug.sjtu.edu.cn :
SSL certificate ... certificate is not yet valid or the system clock is incorrect (9)
```

系统时间 `Sep 01 06:26 UTC`,真实时间 `Sep 05 02:38 UTC` —— 慢 3 天 20 小时,SJTU 镜像新签发的 Let's Encrypt 证书还没到 `notBefore`。

#### 完整根因(三条叠加)

1. **内核用错芯片**: `CONFIG_RTC_{HCTOSYS,SYSTOHC}_DEVICE="rtc0"` = sun6i,无电池掉电停走。开机 dmesg 实测:`sun6i-rtc 7000000.rtc: setting system clock to 2026-09-01T14:22:09 UTC` —— 取到 4 天前的值。同一时刻 PCF8563 里的日期是正确的 Sep 5(它一直在走)。
2. **PCF8563 `=m` 导致编号错位,永远匹配不上 "rtc0"**: `rtc-pcf8563 0-0051: registered as rtc1` 在 `[11.56s]`,而 sun6i 是 builtin platform driver,`[0.96s]` 就 probe 完并用 `ida_alloc()` 抢到 id 0。注意 `rtc_hctosys()` **不是 initcall**,而是在 `class.c:433` 的 `__devm_rtc_register_device()` 里按名字匹配触发:`if (!strcmp(dev_name(&rtc->dev), CONFIG_RTC_HCTOSYS_DEVICE)) rtc_hctosys(rtc);` —— 注册成 rtc1 就等于这段代码对它永不执行。
3. **timesyncd 被禁用**: 历史 rootfs 配置中有 `systemctl disable systemd-timesyncd.service`,于是这 4 天误差没有任何东西纠正。

附带发现:`/etc/adjtime` 写 `LOCAL` 会让 systemd 开机 `warp_clock()` 减 480 分钟并置内核 `persistent_clock_is_local=1`,使内核 11 分钟同步往 rtc0 写**本地时间**(实测 `rtc0_raw=11:30:25` vs `UTC=03:30:24`)。两块 RTC、两个维护者、两种时间基准。

#### 二次修复(2026-09-05)

1. `rg35xxsp.config`: `CONFIG_RTC_DRV_PCF8563=y`(builtin。编号靠下面的 aliases 钉死,builtin 的作用是让 hctosys 发生在 `[~1s]` 而非 `[~11.5s]`,否则 systemd 头 11 秒都在用错误时钟跑。前置 `CONFIG_I2C=y` + `CONFIG_I2C_MV64XXX=y` 已满足)
2. `sun50i-h700-anbernic-rg35xx-sp.dts`: 加 `aliases { rtc0 = &rtc_ext; rtc1 = &rtc; }` 把 PCF8563 钉成 rtc0。**两个节点都要给 alias** —— `class.c:rtc_device_get_id()` 先试 `of_alias_get_id()` 再 `ida_alloc()`,只给一个的话 sun6i(platform drv,先 probe)会抢到 id 0
3. 用户态配置： 删掉 `99-rtc-pcf8563.rules`(PCF8563 成 rtc0 后,udev 自带 `ATTR{hctosys}=="1"` 规则会自动指过去;**若保留该规则会把 `/dev/rtc` 反指到 sun6i,更糟**);`/etc/adjtime` 改 `UTC` 并去掉无效的第 4 行
4. 用户态配置： 改回 `systemctl enable systemd-timesyncd`,并配置适合目标网络的 NTP 服务器

#### 顺带纠正的一个错误结论

排查中曾因 `hwclock` 默认打开 `/dev/rtc`(→ rtc1)而误判"`CONFIG_RTC_SYSTOHC=y` 在这块板子上空转"。实测证伪:直接对 rtc0 拨慢 20 分钟,内核在 6 分钟后纠正(11 分钟模式正常工作)。**内核 SYSTOHC 无问题,此前是测错了芯片。**

#### 实机验证(2026-09-05, Linux 7.2.3)

- 连续重启验证:`rtc0 = rtc-pcf8563 0-0051`,`hctosys=1`;`rtc1 = sun6i-rtc 7000000.rtc`,`hctosys=0`;`/dev/rtc -> rtc0`
- 启动日志:`[1.317992] rtc-pcf8563 0-0051: registered as rtc0`,`[1.324442] ... setting system clock to 2026-09-05T07:40:01 UTC`;不再由 sun6i-rtc 设置系统时间
- `/etc/adjtime` 为 `UTC`;RTC 原始值 `2026-09-05 07:41:59` 与同秒系统 UTC 一致;不再出现 `RTC configured in localtime, applying delta of 480 minutes`
- `systemd-timesyncd` enabled + active,实测连接 `ntp2.aliyun.com` (`203.107.6.88`)并完成初始同步
- 已删除旧 `/etc/udev/rules.d/99-rtc-pcf8563.rules`;临时兜底 `rtc-sync.service` / `rtc-sync.timer` 已 disable、删除,再次重启仍正常
- **部署注意**：只更新内核、模块和固件不会自动应用 RTC 的用户态配置；需在目标 rootfs 中单独配置并验证。
- **后续观察(不阻塞关闭)**:断电静置数小时后开机,比较 RTC 与真实 UTC,量化 PCF8563 漂移率;若持续异常则排查晶振/负载电容硬件

---

### P1-21 H616 Cedrus VPU 硬解 — **HEVC / H.264 基础链路已通，部分视频兼容性仍待处理**

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
- **已配置**：2026-06-04 已记录 `/etc/mpv/mpv.conf` 使用 `hwdec=drm`、`vo=gpu-next`、`gpu-context=drm`、`profile=fast`；不再保留“将 hwdec=auto 写入 menu”旧待办。历史验证中 auto 未自动检测 request 硬解。
- **尚未记录解决**：640×480 H.264 P 帧损坏、720×540 DMA-BUF pitch 导入失败，以及部分播放掉帧 / 音画同步问题；先在当前 7.2.9 复测，区分解码输出与显示导入。详见 [H.264 记录](v4l2-request-h264.md) 和 [H.264 记录](v4l2-request-h264.md)。
- **当前能力限制**：正式 DE33 驱动不公布 NV12 和硬件缩放；旧实验中临时暴露格式、写 CSC 或扩展 guard 的成功不代表这些改动已纳入当前驱动。
- **可选增强**：MPEG-2 / VP8 v4l2-request hwaccel；低优先级。

---

### P1-22 音量按键调节 ALSA — **服务已实现，实体按键验收记录待补**

- **现象**: `gpio-keys-volume` 已注册,但按音量键不会改变实际播放音量。
- **根因**: 当前路线 C 是最小 console/systemd rootfs,没有桌面/session 音量守护进程消费 `KEY_VOLUMEUP/DOWN` 事件。
- **设备状态**:
  - `/proc/bus/input/devices`: `gpio-keys-volume` 注册为 `event1`,带 `kbd` handler
  - ALSA mixer:可调输出控件为 `DAC`(0-63)和 `Line Out`(0-31);当前使用 `DAC` 做用户音量
- **修复**:
  - 新增 `/usr/local/bin/volume-buttond`:自动查找名为 `gpio-keys-volume` 的 event 节点,用 `evtest` 监听 `KEY_VOLUMEUP/DOWN`,调用 `amixer -c 0 set DAC 3%+/- unmute`
  - 新增 `volume-buttond.service`,已在设备 `sp` 上 `enable --now`
  - 服务属于用户态配置，制作 rootfs 时需单独准备
- **验证**:
  - `systemctl status volume-buttond.service`:active/running
  - `ps`:服务正在运行 `evtest /dev/input/event1`
  - **人工验收待补**：记录实体音量 +/- 的 DAC 数值变化及实际音量变化。已有运行服务与 evtest 进程记录，不等于完成了实体按键和听音验收，也未据此认定当前按键失效。

---

## 待调查 — 不确定但应该早看

### I-1 USB-OTG / Host 模式 — **gadget 已验证，Host 仍待验收**

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

## 历史一期完成确认（2026-05-20）

| 子系统 | 状态 |
|---|---|
| TF-A v2.14.2 → U-Boot 2026.04 → Linux 7.0.9 全链路 | ✅ |
| earlycon + serial console ttyS0 | ✅ |
| systemd init + getty | ✅ |
| EXT4 rootfs mount (mmcblk0p5) | ✅ |
| AXP717 PMIC probe | ✅ |
| 4 核 SMP + PSCI(无路线 A 的 RCU stall) | ✅ |
| root 登录、shell 可用 | ✅ |

## 状态汇总（2026-10-11）

- **已关闭 / 已验证的主要项**：WiFi 基础联网、LCD 点亮及边缘路由、GPU、cpufreq、电池、TF2、RTC、USB gadget；各项验证日期和版本以正文为准，7.2.9 并未重新测试全部外设。
- **实现完成但验收记录待补**：音频听音、实体音量键、binfmt 服务、SO_PASSSEC 运行表现。
- **仍有功能或流程工作**：视频兼容与性能、DE33 NV12 / 缩放、USB Host、刷卡分项模式、自动备份与回退。P2-1/2/3/6 已完成，P2-4 部分完成，P2-5 通用流程未完成。
- **已接受 / 可选 / 不适用**：蓝牙弃用、KASLR 不修、无模拟摇杆；PWM 调光和 MPEG-2 / VP8 硬解为增强项。旧 warnings 需复核后再计为当前问题。
- 历史总数“P0 7/7、P2 6/6”与上述范围不符，已撤销；当前待办以本文开头表格为准。

**本期新增成就**:
- USB gadget CDC ECM (网络地址按使用环境配置)
- TF2 卡槽 (PE4/PE22, 59.5 GB SDXC)
- 音频默认设备修复 (asound.conf audiocodec→Codec)
- 音量按键直控 ALSA DAC 音量(volume-buttond)
- SSH config USB/WiFi 自动切换
- 构建与部署属于不同步骤，公开入口见主线指南。
