# 启动日志配置

公开 BSP 将日志参数统一维护在 `configs/rg35xxsp.bootargs`，默认：

```text
drm.debug=0
```

内核启动配置和 U-Boot 直接 bootcmd 使用同一套参数。
`dev` / `release` 只决定构建方式，不决定是否开启详细日志。
正常启动保留标准消息、告警和错误。

## 临时诊断

在公开 BSP 仓库根目录执行：

```bash
BOOT_LOG_ARGS='ignore_loglevel drm.debug=0x2' ./scripts/build_kernel.sh
BOOT_LOG_ARGS='ignore_loglevel drm.debug=0x2' ./scripts/build_uboot.sh
```

恢复正常构建时不设置上述环境变量。
已有 U-Boot 若通过直接 bootcmd 加载内核，单独替换 extlinux.conf 不会清除旧调试参数，
需要核对实际 bootcmd；运行中的 DRM debug 设置也可能影响追踪输出。

此版本不存在 `CONFIG_DRM_DEBUG` / `CONFIG_DRM_DEBUG_DRIVER` 这两个 Kconfig 选项，
应通过实际支持的日志参数控制追踪。

## 验证记录

2026-10-11 的开发设备已通过整机重启复核：启动参数为 `drm.debug=0`，
不含 `ignore_loglevel`；DRM 详细追踪输出为零，已知告警仍保留。
显示路由、USB / WiFi、RTC、TF2 等检查正常。

公开构建脚本默认使用 `builder@buildhost` 作为构建身份，避免产物记录构建机账号和主机名。
内核还通过关闭 `CONFIG_LOCALVERSION_AUTO` 并传入空 `LOCALVERSION`，
保持版本号与模块目录一致；构建身份和版本后缀是两个独立设置。
