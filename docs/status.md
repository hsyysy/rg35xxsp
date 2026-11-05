# 支持与限制

本页汇总当前 BSP 的验收范围。具体实现见[显示与 DE33](display.md)、
[视频硬解](video.md)和[主线内核指南](mainline-guide.md)。

## 已验证

| 项目 | 结果 |
|---|---|
| 启动链 | TF-A BL31、U-Boot 2026.04、Linux 7.3.1 正常启动 |
| LCD | 640×480 输出；左右边缘回绕已修复；1 px 棋盘不再出现绿边和残影 |
| DE33 显示 | UI / VI 平面、RGB 缩放；16 个 YUV 格式在 LCD 上逐项显示正常 |
| HDMI | mini-HDMI 1080p60 输出；H616 AHUB HDMI 音频 |
| 音频 | 扬声器与耳机输出；插耳机时关闭扬声器 PA；音量键有效 |
| 背光 | PWM0 50 kHz 连续调光，菜单 5%–100%，重启后保持 |
| 电源 | AXP717 PMIC、电池信息、电源键、合盖熄屏与开盖恢复 |
| RTC | PCF8563 固定为 rtc0，开机由硬件时钟校时 |
| WiFi | RTL8821CS SDIO，mainline `rtw88` 驱动 |
| USB | 默认 gadget（CDC-ECM / SSH）；运行时切换到 Host |
| 存储 | TF1 启动 / 根分区；TF2 exFAT 用户卡 |
| 视频 | H.264、HEVC Main 8-bit、MPEG-2、VP8 硬件解码；mpv `drmprime` 播放 |
| 输入 | 15 个 gamepad 键、音量键、电源键；按键测试与菜单交接正常 |

## 待办与未覆盖

| 项目 | 说明 |
|---|---|
| HDMI 4K | 默认限制在 200 MHz TMDS，只输出 1080p60；4K30/4K60 需要换线后单独验证 |
| HDMI CEC / HDCP | 未验收 |
| HDMI VI ch1/ch2 | 这两层只在 HDMI CRTC 上可用，尚未接 HDMI sink 做 YUV 扫描输出验收 |
| DE33 缩小路径 | 640×480 → 320×240 只验证了 KMS 提交；没有 writeback，无法自动比对合成像素 |
| 视频长时播放 | 短时播放、掉帧和音画同步已通过；长时间热稳定和功耗仍需观察 |
| 上游 VE IOMMU/SRAM | 直接采用上游 H616 VE IOMMU/SRAM 配置会触发页错误和 request 超时，当前未纳入 |
| 刷写流程 | 缺少自动备份、校验和回滚；`--incremental` 之外没有独立的分项更新模式 |
| 开机图 | U-Boot 尚未启用显示和 splash，内核 logo 也未接入 |
| 蓝牙 | 禁用；RTL8821CS 组合芯片的 BT 初始化会影响 WiFi |
| KASLR | 未启用，属于已接受的限制 |
| 模拟摇杆 | 本机没有模拟摇杆，不适用 |

## 说明

- 没有 DE33 writeback，也没有视频采集设备，mixer 合成后的颜色只能目视确认。
  `kmsgrab -plane_id` 只能读取单个 plane 的 buffer。
- HDMI 音量由电视或应用控制；H616 模拟 codec 的音量键不控制 HDMI codec。
- 角色切换不持久化，重启后回到 DTS 默认的 USB gadget 模式。
