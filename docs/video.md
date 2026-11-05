# 视频硬解

## 支持范围

H616 的 Cedrus VPU 通过 V4L2 Request API 提供 stateless 硬件解码。当前 BSP
包含 cedrus 的 H616 支持、64 字节 NV12 线性 stride 对齐和 128 MiB CMA。

| 编码 | 状态 | 备注 |
|---|---|---|
| H.264 | 硬件解码 | 8-bit 4:2:0；覆盖单/多 slice、MBAFF、PAFF、加权预测 |
| HEVC | 硬件解码 | Main 8-bit；tiles、多 slice、CRA seek、场编码 |
| MPEG-2 | 硬件解码 | 8-bit 4:2:0 frame/field；PSNR 62–74 dB |
| VP8 | 硬件解码 | profile 0–3；altref、sign bias、分辨率切换 |
| HEVC 10-bit | 软件回退 | H616 内核能力拒绝 |
| H.264/HEVC 4:2:2、4:4:4 | 软件回退 | 硬件不支持 |

## 用户态

- FFmpeg 使用带 v4l2-request 支持的构建，Cedrus 输出 NV12 DMA-BUF。
- mpv 可使用 `--hwdec=drm --gpu-hwdec-interop=drmprime` 做零拷贝播放。
- `drmprime-overlay` 可让视频直接走 VI plane；是否设为默认取决于 OSD、字幕和
  多窗口需求。
- 需要 `/dev/dma_heap/system`、`/dev/dma_heap/linux,cma` 和足够的 CMA。

## 播放验证

短时播放已验证 640×480、720×540、1280×720 的 H.264，以及 720p HEVC，
硬解、音画同步和掉帧正常。720 宽度曾因 Cedrus 的 stride 要求出现参考帧
错误，已通过 64 字节对齐修复。

## 限制

- 长时间播放的热稳定、功耗和内存碎片仍需观察。
- 上游 H616 VE IOMMU/SRAM 配置直接加入会触发 DE/VE_R 页错误和 request
  超时，当前未纳入 BSP。
- 1080p 解码依赖 128 MiB CMA；碎片化严重时仍可能分配失败。
- 10-bit、4:2:2 和 4:4:4 内容由软件解码，CPU 占用会明显升高。
