# 启动告警清理

2026-10-11，Linux 7.2.9 开发设备完成修复、部署与重启验证。
该次启动 warning 从 16 条降至 5 条，没有失败的 systemd 服务。
计数对应这一验证环境，不作为其他 rootfs 的预期值。

## 已修复项

| 告警 | 原因与修复 |
|---|---|
| GPIO static allocation | sunxi pinctrl 使用动态 GPIO 编号，硬件引脚与 IRQ 映射仍使用描述符 pin_base |
| AXP717 子设备 DMA mask not set | MFD 仅在父设备有 DMA mask 时继承，避免以 NULL 覆盖子设备默认值 |
| TF2 read-only switch | TF2 没有可读取的机械写保护信号，DTS 补充 disable-wp，保留卡检测 |
| networkd nftables socket | 启用 CONFIG_NF_TABLES=m，提供所需 netlink API |
| networkd unpredictable interface name | 开发设备的网络配置按驱动匹配，避免只依赖内核枚举名称 |

前四项内核源码 / 配置改动已包含在公开 BSP 中。
按驱动匹配接口属于 rootfs 的网络配置，需由使用者按系统环境设置。

## 保留的告警与限制

| 提示 | 结论 |
|---|---|
| KASLR disabled due to lack of seed | U-Boot 未提供随机种子，保留该限制 |
| usb_phy_generic optional vbus | 兼容 transceiver 的可选电源提示，不能据此单独判断 Host 供电；Host 仍需验收 |
| R_PIO vcc-pl dummy regulator | PL 供电连接尚未核实，不用猜测的电源映射消除提示 |
| SDIO voltage range | 过滤 SDIO OCR 的无效低位声明后选择有效电压；已有 WiFi 连接验证 |
| sunxi_cedrus staging | 驱动成熟度提示，保留 |

没有通过 quiet 或 loglevel 隐藏这些告警。

## 验证范围

- Image、DTB 与设备产物校验值一致，模块与 7.2.9 匹配。
- USB / WiFi、TF2、PMIC 与 RTC 状态正常。
- LCD 路由保持 `0x20`，已有显示自检和前端恢复记录；此次没有新增逐图案人工验收。
- fbcon 解绑和前端 journal 输出配置通过重启复核。

后续移植内核版本时，需逐项检查上游是否已包含这些修复，避免重复回移。
