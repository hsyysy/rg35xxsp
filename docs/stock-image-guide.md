# 原厂镜像路线

## 推荐路线

优先使用[主线内核指南](mainline-guide.md)中的 BSP 构建双分区卡。原厂镜像
适合只读分析或作为回滚参考，不建议在保留原厂 rootfs 的同时混用主线模块。

## 原厂镜像结构

原厂固件使用 7 个分区：

- 前 36 MiB：BOOT0 / U-Boot / DTB 原始区。
- p1：Roms / 系统资源，FAT。
- p2：boot-resource，FAT，存放 logo 和充电图。
- p3：U-Boot 环境变量。
- p4：Android boot image v2，包含 kernel、ramdisk 和 DTB。
- p5：Ubuntu 22.04 rootfs。
- p6：appfs。
- p7：UDISK。

原厂 U-Boot 根据 `lcd_type` 在多个 DTB 中选择面板时序，并使用直接 bootcmd。
这些行为与主线 U-Boot 不同。

## 保留原厂启动链

如果只想更换用户态：

1. 备份 p4 原厂 boot image 和 p5 rootfs。
2. 保留 p4 作为回滚入口，不要覆盖。
3. 在 p5 上准备与内核匹配的 rootfs。
4. 原厂内核模块与主线模块不能混用。

注意：清理原厂 `/lib/modules/4.9.170` 后，即使恢复 p4，也需要从备份恢复
模块才能回到完整原厂状态。

## 树外模块

原厂 4.9 内核可以编译少量树外模块，但需要匹配原厂内核源码和编译器。
主线内核不能加载这些模块；反之亦然。不要把树外模块当作主线驱动的替代品。

## 风险

- 原厂分区表和 U-Boot 环境变量与主线双分区布局不兼容。
- 改动 p5 是不可逆的，除非事先做完整备份。
- 原厂 boot image 依赖闭源 U-Boot 的 `lcd_type` 选择逻辑；主线直接使用
  `sun50i-h700-anbernic-rg35xx-sp.dtb`。

刷写前请先建立可恢复的整卡备份。当前 BSP 的刷写脚本尚未内置自动备份和
回滚，见[支持与限制](status.md)。
