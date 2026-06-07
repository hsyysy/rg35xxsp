# 自做 RG35XX-SP rootfs 前的固件 / 数据准备清单

> 用途:将来从原厂 Ubuntu 22.04 (mmcblk0p5) 换到任何自做 rootfs(Alpine / Debian / Buildroot / 自己 squashfs)前,把硬件依赖的固件和设备特定数据先抓出来归档。
>
> 创建:2026-05-20(路线 C 一期跑通后)。当前 mmcblk0p5 是原厂 Ubuntu 22.04,所有固件原文件还在 SD 上未动。

## 一、真硬件固件清单(`/lib/firmware/`)

走主线 + panfrost 路线,**共 4 个文件,其中 3 个上游 + 1 个必须从设备抓**:

| 子系统 | 文件路径 | 来源 | 备注 |
|---|---|---|---|
| WiFi RTL8821CS (SDIO) | `rtw88/rtw8821c_fw.bin` | [linux-firmware] `rtw88/` | 137K |
| BT RTL8821CS (UART1) — firmware | `rtl_bt/rtl8821cs_fw.bin` | [linux-firmware] `rtl_bt/` | 56K |
| **BT RTL8821CS — config** | `rtl_bt/rtl8821cs_config.bin` | **从设备 `/lib/firmware/rtl_bt/rtlbt_config` 抓 + 改名** | **55B,linux-firmware 没收录,btrtl 强制要求否则 hci0 不出** |
| Wireless 监管数据库 | `regulatory.db` + `regulatory.db.p7s` | [wireless-regdb] | `.p7s` 可选(distro 签名后才有) |

[linux-firmware]: https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/
[wireless-regdb]: https://git.kernel.org/pub/scm/linux/kernel/git/sforshee/wireless-regdb.git/

**前三个已在路线 C 一期下载到 `out/firmware/`**(`flash_sd.sh` 会 cp 到 p5)。`rtl8821cs_config.bin` 是 Anbernic 厂家烧的 Realtek 标准格式 config(开头 `55 ab 23 87` magic),实测兼容 mainline btrtl,内容可能含 MAC/RF 校准参数,**不能用别的板子的同名文件替换**。

**主线路线不需要的**:
- Mali GPU(panfrost 开源,**无 firmware blob**)
- libmali blob(仅 BSP 路线需要,我们放弃了)
- Cedrus VPU、Allwinner Crypto Engine、内置音频 codec、内置 USB phy、屏 NV3052C(全部直接 register-level 驱动,无 firmware)

## 二、设备特定运行时数据(不是 firmware 但与硬件绑定)

| 类别 | 来源 | 影响 / 处理 |
|---|---|---|
| **WiFi MAC** | sunxi eFuse(SID),`/sys/bus/nvmem/devices/sunxi-sid0/nvmem` 偏移 ~0xC | RTW88 默认自动从 nvmem 读;读失败 → 随机 MAC,每次 boot 变。**抓 efuse dump 备份**。 |
| **BT MAC** | 同上(通常 WiFi+1 或同一段) | btrtl 同上 |
| **AXP717 电池 OCV-SOC 表** | 已在 DTS(`x-powers,axp717-battery` 节点 11 点表) | 不在 rootfs。**自做 rootfs 不用操心**,只要 DTS 不动 |
| **屏分辨率 / panel 初始化时序** | DTS panel 节点(或 Batocera 的 `.panel` 固件描述文件) | 不在 rootfs。跟着 DTS 走 |
| **音频默认音量 / 路由** | `/var/lib/alsa/asound.state`(systemd `alsa-restore.service` 读) | 运行时状态,第一次手动 `alsamixer` + `alsactl store` 重建即可。**可从原 rootfs 抓**作为初始模板 |
| **背光默认亮度** | `/sys/class/backlight/.../brightness`(systemd-backlight 持久化) | 运行时,自做 rootfs 自动管理 |
| **摇杆 deadzone / 校准** | Anbernic 的 evmapy 配置(`/usr/local/share/evmapy/*.yml`)或类似 | 仅当沿用同款前端;自定义前端用 SDL `joydev` 自己校准 |
| **emuelec / muOS / RetroArch 配置** | `/emuelec/`、`/etc/emuelec/`、`/storage/.config/retroarch/` 等 | 完全用户层,跟你选什么模拟器前端 |

## 三、建议一次性抓取的清单

在 SD 卡挂在主机 `/mnt/usb` 或设备运行时(`/`),跑以下命令归档到 `bsp/device/` 下,以备将来:

```bash
P5=/mnt/usb     # 设备运行则改 P5=

# 1. 整个 /lib/firmware 归档(防上游 linux-firmware 没收录的 vendor blob)
mkdir -p device
tar -C $P5 -czf device/p5-firmware-snapshot.tar.gz lib/firmware
find $P5/lib/firmware -type f | sed "s|^$P5||" > device/p5-firmware.list

# 2. eFuse / SID dump(MAC / 校准来源)只能在设备上跑,主机看 SD 看不到
#    在设备:
hexdump -C /sys/bus/nvmem/devices/sunxi-sid0/nvmem > /tmp/sid-dump.txt
# scp 回主机 → bsp/device/sid-nvmem-dump.txt

# 3. ALSA 音频初始状态
cp $P5/var/lib/alsa/asound.state device/asound.state.orig 2>/dev/null

# 4. Anbernic 特定服务 / 脚本(评估哪些需要复刻)
find $P5/etc/systemd -name "*.service" \
  | xargs grep -lE "anbernic|emuelec|joystick|gamepad|drop_cache" 2>/dev/null \
  > device/anbernic-services.list
find $P5/usr/local -type f > device/anbernic-userlocal.list

# 5. udev / modprobe / 关键配置目录
for d in /etc/udev/rules.d /lib/udev/rules.d /etc/modprobe.d /etc/wpa_supplicant /etc/NetworkManager/conf.d; do
  ls -la $P5$d 2>/dev/null
done > device/key-configs-listing.txt
```

## 四、做 rootfs 时的 minimum viable bundle

按"够用就走"判断,新 rootfs 需要包含:

### A. 必须装入 rootfs 的(非内核模块)

- `/lib/firmware/rtw88/rtw8821c_fw.bin`(已下载到 `out/firmware/`)
- `/lib/firmware/rtl_bt/rtl8821cs_fw.bin`(已下载)
- `/lib/firmware/regulatory.db` + `regulatory.db.p7s`(未下载,**做新 rootfs 时记得加**)
- 内核模块树 `/lib/modules/7.0.9/`(由 `build_kernel.sh` 产出 `modules.tar.gz`)

### B. 推荐装入(否则要手动配)

- `alsa-utils`(运行 `alsactl restore`/`store`)
- `wpa_supplicant` + `iw`(WiFi 管理)
- `bluez`(蓝牙栈,如果要用 BT)
- `udev`(否则 `/dev/input/event*` 权限会乱)

### C. 不要装入(走 mainline 等于不要)

- libmali blob、vendor 8821cs.ko、mali-bifrost kbase(BSP 路线产物)

## 五、相关条目

- `out/firmware/` —— 路线 C 已下载的 3 个固件之 2(WiFi+BT)。**regdb 还差**。
- `issues.md` —— 一期 / 二期问题清单,含 P0-1/P0-2 WiFi/BT 实测进度
- `device-facts.md` —— 设备硬件实测事实(§10 PMIC / §18 WiFi 等)
