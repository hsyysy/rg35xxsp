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

驱动为每个通道维护独立的 scaler 类型和 scanline 上限。VI ch1/ch2（VSU8）的
YUV 输出目前偏绿，暂不作为可用路径。

VI plane 在 LCD 上公布 16 个 YUV 格式：

```text
NV12 NV21 NV16 NV61 P010 P210
UYVY VYUY YUYV YVYU
YU12 YV12 YU16 YV16 YU11 YV11
```

这些格式已按 320×240 → 640×480、1:1 居中、640×480 → 320×240 三种几何验证
KMS 提交。颜色验收使用 `de33-atomic`（CMA DMA-BUF + atomic）：16 个格式
1:1 颜色全部正常，12 个 non-packed 格式缩小颜色正常；packed 4:2:2 双向
缩小不支持。`de33-yuv` 的 dumb-buffer/legacy 路径不作为颜色依据。
UI 与 VI 的 RGB 2× 缩放已目视正常。

## HDMI

- mini-HDMI（HDMI-C）接口，DWC HDMI 2.0 TX + H616 PHY。
- EDID 可读（256 字节）；首选 3840×2160@60（594 MHz），同时支持 4K30、
  1440p 和 1080p。
- 4K60 已实测出图：RGB 主平面 3840×2160@60 彩条颜色与顺序正常，自动模式
  按 EDID 首选 4K60。切换模式时闪一次属链路重新训练。
- LCD 菜单与 HDMI 4K60 可同时工作：LCD TCON 固定在 `PLL_VIDEO1`，HDMI TCON
  使用 `PLL_VIDEO0`。
- 4K60 静态演示偶发 1–2 次闪烁/撕裂，4K30 未观察到；链路余量仍需观察。
- HDMI 音频由 H616 AHUB DAM + port 1 接到 DW-HDMI I2S codec，注册为独立的
  HDMI 声卡。
- LCD 与 HDMI 可同时工作。

HDMI 的 VI ch1/ch2（VSU8）YUV 输出颜色偏绿，暂不作为可用路径；4K60 结论只
覆盖 RGB 主平面。HDMI CEC/HDCP 尚未验收。

## 限制

- HDMI 上的 VI ch1/ch2 使用 VSU8，需要接 HDMI sink 才能做 YUV 扫描输出验收。
- 没有 DE33 writeback，无法回读 mixer 合成后的像素；颜色结论来自目视。
- 缩小路径：16 个格式 1:1 和 12 个 non-packed 格式缩小颜色正常；
  packed 4:2:2 双向缩小不支持，驱动返回 `-ERANGE`，1:1 和单方向缩小正常。
