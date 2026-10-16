# RG35XX-SP 主线内核指南

当前公开构建基线为 **Linux 7.2.9 + U-Boot 2026.04 + TF-A lts-v2.14.2**。
脚本、补丁和配置见 [bsp-rg35xxsp](https://github.com/hsyysy/bsp-rg35xxsp)。

## 内核基线与迁移状态

2026-10-10 已完成 7.2.9 的构建、设备部署、重启及 LCD 视觉验收。
7.2.9 原生包含 TCON TOP 路由修复，不需要重复回移旧版本补丁。
2026-10-11 继续完成启动告警清理、RTC 和正常日志配置的复核。

公开 BSP 于 2026-10-16 同步上述适配，已在干净的 7.2.9 源码上验证补丁，
并完成 Image、DTB、模块编译和打包。此次公开仓库同步没有重复设备部署或硬件验收。
详细硬件范围见[问题清单](issues.md)，LCD 根因见[调试记录](display-worklog.md#lcd-route-fix-20261003)。

## 获取与构建

```bash
git clone --recurse-submodules https://github.com/hsyysy/bsp-rg35xxsp.git
cd bsp-rg35xxsp

# Debian / Ubuntu；TF-A 需要 GCC 11 或更新版本
sudo apt-get install gcc-aarch64-linux-gnu bc bison flex libssl-dev libelf-dev \
    libgnutls28-dev python3-dev python3-setuptools swig

./scripts/build_tfa.sh
./scripts/build_uboot.sh
./scripts/build_kernel.sh          # 增量编译
./scripts/build_kernel.sh release  # distclean 后全量编译
```

内核可以独立构建，U-Boot 依赖 `out/bl31.bin`。
release 模式会清除内核构建配置；自定义配置需提前保存。
构建脚本会核对源码版本和补丁实际状态，每次合并 `configs/rg35xxsp.config`。

## 启动分区与产物

默认双分区布局：p1 为启动分区，p2 为根分区。
旧五分区卡须为两个构建入口使用相同参数：

```bash
BOOT_PART=0:2 ROOT_DEV=/dev/mmcblk0p5 ./scripts/build_uboot.sh
BOOT_PART=0:2 ROOT_DEV=/dev/mmcblk0p5 ./scripts/build_kernel.sh
```

| `out/` 下文件 | 用途 |
|---|---|
| `bl31.bin` | TF-A BL31 |
| `u-boot-sunxi-with-spl.bin` | U-Boot SPL + BL31 |
| `Image.gz` | 内核镜像 |
| `sun50i-h700-anbernic-rg35xx-sp.dtb` | 板级设备树 |
| `modules.tar.gz` | 与 7.2.9 内核匹配的模块 |
| `p2-payload/` | 内核、DTB 与 extlinux.conf；目录名保留历史兼容 |
| `SHA256SUMS` | 产物校验值 |

`p2-payload` 应部署到实际启动分区，目录名不决定分区编号。
Image、DTB 和模块应使用同一版本的构建产物。解包模块到带 `/lib` 目录软链的
rootfs 时，应保留该软链，例如使用 tar 的 `--keep-directory-symlink` 选项。
固件及用户态准备见 [rootfs 清单](rootfs-prep.md)。

U-Boot 使用直接 bootcmd；生成的 extlinux.conf 适用于使用 extlinux 的启动环境。
只更新 extlinux.conf 不会改变已有 U-Boot 的直接启动参数。调整卡布局或日志参数时，
须核对实际 bootcmd。当前配置保留 `fw_devlink=off`，移除前需独立验证设备依赖。

## 已验证范围

| 子系统 | 记录 |
|---|---|
| 启动链 | TF-A、U-Boot 和 7.2.9 已有设备启动记录 |
| LCD | NV3052C / DE33 点亮、重启后边缘正常 |
| WiFi / USB gadget | 已有连接验证，设备地址由使用环境配置 |
| RTC | PCF8563 为 rtc0、内建，`hctosys=1` |
| PMIC / TF2 | 电源读数有效，TF2 exFAT 读写挂载 |
| 音频 / GPU / VPU | 有驱动初始化及历史功能验证；不代表每次迁移都完成全部人工验收 |

蓝牙保持禁用。USB Host、部分视频样本及 DE33 硬件缩放仍有待办。

<a id="lcd-控制台输出2026-10-11"></a>

## LCD 控制台输出

fbcon 与前端可能共用 UI 主层。开发设备通过解绑 framebuffer vtconsole、
将前端标准输出接入 journal，避免控制台文本覆盖界面；已通过重启复核。
这是 rootfs / 服务配置，不能仅靠更新内核或降低日志等级代替。

启动告警修复见[清理记录](boot-warning-cleanup.md)，正常和诊断日志参数见
[日志配置](kernel-debug-branches.md)。
