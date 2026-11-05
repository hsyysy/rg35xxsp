# 主线内核指南

公开 BSP 基线为 **Linux 7.3.1 + U-Boot 2026.04 + TF-A lts-v2.14.2**。
脚本、补丁和配置见 [bsp-rg35xxsp](https://github.com/hsyysy/bsp-rg35xxsp)。

## 获取源码

```bash
git clone --recurse-submodules https://github.com/hsyysy/bsp-rg35xxsp.git
cd bsp-rg35xxsp
```

已有克隆更新后运行：

```bash
git submodule update --init --recursive
```

## 依赖与构建

Debian / Ubuntu（TF-A 需要 GCC 11 或更新版本）：

```bash
sudo apt-get install gcc-aarch64-linux-gnu bc bison flex libssl-dev libelf-dev \
    libgnutls28-dev python3-dev python3-setuptools swig
```

```bash
./scripts/build_tfa.sh
./scripts/build_uboot.sh
./scripts/build_kernel.sh          # 增量编译
./scripts/build_kernel.sh release  # distclean 后全量编译
```

内核可以独立构建；U-Boot 依赖 `out/bl31.bin`。release 模式会清除内核构建配置，
自定义配置需要提前保存。构建脚本会核对源码版本和补丁状态，并每次合并
`configs/rg35xxsp.config`。

## 启动分区与产物

默认双分区布局：p1 为启动分区，p2 为根分区。旧五分区卡需要为 U-Boot 和内核
使用相同参数：

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
| `modules.tar.gz` | 与 7.3.1 内核匹配的模块 |
| `p2-payload/` | 内核、DTB 与 extlinux.conf |
| `SHA256SUMS` | 产物校验值 |

`p2-payload` 应部署到实际启动分区，目录名不决定分区编号。Image、DTB 和模块
必须来自同一次构建。解包模块到带 `/lib` 软链的 rootfs 时，应保留该软链，例如
使用 tar 的 `--keep-directory-symlink`。

U-Boot 使用直接 bootcmd；生成的 `extlinux.conf` 只适用于使用 extlinux 的启动
环境。调整卡布局或日志参数时，需要同时核对实际 bootcmd。当前保留
`fw_devlink=off`，移除前应独立验证设备依赖。

固件和用户态准备见 [rootfs 准备](rootfs-prep.md)。

## 已验证范围

| 子系统 | 状态 |
|---|---|
| 启动链 | TF-A、U-Boot、Linux 7.3.1 启动正常 |
| LCD | 640×480 输出、DE33 平面、背光调光 |
| HDMI | 1080p60 输出与 HDMI 音频 |
| WiFi / USB | RTL8821CS 联网；gadget SSH；Host 模式切换 |
| RTC | PCF8563 内建为 rtc0，开机校时 |
| PMIC / TF2 | 电池与电源读数；TF2 exFAT 挂载 |
| 视频 | H.264 / HEVC / MPEG-2 / VP8 硬件解码 |

蓝牙保持禁用。完整列表和限制见[支持与限制](status.md)，显示细节见
[显示与 DE33](display.md)。

## LCD 控制台输出

fbcon 与前端可能共用 UI 主层。建议解绑 framebuffer vtconsole，并把前端标准
输出接入 journal，避免控制台文本覆盖界面。这是 rootfs / 服务配置，不能只靠
更新内核或降低日志等级代替。

启动告警见[清理记录](boot-warning-cleanup.md)，正常和诊断日志参数见
[启动日志](kernel-debug-branches.md)。
