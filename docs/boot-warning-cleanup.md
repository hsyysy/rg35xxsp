# 启动告警

当前 7.3.1 启动没有失败的 systemd 服务，`journal` 中没有 error 级记录。
`dmesg --level=warn,err` 只剩以下已接受项。

## 已修复

| 告警 | 处理 |
|---|---|
| GPIO static allocation | sunxi pinctrl 使用动态 GPIO 编号 |
| AXP717 子设备 DMA mask not set | MFD 在父设备有 DMA mask 时才继承 |
| TF2 read-only switch | TF2 没有机械写保护信号，DTS 使用 `disable-wp` |
| networkd nftables socket | 启用 `CONFIG_NF_TABLES=m` |
| `usb_phy_generic` vbus 提示 | PHY 注册流程已整理，不再出现 |
| `vcc-pl` dummy regulator | 按实际供电关系处理，不再出现 |
| 面板模块 out-of-tree taint | 使用整树 in-tree 构建消除 |

## 保留项

| 提示 | 结论 |
|---|---|
| KASLR disabled due to lack of seed | U-Boot 未提供随机种子，保留 |
| SDIO voltage range | 过滤无效 OCR 低位后选择有效电压；WiFi 已验证 |
| `sunxi_cedrus` staging | 驱动成熟度提示，保留 |

当前 `tainted=1024`，只来自 cedrus staging，不是 out-of-tree 模块。
没有通过 `quiet` 或降低 loglevel 隐藏这些提示。

## 验证

整树构建并重启后检查：

```bash
systemctl --failed
journalctl -p err -b
dmesg --level=warn,err
cat /proc/sys/kernel/tainted
```

升级内核版本时，需要逐项确认上游是否已包含这些修复，避免重复回移。
