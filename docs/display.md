# 显示与 DE33

## LCD

- 3.5 英寸 640×480 RGB888 IPS，NV3052C 控制器。
- 厂商面板代号 `fog_fj035fhd05_v1`，主线 compatible 为
  `anbernic,rg35xx-sp-panel`。
- 背光由 PWM0 驱动，频率 50 kHz，支持 5%–100% 连续调光。
- TCON TOP 路由和 H616 LCD 时序已修正，正常启动后左右边缘不再回绕。
- 面板声明 6 bpc 并启用 TCON FRM；1 px 棋盘测试不再出现绿边或残影。

## DE33 平面与缩放

| 通道 | scaler |
|---|---|
| VI ch0 | VSU10 + edge detection |
| VI ch1/ch2 | VSU8 |
| UI ch6/ch7/ch8 | VSU8 |

驱动为每个通道维护独立的 scaler 类型和 scanline 上限。只有通道具备 scaler 时
才公布 YUV 格式，避免用户态选中无法做色度上采样的格式。

VI plane 在 LCD 上公布 16 个 YUV 格式：

```text
NV12 NV21 NV16 NV61 P010 P210
UYVY VYUY YUYV YVYU
YU12 YV12 YU16 YV16 YU11 YV11
```

这些格式已按 320×240 → 640×480、1:1 居中、640×480 → 320×240 三种几何验证
KMS 提交。颜色验收使用 mpv 的 DMA-BUF 路径：NV12 1280×960 → 640×480
缩小颜色正常。`de33-yuv` 的 dumb-buffer/legacy 路径不作为颜色依据。
UI 与 VI 的 RGB 2× 缩放已目视正常。

## HDMI

- mini-HDMI（HDMI-C）接口，DWC HDMI 2.0 TX + H616 PHY。
- 默认输出 1080p60。板级 DTS 用 `max-tmds-clock = <200000>` 过滤更高时钟的
  模式，因此 4K30/4K60 不会自动选中。
- HDMI 音频由 H616 AHUB DAM + port 1 接到 DW-HDMI I2S codec，注册为独立的
  HDMI 声卡。
- LCD 与 HDMI 可同时工作。

4K60 在当前线材和转接器组合下没有出图，不能据此判断 H616 PHY 不支持；
需要更换高质量 mini-HDMI 线后，删除 `max-tmds-clock` 再验证。HDMI CEC/HDCP
尚未验收。

## 限制

- HDMI 上的 VI ch1/ch2 使用 VSU8，需要接 HDMI sink 才能做 YUV 扫描输出验收。
- 没有 DE33 writeback，无法回读 mixer 合成后的像素；颜色结论来自目视。
- 缩小路径已用 mpv NV12 2× 缩小确认；16 格式逐项颜色和 packed 4:2:2
  双向缩小仍待用真实客户端复核。
