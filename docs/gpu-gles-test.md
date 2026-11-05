# GPU / GLES

## 硬件与驱动

| 项目 | 值 |
|---|---|
| GPU | Mali-G31 MP1（Bifrost） |
| 内核驱动 | Panfrost |
| 用户态 | Mesa（OpenGL ES / EGL） |
| 显示 | 640×480 RGB LCD，XRGB8888 |

Panfrost 需要 IOMMU 和 H616 PRCM PPU，二者已在 BSP 内核配置中启用。

## 支持情况

| 功能 | 状态 |
|---|---|
| OpenGL ES 3.1 | 可用 |
| EGL 1.5 / GBM | 可用 |
| DRM/KMS 直接渲染 | 可用 |
| Vulkan | 不可用（PanVK 不支持 Mali-G31） |
| OpenCL | 不可用 |
| VA-API 视频解码 | 不可用；视频硬解走 Cedrus / V4L2 Request |

## DRM 输出

不要硬编码 DRM 卡号。先确认 `sun4i-drm` 和 Panfrost 的枚举顺序。

EGL 配置的原生格式可能是 RGB565 或 XRGB2101010。使用 LCD 时应选择
XRGB8888（`DRM_FORMAT_XRGB8888`，数值 `0x34325258`）：

```c
for (int i = 0; i < nc; i++) {
    EGLint v;
    eglGetConfigAttrib(dpy, cfgs[i], EGL_NATIVE_VISUAL_ID, &v);
    if ((uint32_t)v == DRM_FORMAT_XRGB8888) {
        cfg = cfgs[i];
        break;
    }
}
```

GBM surface 和 `drmModeAddFB2` 必须使用同一格式：

```c
gbm_surface_create(gbm_dev, w, h, DRM_FORMAT_XRGB8888,
                   GBM_BO_USE_SCANOUT | GBM_BO_USE_RENDERING);
```

## 编译

```bash
gcc -D_GNU_SOURCE -I/usr/include/libdrm -o drm_gles drm_gles.c \
    -lEGL -lGLESv2 -lgbm -ldrm -lm
```

## 诊断

```bash
eglinfo
cat /sys/class/drm/card*/*/status
cat /sys/class/drm/card*/*/modes
cat /sys/kernel/debug/dri/*/name
dmesg | grep -i panfrost
```

`glGetUniformLocation` 返回 0 是有效位置，错误值是 -1。
