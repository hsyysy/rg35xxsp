# 启动日志配置

日志参数统一维护在 `configs/rg35xxsp.bootargs`，默认：

```text
drm.debug=0
```

U-Boot 直接 bootcmd 和内核启动参数使用同一套配置。`dev` / `release` 只决定
构建方式，不决定日志级别。正常启动保留标准消息、告警和错误。

## 临时诊断

在 BSP 仓库根目录：

```bash
BOOT_LOG_ARGS='ignore_loglevel drm.debug=0x2' ./scripts/build_kernel.sh
BOOT_LOG_ARGS='ignore_loglevel drm.debug=0x2' ./scripts/build_uboot.sh
```

恢复正常构建时不设置这些变量。如果 U-Boot 使用直接 bootcmd，单独替换
`extlinux.conf` 不会清除旧参数，需要核对实际 bootcmd。

7.3.1 没有 `CONFIG_DRM_DEBUG` / `CONFIG_DRM_DEBUG_DRIVER` Kconfig 选项，
调试追踪应通过 `drm.debug` 参数控制。

## 稳定版本号

BSP 默认使用 `builder@buildhost` 作为构建身份，避免产物记录构建机账号和
主机名。内核关闭 `CONFIG_LOCALVERSION_AUTO` 并传入空 `LOCALVERSION`，
使 `kernelrelease` 和 `/lib/modules` 目录保持为 `7.3.1`。
