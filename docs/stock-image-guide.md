# RG35XX-SP 原厂镜像路线

> 保留原厂 boot.img (SPL + ATF + U-Boot + BSP 4.9.170 kernel), 仅替换 userspace 或追加外设驱动。

## 路线总览

| 路线 | 内核源 | 现成度 | 状态 | 优 | 劣 |
|---|---|---|---|---|---|
| **C. 二进制注入 + rootfs 重做** | 保留原厂 Image + 全部 `.ko`, 仅换 userspace | 100% | **最低成本可行** | 0 编译问题; 硬件 100% 兼容 | 不能改任何内核逻辑 |
| **A'. Alpine 直接套** | 复用原厂 boot.img 里的 BSP 4.9 内核 | 100% | 可用, 但实质同 C | 最快 2 小时出图 | 仍是原厂 BSP 4.9 内核 |
| **D. 树外 `.ko` 编译** | 原厂内核 + 干净 `linux-4.9.170` 树编单个 `.ko` | 100% | 中等成本可行 | 加 USB 外设驱动不动原厂内核 | 只能加附加驱动, 不能替换原厂模块 |
| **A. BSP 4.9 重编** | HandsomeMod BSP 4.9.118 + 设备 dts + config | 0% | **已死亡** | 理论可保留 libmali / DE2 / 4K Cedar | CPU 0 RCU stall, 详见下方 |

**推荐**: 如果不需要改内核, 直接走 C（二进制注入）或 A'（Alpine）。需要加外设驱动走 D。想换主线内核见 [`mainline-guide.md`](mainline-guide.md)。

## 路线 C: 二进制注入 + rootfs 重做

**原理**: 保留原厂 `boot.img` (kernel + ramdisk + dtb) 不动, 仅替换 rootfs 分区 (mmcblk0p5) 上的 userspace。

**参考实现**: MuOS / StockOS MOD 都走这条路。

### 基本步骤

1. 备份原厂 SD 卡 (至少 `mmcblk0p4` + `mmcblk0p5`)
2. 准备新 rootfs (Ubuntu/Alpine/Debian arm64)
3. 将新 rootfs 写入 mmcblk0p5
4. 确保 `/lib/modules/4.9.170/` 完整保留 (原厂 `.ko` 全集)
5. 确保固件文件在 `/lib/firmware/` (WiFi/BT 等)

### 注意事项

- 原厂 kernel Image 和所有 `.ko` 必须原封不动保留
- `mali_kbase.ko` (BSP r20p0) + `libmali.so.0.20.0` 必须配套
- 原厂 `launcher.service` 每 30s 写 `drop_caches`, 换 rootfs 后记得 `systemctl mask`
- `/etc/modules` 里的 `rtl_btlpm` 和 `/etc/rc.local` 里失效的 BSP `insmod` 行需清理

## 路线 A': Alpine 直接套

**参考**: `thirdparty/alpine-h700/`

**原理**: 原厂内核 + Alpine mini rootfs, 最轻量方案。

**优势**: 最快出图; 原厂 mali blob + 原厂内核 = 硬件全兼容。

**限制**: 本质还是 BSP 4.9 内核, 无法获得主线安全更新和新特性。

## 路线 D: 树外 `.ko` 编译

**原理**: 用原厂 4.9.170 内核 + 干净 `linux-4.9.170` 源码树 + Knulli 的 config 编译单个 `.ko` (如 `cdc-acm.ko`), 然后 `insmod` 加载。

**参考 config**: `knulli-cfw/distribution/.../linux-sunxi64-legacy.config`

**限制**: 只能加附加驱动 (USB 外设等), 不能替换原厂已有模块。编出的 `.ko` vermagic 必须与原厂 `4.9.170` 完全一致。

## 路线 A: BSP 4.9 重编 (已死亡)

> **判定**: 2026-05-19 串口实测确认 CPU 0 进 idle 后永不唤醒 (RCU stall), 实质性死亡。

### 死亡根因

HandsomeMod 2021 快照 (4.9.118) 缺少 Allwinner 2021→2025 间加入的 29 个孤儿驱动, 最关键的是:

- `CONFIG_AXP2202_POWER` (PMIC 电池 + USB 电源管理) — 整个符号在 HandsomeMod 树中不存在
- `SUNXI_THERMAL_NG` / `ARM_ALLWINNER_SUN50I_CPUFREQ_NVMEM` — cpuidle/cpufreq 依赖
- 12 个 Audio HUB v2 驱动
- BSP 私有 `arch/arm64/kernel/cpuidle.c` patch — CPU 0 进 WFI 后 IPI 无法唤醒

公网不存在 Anbernic 原厂 4.9.170 BSP 源码 (GPL 违规), 修复路径不通。

### 串口 panic 日志摘要

```
[    0.000000] Linux version 4.9.170 (x@SYPC) (gcc version 5.3.1 20160412
                  (Linaro GCC 5.3-2016.05) ) #3 SMP PREEMPT Tue May 19 20:12:54 CST 2026
[    0.767306] uart uart1: uart1 error to get fifo size property
[   21.793959] INFO: rcu_preempt detected stalls on CPUs/tasks:
                  0-...: (0 ticks this GP) idle=2fd/140000000000000/0 softirq=29/29 fqs=0
```

CPU 0 在 0.767s 后进 idle 永不出来, CPU 3 在 21s 后报 RCU stall。

### 编译历史 (留存参考)

**工具链**: 固定 Linaro GCC 5.3.1 (与 Anbernic 设备 BSP 同源)。仅 2 处真实源码补丁:
1. `sensor_helper.c` 的 `ccm0`/`ccm1` 多重定义 (任何工具链都冲突)
2. `disp2/` 缺 SUN50IW9 平台分发 (HandsomeMod 2021 快照缺失)

**板级 dts**: 用设备 `device/dtb/0_1334800.dts` (lcd_type=old) 作为基线。

**配置**: `cp device/config-4.9.170 .config && make ARCH=arm64 oldconfig` — 暴露 91 个 `(NEW)` 符号。

### 首次试刷复盘 (2026-05-16)

黑屏 brick。回滚路径: 拔 TF1 SD 卡 → 主机读卡器 `dd if=device/boot.img of=/dev/sdd4` → 装回。

**教训**: deploy 脚本备份了 `/dev/mmcblk0p4` 但没防 `tar -xzf` 覆盖 `/lib/modules/`, 需要先删旧目录再解压。

### AXP2202 驱动方案 (未实施, 留存)

三条可行路径:
- **方案 A**: 从 mainline 移植 AXP717 驱动 (mainline 中 AXP2202 = AXP717, `x-powers,axp717`)
- **方案 B**: 扩展 HandsomeMod 已有 `axp2101-chg.c` 加 AXP2202 分支
- **方案 C**: 写独立 platform driver

详见原 README 路线 A 章节。

### 原厂源码检索审计

**结论**: Anbernic RG35XX-SP 原厂 4.9.170 BSP 内核源码在公网上不存在。完整审计: `bsp-kernel-source-search.md`。

## 设备关键事实

详见 [`device-facts.md`](device-facts.md), 这里仅列路线选择相关的:

- **内核**: 4.9.170, build `#11 SMP PREEMPT 2025-12-24 flower@flower-B85M-D2V`
- **U-Boot**: 2018.05 (Dec 25 2025)
- **ATF**: v1.0(debug):5a77824
- **Boot 格式**: Android boot image v0, kernel load 0x40080000
- **PMIC**: AXP2202 (≠ AXP717 简单等价, BSP 同时挂载 axp2101/axp2202 两套驱动)
- **WiFi**: 厂商 RTW `8821cs.ko`, 非 mainline rtw88
- **GPU**: Mali-G31 Bifrost, libmali.so.0.20.0 + mali_kbase r20p0
- **DTB**: 三选一, 仅 `lcd_type=old → dtb 0` 实测确认
