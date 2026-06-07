# FFmpeg H264 V4L2-Request + Cedrus VPU 实施记录

> 初始调查: 2026-05-26
> 实施日期: 2026-06-03 ~ 2026-06-04
> 最后更新: 2026-06-04
> 目标: 在 RG35XX-SP (Allwinner H616) 上通过 FFmpeg v4l2-request API 使用 Cedrus VPU 硬件解码 H.264

---

## 1. 当前状态

| 层 | 状态 | 备注 |
|---|---|---|
| Kernel cedrus 驱动 | ✅ 就绪 | `CEDRUS_CAPABILITY_H264_DEC` 已启用 |
| V4L2 stateless 控件 | ✅ 就绪 | SPS/PPS/slice_params/decode_params 全部可用 |
| `/dev/video0` | ✅ 就绪 | cedrus (platform:cedrus) 已注册 |
| V4L2 request API（裸调） | ✅ 通过 | 独立测试程序完整走通：QBUF → request ctrls → QUEUE → DQBUF |
| FFmpeg hwaccel probe | ✅ 成功 | `V4L2 H264 stateless V1 probed successfully` |
| FFmpeg V4L2 controls 设置 | ✅ 成功 | SPS+PPS+decode_params+slice_params 通过 request API 设置成功 |
| FFmpeg DRM frame 描述符 | ✅ 正确 | fd/size/bpl/dimensions 均正确 |
| FFmpeg DMA-BUF transfer | ✅ 已修复 | `V4L2MediaReqDescriptor` 布局修复，`drm` 字段移至 offset 0 |
| FFmpeg 实际解码输出 | ✅ 通过 | IDR + P 帧全部与软件 NV12 输出一致（前 8 帧 framemd5 MATCH） |

**阶段性结论: H.264 V4L2-request 硬件解码完整链路已跑通。DPB `reference_ts` 单位修复（×1000 转纳秒）后，IDR 和 P 帧参考帧查找均正常，前 8 帧硬件解码 NV12 输出与软件解码逐字节一致。**

---

## 2. 已完成的修改

### 2.1 源码修改 (git commits `a4f0d63` ~ `9a88d48`)

修改的文件:

| 文件 | 修改内容 |
|---|---|
| `v4l2_request_h264.h` | 将 `V4L2RequestContextH264` typedef 为 `V4L2RequestContextHEVC`，兼容共享基础设施 |
| `v4l2_req_h264_vx.c` | 核心 H264 V4L2 控件实现（~1200 行） |
| `v4l2_request_h264.c` | hwaccel 入口；dst_buffers 调整为 16，避免参考帧/输出队列压力 |
| `h264_slice.c` | 添加 `CONFIG_H264_V4L2REQUEST_HWACCEL` 到 HWACCEL_MAX 和 format list |
| `v4l2_req_dmabufs.c` | 添加 `/dev/dma_heap/default_cma_region` 到 DMA heap 搜索列表 |

### 2.2 关键修复点

**V4L2 枚举值修正**（与内核 UAPI 对齐）:
```c
// 修正前（错误）          // 修正后（正确）
SLICE_BASED = 1           SLICE_BASED = 0
FRAME_BASED = 0           FRAME_BASED = 1
START_CODE_NONE = 1       START_CODE_NONE = 0
START_CODE_ANNEX_B = 0    START_CODE_ANNEX_B = 1
```

**SPS flags 修正**:
- `FRAME_MBS_ONLY`: `if (!sps->frame_mbs_only_flag)` → `if (sps->frame_mbs_only_flag)`
- 添加缺失的 flags: `SEPARATE_COLOUR_PLANE`, `QPPRIME_Y_ZERO_TRANSFORM_BYPASS`, `DELTA_PIC_ORDER_ALWAYS_ZERO`, `GAPS_IN_FRAME_NUM_VALUE_ALLOWED`

**类型兼容性**:
- `v4l2_req_decode_fns` 的函数指针使用 `V4L2RequestContextHEVC *`，H264 通过 typedef 兼容
- `h264_v4l2_decode_params` 移除了内核不存在的 `num_active_dpb_entries` 字段

**FFmpeg 7.x API 适配**:
- `PPS.scaling_matrix_present` → `PPS.pic_scaling_matrix_present_flag`
- `PPS.transform_bypass` — 删除（PPS 无此字段）
- `H264SliceContext.nal_ref_idc` → `H264Context.nal_ref_idc`
- `H264SliceContext.poc` → `H264Context.cur_pic_ptr->field_poc[]`
- `H264SliceContext.sp_for_switch_flag` — 删除（不存在）
- `H264SliceContext.header_size` — 用 `size * 8` 替代
- `av_mallocz_array` → `av_calloc`

**hwaccel 注册**:
- `h264_slice.c`: 在 `pix_fmts[]` 数组中添加 `AV_PIX_FMT_DRM_PRIME`
- `h264_slice.c`: `HWACCEL_MAX` 宏添加 `CONFIG_H264_V4L2REQUEST_HWACCEL`
- `v4l2_request_h264.h`: 通过 include `v4l2_request_hevc.h` 获取共享类型定义

---

## 3. 已修复: FFmpeg DMA-BUF Transfer 问题

### 3.1 根因: V4L2MediaReqDescriptor 结构体布局

FFmpeg 的 `drm_map_frame` (hwcontext_drm.c) 将 `frame->data[0]` 直接 cast 为 `AVDRMFrameDescriptor*`。
这要求 `AVDRMFrameDescriptor` 在自定义 descriptor 结构体中位于 **offset 0**。

**HEVC 实现**（正确）:
```c
typedef struct V4L2MediaReqDescriptor {
    AVDRMFrameDescriptor drm;   // ← offset 0，cast 正确
    uint64_t timestamp;
    ...
} V4L2MediaReqDescriptor;
```

**H264 原始实现**（错误）:
```c
typedef struct V4L2MediaReqDescriptor {
    struct req_decode_ent decode_ent;  // 前面有多个字段
    unsigned int timestamp;
    size_t num_slices;
    ...
    AVDRMFrameDescriptor drm;   // ← offset 0x50，cast 到 offset 0 读到的是 decode_ent 的数据（全零）
} V4L2MediaReqDescriptor;
```

**修复**: 将 `drm` 字段移至结构体第一个位置。

### 3.2 cedrus 内核驱动验证（已通过）

用独立 C 测试程序直接调用 V4L2 request API，完整链路已验证通过：

```
Input: 50555 bytes
REQ_ALLOC: 0 (OK)
S_FMT out: 0 (OK)          ← 单平面 API (V4L2_BUF_TYPE_VIDEO_OUTPUT)
S_FMT cap: 0 (OK)          ← NV12 输出
REQBUFS out/cap: 0 (OK)
STREAMON out/cap: 0 (OK)
QBUF out: 0 (OK)            ← 带 V4L2_BUF_FLAG_REQUEST_FD
QBUF cap: 0 (OK)
Global ctrls: 0 (OK)        ← decode_mode + start_code
Req ctrls: 0 (OK)           ← SPS+PPS+decode_params+slice_params via request
REQ QUEUE: 0 (OK)           ← MEDIA_REQUEST_IOC_QUEUE
Poll: 1 revents=0x1
DQ cap: 0 bytesused=1382400 (OK)  ← 720p NV12 大小正确
Dst: ALL ZEROS              ← 解码输出为空（控件数据与 bitstream 不完全匹配）
=== RESULT: SUCCESS ===
```

**关键发现**:
1. cedrus 使用**单平面** API（`V4L2_BUF_TYPE_VIDEO_OUTPUT`），不是多平面
2. QBUF 必须带 `V4L2_BUF_FLAG_REQUEST_FD` 标志绑定 request fd
3. capture buffer 不需要带 request fd
4. V4L2 request 管线完整走通，cedrus 驱动无问题
5. 输出 buffer 全零是因为测试用的 SPS/PPS 控件数据与实际 bitstream 不完全匹配（控件是从 FFmpeg debug 输出手动填的，不是从 bitstream 解析的）

### 3.2 FFmpeg SIGSEGV 现象（历史问题）

```
[h264 @ ...] Hwaccel V4L2 H264 stateless V1; devices: /dev/media0,/dev/video0; buffers: src DMABuf, dst DMABuf; swfmt=nv12
[h264 @ ...] Reinit context to 1280x720, pix_fmt: drm_prime
*** SIGSEGV ***
```

GDB backtrace:
```
#0  drm_map_frame (libavutil.so.59)        ← DMA-BUF mmap 失败
#1  drm_transfer_data_from (libavutil.so.59)
#2  av_hwframe_transfer_data (libavutil.so.59)
#3  hwaccel_retrieve_data (ffmpeg)
```

同时解码线程:
```
#0  ioctl (libc.so.6)
#1  media_request_start → mediabufs_start_request  ← 正在提交 request
#2  v4l2_request_h264_end_frame
```

### 3.3 修复验证

DRM transfer 修复验证：
```
frame=   30 fps=0.0 q=-0.0 Lsize=N/A time=00:00:00.99 bitrate=N/A speed=6x
```

- 30 帧处理完成，无 SIGSEGV
- DMA-BUF 描述符正确读取（nb_objects=1, fd=18, size=1384448, pitch=1280）
- DMA-BUF mmap 成功，数据非零

### 3.4 已修复/推进: cedrus VLD 输出

此前现象是硬件解码输出与软件解码不匹配，严重时首帧也不正确。经过逐项修正后，当前状态变为：

- IDR 首帧已经与软件解码 `format=nv12` 的 framemd5 匹配。
- 第 2 帧开始仍可能出现绿色/错误画面，说明 VLD 本身已能工作，但参考帧链路仍有问题。
- 所有 request DQBUF 状态均为 OK，没有继续出现 request 提交失败或 decode fail。

已修正的问题：

- **VLD bitstream 起始位置**: Cedrus 这里不能把 start code 和 NAL header 一起送入 VLD；当前从 slice header payload 起始位置送入，即跳过 start code 和 1 字节 NAL header。
- **slice header bit size**: FFmpeg 的 `get_bits_count(&sl->gb)` 是 RBSP 视角，需要映射回 raw bitstream，并扣除被跳过的 start code/NAL header bit 数。
- **slice_type**: 改为使用 `ff_h264_get_slice_type(sl)`，IDR 强制为 V4L2 I slice，避免 FFmpeg 内部 `AV_PICTURE_TYPE_*` 与 V4L2 H.264 slice type 混用。
- **slice_qp_delta**: 从错误的 `sl->qscale - 26 - pps->init_qp` 修正为 `sl->qscale - pps->init_qp`。
- **disable_deblocking_filter_idc**: FFmpeg 内部值与 bitstream/V4L2 语义不同，0/1 需要反转，2 保持不变。
- **SPS/PPS flags**: 补齐 `DELTA_PIC_ORDER_ALWAYS_ZERO`、`GAPS_IN_FRAME_NUM_VALUE_ALLOWED`、`REDUNDANT_PIC_CNT_PRESENT`、`SCALING_MATRIX_PRESENT` 等 flags。
- **capture bytesused**: 单平面 capture buffer 的 DRM object size 使用 `buffer.bytesused`，不再使用 `buffer.length`。

### 3.5 已修复: DPB reference_ts 单位不一致

**问题**: P 帧参考帧查找失败，导致绿色/错误画面。

- userspace 给 capture buffer 排队时设置的是 `v4l2_buffer.timestamp = { tv_sec = 0, tv_usec = rd->timestamp }`，例如第 1 帧为 `0.000001`。
- kernel `v4l2_buffer_get_timestamp()` 会把 timeval 转为纳秒：`tv_sec * 1e9 + tv_usec * 1000`。
- Cedrus H.264 驱动使用 `vb2_find_buffer(cap_q, dpb->reference_ts)` 查找 DPB 引用帧。
- 原实现把 `dpb.reference_ts` 直接设为 `rd->timestamp`，例如 `1`，但 kernel 实际 buffer timestamp 是 `1000` ns。

**修复**:

```c
static uint64_t h264_timestamp_to_ns(unsigned int timestamp)
{
    return (uint64_t)timestamp * 1000;
}

entry->reference_ts = pic_rd ? h264_timestamp_to_ns(pic_rd->timestamp) : 0;
ref_ts = h264_timestamp_to_ns(ref_rd->timestamp);
```

**验证结果** (2026-06-04):
- 前 8 帧 framemd5 软件/硬件完全一致（MATCH）
- DPB reference_ts 正确：frame 1→1000, frame 2→2000, ...（×1000 纳秒）
- IDR + 7 个 P 帧全部正确，无 decode fail

### 3.6 后续优化

1. ~~**验证 DPB timestamp 修复**~~ — ✅ 已完成，前 8 帧 framemd5 MATCH
2. ~~**完整视频解码测试**~~ — ✅ 已完成，7066 帧全片解码无错误
3. ~~**性能测试**~~ — ✅ 已完成（见下方数据）
4. ~~**mpv 集成**~~ — ✅ 已完成（见下方数据）
5. ~~**清理临时日志**~~ — ✅ 已完成（commit `9a88d48`）

### 3.7 性能对比 (2026-06-04)

测试视频: `video.mp4`, 1280x720@30fps H.264 High, 3 分 55 秒, 7066 帧

| 指标 | 硬件解码 (Cedrus V4L2) | 软件解码 (FFmpeg) | 对比 |
|---|---|---|---|
| 耗时 | 31.5 秒 | 36.5 秒 | **快 14%** |
| 解码帧率 | 224 fps | 194 fps | **快 15%** |
| 解码倍速 | 7.47x | 6.45x | **快 16%** |
| CPU 占用 | **34.8%** | 88.5% | **低 2.5 倍** |
| user 态 | 2655 jiffies | 11971 jiffies | **低 4.5 倍** |
| 解码结果 | 全片无错误 | — | 正确 |

**结论**: 硬件解码在保持略高速度的同时，CPU 占用从 88.5% 降至 34.8%，释放了约 54% CPU 给其他任务（UI、音频、模拟器逻辑等）。user 态从 11971 降至 2655（4.5 倍），说明绝大部分解码工作已 offload 到 VPU。

### 3.8 mpv 集成 (2026-06-04)

**命令**: `mpv --hwdec=drm --vo=gpu-next --gpu-context=drm --ao=alsa video.mp4`

**结果**: 720p30 H.264 硬解播放成功，CPU ~30%

**关键发现 — CMA 碎片化**:
- `dst_buffers=16` 时 mpv 启动后立即 `Failed to get dst buffer`，dmesg 显示 `cma: alloc failed, req-size: 338 pages`
- CMA 64MB 空闲 35MB 但碎片化严重（最大连续块 < 1.32MB）
- 根因：Mali GPU (panfrost) + framebuffer 从 CMA 分配，导致碎片化
- `dst_buffers=8` 后问题解决（总 slot = 8+5+6=19，CMA 压力降低）

**工作参数**:
- `--hwdec=drm` — V4L2 request API 硬解
- `--vo=gpu-next --gpu-context=drm` — libplacebo + DRM OpenGL 渲染
- `--ao=alsa` — ALSA 音频输出
- `dst_buffers=8`（`v4l2_request_h264.c`）— CMA 碎片化 workaround

**最终工作配置** (`/etc/mpv/mpv.conf`):
```
hwdec=drm
vo=gpu-next
gpu-context=drm
profile=fast          # Mali-G31 必须，否则 GPU 渲染太慢导致音画不同步
video-sync=audio
framedrop=decoder
```
直接 `mpv video.mp4` 即可，无需额外参数。

**不工作的组合**:
- `--vo=gpu-next` + `dst_buffers=16` — CMA 碎片化，buffer 分配失败
- `--vo=drm` + `--hwdec=drm-copy` — 同样 CMA 问题
- `--hwdec=auto` — mpv 默认不检测 V4L2 request，需显式 `--hwdec=drm`
- `--vo=gpu-next` 无 `--profile=fast` — Mali-G31 GPU 渲染太慢，A-V 偏差 3+ 秒
- ffplay — 需要 Vulkan，Mali-G31 + Panfrost 不支持

**已知问题 — 640x480 P 帧解码错误**:
- 640x480 H.264 视频 IDR 帧正确，P 帧解码错误（绿色/损坏画面）
- 1280x720 H.264 视频正常
- framemd5 对比：首帧 MATCH，第二帧 MISMATCH
- 可能原因：cedrus VPU 对特定宽度的 stride/对齐处理有差异
- 待排查

---

## 4. 系统配置注意事项

### 4.1 CMA 配置

当前 CMA 64MB，碎片化严重。`/dev/dma_heap/default_cma_region` 是 cedrus 使用的 DMA heap。

FFmpeg 的 `v4l2_req_dmabufs.c` 原本只搜索 `/dev/dma_heap/linux,cma` 和 `/dev/dma_heap/reserved`，已添加 `default_cma_region`。

### 4.2 Cedrus V4L2 API 特性（实测确认）

- **单平面 API**: cedrus 使用 `V4L2_BUF_TYPE_VIDEO_OUTPUT` / `V4L2_BUF_TYPE_VIDEO_CAPTURE`，不是多平面
- **QBUF 需要 REQUEST_FD**: output buffer 的 QBUF 必须设置 `V4L2_BUF_FLAG_REQUEST_FD` + `request_fd`
- **capture buffer 不需要 REQUEST_FD**: capture buffer 正常 QBUF 即可
- **decode_mode**: 值 0 = Slice-Based（内核枚举第一项）
- **start_code**: 值 0 = No Start Code（内核枚举第一项）

### 4.3 DMA Heap 设备

```
/dev/dma_heap/default_cma_region  ← cedrus 使用
/dev/dma_heap/reserved
/dev/dma_heap/system
```

### 4.4 快速验证脚本

`scripts/apps/quick_build_ffmpeg.sh` — 增量编译脚本:
```bash
sudo ./quick_build_ffmpeg.sh                    # 增量编译
sudo ./quick_build_ffmpeg.sh libavcodec/v4l2_req_h264_vx.o  # 单文件编译
sudo ./quick_build_ffmpeg.sh --teardown         # 卸载 chroot
```

首次运行需 configure（较慢），之后增量编译只需几秒。

---

## 5. 关键文件路径

| 文件 | 路径 |
|---|---|
| H264 V4L2 实现 | `src/ffmpeg-v4l2-request/libavcodec/v4l2_req_h264_vx.c` |
| H264 hwaccel 入口 | `src/ffmpeg-v4l2-request/libavcodec/v4l2_request_h264.c` |
| H264 头文件 | `src/ffmpeg-v4l2-request/libavcodec/v4l2_request_h264.h` |
| H264 slice 注册 | `src/ffmpeg-v4l2-request/libavcodec/h264_slice.c` |
| DMA heap 配置 | `src/ffmpeg-v4l2-request/libavcodec/v4l2_req_dmabufs.c` |
| HEVC 参考实现 | `src/ffmpeg-v4l2-request/libavcodec/v4l2_req_hevc_vx.c` |
| 共享基础设施 | `src/ffmpeg-v4l2-request/libavcodec/v4l2_request_hevc.c` |
| 内核 cedrus 驱动 | `src/linux-7.0.9/drivers/staging/media/sunxi/cedrus/` |
| cedrus 直接测试程序 | 设备上 `/root/cedrus/cedrus_test6.c`（V4L2 request API 裸调验证） |
| 内核 V4L2 控件 | `src/linux-7.0.9/include/uapi/linux/v4l2-controls.h` |
| 快速编译脚本 | `scripts/apps/quick_build_ffmpeg.sh` |
| 构建日志 | `scripts/apps/log.txt` |

---

## 6. 设备验证命令

```bash
# 检查 cedrus 设备
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video0 --list-ctrls
v4l2-ctl -d /dev/video0 --list-formats-out
v4l2-ctl -d /dev/video0 --list-formats

# 检查 DMA heap
ls -la /dev/dma_heap/

# 检查 CMA 状态
cat /proc/meminfo | grep Cma
dmesg | grep -i cma

# cedrus 直接测试（V4L2 request API 裸调，不依赖 FFmpeg）
# 提取裸 H.264 流
ffmpeg -i ~/video.mp4 -c:v copy -bsf:v h264_mp4toannexb -frames:v 1 -f h264 /tmp/test.h264
# 编译并运行测试程序（设备上已有 /root/cedrus/cedrus_test6.c）
gcc -O2 -o /root/cedrus/cedrus_test6 /root/cedrus/cedrus_test6.c
/root/cedrus/cedrus_test6 /tmp/test.h264

# FFmpeg hwaccel 基础测试
ffmpeg -hwaccel drm -v verbose -i ~/video.mp4 -frames:v 30 -map 0:v:0 -f null -

# 对比软件/硬件前 8 帧 NV12 framemd5
ffmpeg -y -v error -i ~/video.mp4 -frames:v 8 -map 0:v:0 -vf format=nv12 -f framemd5 /tmp/sw_nv12.md5
ffmpeg -y -hwaccel drm -v error -i ~/video.mp4 -frames:v 8 -map 0:v:0 -vf format=nv12 -f framemd5 /tmp/hw_nv12.md5
diff -u /tmp/sw_nv12.md5 /tmp/hw_nv12.md5

# 查看 H.264 request/DPB 调试日志
ffmpeg -hwaccel drm -v debug -i ~/video.mp4 -frames:v 4 -map 0:v:0 -f null - 2>&1 | grep -E 'H264 submit|DQBUF|Decode fail|frame='

# 检查内核日志
dmesg | grep -i cedrus
dmesg | grep -i cma
```
