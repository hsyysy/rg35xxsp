# 支持与限制

本页汇总当前 BSP 的验收范围。具体实现见[显示与 DE33](display.md)、
[视频硬解](video.md)和[主线内核指南](mainline-guide.md)。

## 已验证

| 项目 | 结果 |
|---|---|
| 启动链 | TF-A BL31、U-Boot 2026.04、Linux 7.3.1 正常启动 |
| LCD | 640×480 输出；左右边缘回绕已修复；1 px 棋盘不再出现绿边和残影 |
| DE33 显示 | UI / VI 平面、RGB 缩放；16 个 YUV 格式 1:1 颜色全部正常，12 个 non-packed 格式缩小正常 |
| HDMI | mini-HDMI 1080p60 与 4K60（RGB 主平面）输出；H616 AHUB HDMI 音频；LCD 与 HDMI 可同时工作 |
| 音频 | 扬声器与耳机输出；插耳机时关闭扬声器 PA；音量键有效 |
| 背光 | PWM0 50 kHz 连续调光，菜单 5%–100%，重启后保持 |
| 电源 | AXP717 PMIC、电池信息、电源键、合盖熄屏与开盖恢复 |
| 看门狗 | H616 WDT，systemd 16s 喂狗；1s 停喂实测约 25s 自动复位 |
| RTC | PCF8563 固定为 rtc0，开机由硬件时钟校时 |
| WiFi | RTL8821CS SDIO，mainline `rtw88` 驱动 |
| USB | 默认 gadget（CDC-ECM / SSH）；运行时切换到 Host |
| 存储 | TF1 启动 / 根分区；TF2 exFAT 用户卡 |
| 刷写 | 整卡/增量/分项更新，自动备份、校验与 `--rollback` |
| 视频 | H.264、HEVC Main 8-bit、MPEG-2、VP8 硬件解码；mpv `drmprime` 播放 |
| 输入 | 15 个 gamepad 键、音量键、电源键；按键测试与菜单交接正常 |

## 待办与未覆盖

| 项目 | 说明 |
|---|---|
| HDMI 4K60 稳定性 | 4K60 已出图；曾偶发 1–2 次闪烁/撕裂，2026-11-09 15s 复测未观察到，4K30 正常；仍待长时间复测 |
| HDMI 输出模式选择 | 计划在菜单中增加：自动探测 EDID 可用模式，并允许手动指定 |
| HDMI CEC / HDCP | 未验收 |
| HDMI VI ch1/ch2 | 这两层使用 VSU8；驱动已限制为只公布 RGB，YUV 请求会被拒绝，HDMI 视频走 RGB/GPU 路径 |
| DE33 缩小路径 | 12 个 non-packed 格式缩小正常；packed 4:2:2 双向缩小不支持，驱动返回 `-ERANGE`，1:1 和单方向正常 |
| 视频长时播放 | 短时播放、掉帧和音画同步已通过；长时间热稳定和功耗仍需观察 |
| 上游 VE IOMMU/SRAM | 直接采用上游 H616 VE IOMMU/SRAM 配置会触发页错误和 request 超时，当前未纳入 |
| 开机图 | U-Boot 尚未启用显示和 splash，内核 logo 也未接入 |
| 蓝牙 | 禁用；RTL8821CS 组合芯片的 BT 初始化会影响 WiFi |
| KASLR | 未启用，属于已接受的限制 |
| 模拟摇杆 | 本机没有模拟摇杆，不适用 |

## 说明

- 没有 DE33 writeback，也没有视频采集设备，mixer 合成后的颜色只能目视确认。
  `kmsgrab -plane_id` 只能读取单个 plane 的 buffer。
- HDMI 音量由电视或应用控制；H616 模拟 codec 的音量键不控制 HDMI codec。
- 角色切换不持久化，重启后回到 DTS 默认的 USB gadget 模式。
