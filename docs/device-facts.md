# 硬件事实

本页只列出对移植和维护有用的硬件事实。未特别说明时，数值来自主线运行时
的 `/proc`、`/sys`、`dmesg` 和原厂固件对照。

## SoC 与内存

| 项目 | 值 |
|---|---|
| SoC | Allwinner H700（H616 系列） |
| CPU | 4× Cortex-A53，9 档 OPP，最高 1512 MHz |
| GPU | Mali-G31 MP1（主线使用 Panfrost） |
| 内存 | 1 GiB LPDDR4，32-bit 总线，672 MHz |
| IOMMU | 存在，Panfrost 和 VPU 使用 |

## 显示

| 项目 | 值 |
|---|---|
| 面板 | 3.5 英寸 640×480 RGB888 IPS，NV3052C |
| 厂商代号 | `fog_fj035fhd05_v1`；主线 compatible `anbernic,rg35xx-sp-panel` |
| 背光 | PWM0，PD28，50 kHz |
| HDMI | mini-HDMI（HDMI-C），DWC HDMI 2.0 TX + H616 PHY |
| HDMI 音频 | H616 AHUB DAM + port 1 → DW-HDMI I2S codec |

## 音频

- 模拟音频由 `sun4i-codec` 提供，扬声器 PA 由耳机插孔状态门控。
- 插耳机时关闭扬声器 PA；拔出后恢复。
- 音量键和菜单音量控制 H616 模拟 codec 的 `DAC`，不控制 HDMI codec。
- HDMI 音量由电视或播放软件控制。

## 电源

| 项目 | 值 |
|---|---|
| PMIC | X-Powers AXP717（原厂 BSP 称 AXP2202） |
| 电池 | 3000 mAh |
| 充电 / USB 供电 | AXP717 USB power supply |
| 电源键 | `axp20x-pek`，短按产生 `KEY_POWER` |
| 合盖开关 | GPIO，`SW_LID` |
| HDMI 1.8 V | AXP717 ALDO3，`vcc-hdmi` |

## 连接与外设

| 项目 | 值 |
|---|---|
| WiFi | RTL8821CS，SDIO，mainline `rtw88_8821cs` |
| 蓝牙 | RTL8821CS 组合芯片；当前禁用 |
| USB-C | MUSB gadget + EHCI0/OHCI0 host，共用 PHY0 |
| VBUS 使能 | PI16，高有效 |
| TF1 | 启动 / 根分区 |
| TF2 | 第二 SD 卡槽，常用 exFAT |
| UART0 | PH0/PH1，115200 8N1 |
| 按键 | D-pad、A/B/X/Y、L1/L2/R1/R2、Select/Start/Menu、音量、电源 |

本机没有 eMMC。TF1 是可拆卸 SD 卡。

## 存储布局

原厂固件使用 7 个分区：

| 分区 | 用途 | 大小 |
|---|---|---|
| 前导 | BOOT0 / U-Boot / DTB | 36 MiB |
| p1 | Roms / 系统资源 | 2 GiB |
| p2 | boot-resource | 32 MiB |
| p3 | U-Boot env | 16 MiB |
| p4 | Android boot image | 64 MiB |
| p5 | rootfs | 7 GiB |
| p6 | appfs | 4 GiB |
| p7 | UDISK | 约 1.34 GiB |

主线 BSP 默认使用双分区：p1 为 FAT32 启动分区，p2 为 ext4 根分区。将原厂卡
改为双分区后，原厂 p4 内核和 p5 rootfs 不再保留在同一布局中。

## 温度

SoC 有 4 个热区：GPU、VE、CPU、DDR。PMIC 另有电池温度。主板没有风扇，
温度读数来自 die 内部，不能直接当作外壳温度。

## 其他

- 未启用 OPTEE；`secure_os_exist=0`。
- 原厂 U-Boot 会根据 `lcd_type` 选择不同的 DTB；主线直接使用
  `sun50i-h700-anbernic-rg35xx-sp.dtb`。
- 原厂 boot image 为 Android boot image v2，kernel/ramdisk/DTB 分别加载到
  0x40080000 / 0x42000000 / 0x44000000。
