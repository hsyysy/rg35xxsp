# RG35XX SP GPU / GLES 测试总结

> 2026-06 历史测试记录，不能替代当前 7.2.9 的完整 GPU 验收。驱动节点编号与 Mesa 能力需按目标系统重新检查。

设备：RG35XX-SP（Allwinner H700 / H616 系列，Mali-G31，Panfrost）

## 硬件信息

| 项目 | 详情 |
|------|------|
| SoC | Allwinner H700（H616 系列） |
| GPU | Mali-G31 MC1 (Bifrost gen3) |
| 驱动栈 | 内核 Panfrost + Mesa 26.1.2（历史测试环境） |
| DRM 节点 | 按驱动枚举区分 sun4i-drm 与 Panfrost，不固定 card 编号 |
| 显示输出 | RGB LCD，640×480，XRGB8888 |

## 支持情况

| 功能 | 状态 | 说明 |
|------|------|------|
| OpenGL ES 3.1 | ✅ 正常 | Mesa Panfrost 驱动, GLSL ES 3.10 |
| EGL 1.5 | ✅ 正常 | GBM 后端, headless 和 DRM 输出均可 |
| DRM/KMS 直接渲染 | ✅ 正常 | 通过 GBM surface + EGL window surface |
| Vulkan | ❌ 不可用 | PanVK 不支持 Mali-G31 (Bifrost) |
| 硬件视频解码 (VA-API) | ❌ 不可用 | Panfrost 无 drv_video.so |
| OpenCL | ❌ 不可用 | Mali-G31 不支持 |

## 已安装的 GPU 相关包

```
mesa                 # OpenGL ES 驱动 (panfrost_dri.so, panthor_dri.so)
libdrm               # DRM 内核接口
libglvnd             # GL/EGL dispatch
libvulkan            # Vulkan 库 (mpv 依赖链)
vulkan-icd-loader    # Vulkan loader (mpv 依赖链)
spirv-tools          # SPIR-V 工具 (mesa 依赖)
mesa-utils           # eglinfo 等诊断工具
glu                  # OpenGL Utility 库
```

已卸载 (Mali-G31 无 Vulkan):
```
vulkan-headers
vulkan-panfrost
vulkan-mesa-implicit-layers
```

## DRM 输出注意事项

1. 先确认 sun4i-drm 所属节点；下面的 card1 仅为示例，编号不固定：
   ```c
   fd = open("/dev/dri/card1", O_RDWR);
   ```

2. EGL 选择的 config 原生格式可能是 RGB565 或 XR30, 需要遍历 config 找到匹配显示格式 (XRGB8888 = 0x34325258) 的:
   ```c
   for (int i = 0; i < nc; i++) {
       EGLint v;
       eglGetConfigAttrib(dpy, cfgs[i], EGL_NATIVE_VISUAL_ID, &v);
       if ((uint32_t)v == 0x34325258) { cfg = cfgs[i]; break; }
   }
   ```

3. GBM surface 创建时必须用匹配的格式:
   ```c
   gbm_surface_create(gbm_dev, w, h, 0x34325258,
                      GBM_BO_USE_SCANOUT | GBM_BO_USE_RENDERING);
   ```

4. drmModeAddFB2 的格式也要一致:
   ```c
   drmModeAddFB2(fd, w, h, 0x34325258, handles, strides, offsets, &fb_id, 0);
   ```

## GLES 示例

早期测试曾观察到 uniform / UBO 使用问题，尚不能据此判断整个驱动不支持这些功能。
`glGetUniformLocation` 返回 0 是有效位置，错误应检查 -1。
下面保留当时用于隔离问题的内联矩阵示例。

示例 (内联旋转+透视):
```glsl
#version 300 es
uniform float u_t;
layout(location=0) in vec3 a_pos;
layout(location=1) in vec3 a_col;
out vec3 v_col;
void main() {
    float cy = cos(u_t),     sy = sin(u_t);
    float cx = cos(u_t*0.7), sx = sin(u_t*0.7);
    vec3 p = vec3(cy*a_pos.x + sy*a_pos.z,
                  a_pos.y,
                  -sy*a_pos.x + cy*a_pos.z);
    p = vec3(p.x, cx*p.y - sx*p.z, sx*p.y + cx*p.z);
    p.z -= 4.0;
    float n = 0.1, f = 50.0, fov = 1.0;
    float t2 = tan(fov * 0.5);
    float a = 640.0 / 480.0;
    gl_Position = vec4(p.x/(a*t2), p.y/t2,
                       (-(f+n)/(f-n))*p.z + (-(2.0*f*n)/(f-n)),
                       -p.z);
    v_col = a_col;
}
```

## 编译 GLES 程序

```bash
gcc -D_GNU_SOURCE -I/usr/include/libdrm -o drm_gles drm_gles.c \
    -lEGL -lGLESv2 -lgbm -ldrm -lm
```



## 诊断命令

```bash
eglinfo                          # EGL 配置和扩展信息
cat /sys/class/drm/card*/*/status          # 连接器状态
cat /sys/class/drm/card*/*/modes           # 支持的分辨率
cat /sys/kernel/debug/dri/0/name           # DRM 驱动信息
dmesg | grep -i panfrost                   # GPU 初始化日志
VK_LOADER_DEBUG=all ./program 2>&1        # Vulkan loader 调试
```
