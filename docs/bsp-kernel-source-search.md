# **Anbernic RG35XX-SP Allwinner H700 BSP 内核源码搜寻与合规性深度技术报告**

> **来源等级声明（2026-05-20 加注）**
>
> 本文档由 LLM agent 基于**互联网公开搜索**生成，引证均来自互联网公开渠道（GitHub 代码搜索 / Gist / Reddit / 技术博客 / 司徒教學網站 / postmarketOS Wiki / RockNIX / 全球搜索引擎结果等，详见文末 39 条编号引用）——**不是设备实测，也不是 Allwinner 官方 PDF**。
>
> 按 `device-facts.md` 文档头确立的来源可信度规则，互联网搜索结果属"默认不可信"层级；本报告**全文仅作背景调研与索引参考**，不得作为硬件事实依据。
>
> **凡是与设备硬件实际行为相关的事实**（PMIC 驱动栈、Mali 版本、allen 标记位置、DTB 三选一、boot.img 格式、电池/USB 供电、thermal、内存映射、Anbernic 内核版本 #11/#241 等），均以 `device-facts.md`（设备实测 + 串口日志 + 官方 PDF）为准。本文档与 device-facts.md 表述不一致时，**以 device-facts.md 为准**。
>
> **本报告中已知与 device-facts.md 重叠或与之冲突的部分**（仅列已知项）：
> - "Mali-G31 MP2" —— 来自 H700 brief 上限标称；**实测为 MP1**，见 `device-facts.md §2` 与 §10.1
> - "AXP2202 [allen] 调试标记...由 Anbernic 内部工程师 Allen ... 本地修改" —— 部分正确，但具体出现位置见 `device-facts.md §10.4`（设备 DT 中无 allen* 节点，allen 字符串以 U-Boot/Kernel printk 形式出现于 4 处具名场景，且贯穿 LCD 探测/boot0 下载/earlycon LCD 选择/AXP PEK 探测）
> - "京东方 BOE 面板与老款面板" —— device-facts.md §7 / §23 只能确认 `lcd_type=old → dtb 0 (fog_fj035fhd05_v1)`；本报告中 "BOE 面板"称谓未经设备/官方 PDF 验证
> - "霍尔效应磁吸开关" —— device-facts.md 中无 hall sensor 实测条目；mainline DTS 中只有 GPIO `PE7` lid switch（见 `mainline-support.md` RG35XX-SP DTS 节）
> - "AXP2202 = 安伯尼克掌机专用 PMU" —— 该 IC 在 mainline 中识别为 AXP717，BSP 私有驱动栈与 mainline 关系详见 `device-facts.md §10`
> - 内核版本 `#241` (2024-04-11) 与 hostname `cc@cc-H81M-S1`：来自互联网渠道（GitHub Gist 等）公开的**其他 H700 设备**串口日志，与本机无关；本设备实测 hostname=`ANBERNIC`、build=`#11` 2025-12-24 `flower@flower-B85M-D2V`（见 `device-facts.md §25` + 本机 `scripts/serial.log`）
> - hostname `deeplay` 同理：仅出现在互联网公开的他人串口日志中；本机串口实测为 `ANBERNIC`，无 `deeplay` 字串。device-facts.md §10.5 仅记录"该字符串在公网存在且推测与 Deeplay-Keys 中间件有关"，未做进一步断言
> - HandsomeMod 树缺驱动情况见 `verification-report.md`，更具体
> - mainline 源码树状况见 `mainline-support.md`
>
> 报告原文保留如下，未做改动。

---

## **编译日志指纹与硬件环境鉴定**

针对 Anbernic RG35XX-SP 掌机所搭载的全志（Allwinner）H700 / sun50iw9p1 处理器，对其版本号为 4.9.170 的 Linux 内核源码进行深度搜寻、鉴定与技术对齐分析。分析起点基于一份来自该掌机串口的真实 UART 启动日志：  
Linux version 4.9.170 (cc@cc-H81M-S1) (gcc version 5.3.1 20160412 (Linaro GCC 5.3-2016.05)) \#241 SMP PREEMPT Thu Apr 11 12:49:34 CST 2024 1  
该编译特征及相关上下文为原厂软件开发环境提供了明确的指纹画像：

* **编译主机与用户特征**：编译用户为 cc，主机名为 cc-H81M-S1 1。其中，H81M-S1 是技嘉（Gigabyte）在 2014 年前后推出的一款基于 LGA1150 插槽、搭载 Intel Haswell 架构芯片组的经典消费级主板。此硬件配置表明，该内核并非由大型企业级高配编译工作站构建，而是由国内掌机行业常见的 ODM 方案商或独立外包开发人员，使用一台服役时间较长的个人 DIY 电脑作为编译终端进行本地构建。  
* **设备与项目代号**：系统默认主机名或底层项目标识符为 deeplay（或其变体形式） 2。该词在复古掌机软件生态中通常与底层输入外设映射层或菜单中间件（如 Deeplay-Keys 虚拟键盘及映射驱动）紧密关联，指向了专门为安伯尼克（Anbernic）提供软件解决方案的开发实体 2。  
* **迭代与时间戳分析**：构建序列号为 \#241，证明该代码分支在本地经历了频繁的集成与回归测试 1。编译执行的具体时间为 2024 年 4 月 11 日 CST（中国标准时间）12:49:34 1。

在全志 H700 系列掌机产品线中，该编译环境呈现出高度集中的统一特征：

* 安伯尼克 RG28XX 掌机的串口启动日志捕获到了于同一天下午构建的第 \#173 版内核，其编译用户及主机同样为 cc@cc-H81M-S1 5。  
* RG35XX Plus 掌机的启动日志中则记录了编译于 2024 年 3 月 12 日的第 \#212 版内核，其指纹特征同样与该主机完全契合 6。

上述指纹重合度确证了安伯尼克 H700 系列产品的 BSP 内核是由同一个本地编译环境、基于同一套高度耦合的全志 4.9.170 软件开发包（SDK）进行分支派生与编译生成的 1。

## **源码检索路径与全网命中率审计**

为了寻找可能公开托管该原厂 BSP 内核源码的存储库，本研究对全球主流开源平台、代码托管站及复古掌机技术社区执行了全路径检索。排查逻辑依次覆盖了 GitHub、Gitee、Google/DuckDuckGo、国内技术社区及国外复古硬件论坛。检索策略与命中细节如表所示：

| 检索阶段 | 平台与渠道 | 具体检索关键词 / 语句 | 检索结果与命中数分析 |
| :---- | :---- | :---- | :---- |
| **第一阶段** | GitHub Code Search | "sun50i-h700-rg35xx-sp" extension:dts | **零命中**。没有任何公共仓库托管该设备树源文件。 |
|  | GitHub Code Search | "sun50i-h700-rg35xx-sp.dts" | **零命中**。仅在第三方开发者的日志备份 Gist 页面中存在文件名引用 1。 |
|  | GitHub Code Search | "cc@cc-H81M-S1" | **零代码命中**。仅在记录 UART 启动日志的文本 Gist 中有 3 处文本匹配 1。 |
|  | GitHub Code Search | "sun50iw9p1" "4.9.170" | **部分文本命中**。检索结果均为全志 H616 引导日志、香橙派旧分支配置说明，未检索到 RG35XX-SP 的活跃内核源码树 8。 |
|  | GitHub Code Search | "x-powers,axp2202" "sun50iw9p1" | **零命中**。未在任何可编译的内核驱动目录中发现该特定的电源管理集成电路（PMIC）与 SoC 节点组合 12。 |
|  | GitHub Code Search | path:\*\*/drivers/video/fbdev/sunxi/disp2 "rg35xx" | **零命中**。显示引擎核心驱动路径下没有任何针对该设备族的定制代码。 |
| **第二阶段** | GitHub Repository | linux-4.9 rg35xx | **零仓库命中**。结果均为第三方固件（如 Knulli、MuOS）用于拉取 vanilla 4.9 内核进行树外模块编译的辅助脚本，并非原厂内核 13。 |
|  | GitHub Repository | rg35xx-sp kernel | **零仓库命中**。未检索到符合条件的 BSP 内核主存储库。 |
|  | GitHub Repository | h700 linux source | **零仓库命中**。发现 mporrato/alpine-h700 等项目，但其明确指出仅使用原厂二进制固件包（Binary Blobs）重新打包，未拥有源码 16。 |
|  | GitHub 组织页 | 检索 Anbernic / Anbernic-Official / anbernic-rg | **零命中**。官方组织账号下未公开任何针对 H700 芯片组的 U-Boot 或内核源码库 18。 |
| **第三阶段** | Gitee | RG35XX SP 内核 / h700 linux 4.9 | **零命中**。国内镜像及自建库中未发现相关的内核编译工程。 |
|  | Gitee | sun50iw9p1 / deeplay | **零源码命中**。仅发现极少数编译好的二进制应用移植工程（如 DinguxCommander 移植版） 20。 |
| **第四阶段** | 全球搜索引擎 | "sun50i-h700-rg35xx-sp.dts" site:github.com OR site:gitee.com | **零命中**。仅导向 Macromorgan 的设备树 blob 提取日志，未指向任何源码存储库 1。 |
|  | 全球搜索引擎 | "sun50iw9p1" "4.9.170" inurl:linux | **高命中**。全部导向香橙派（Orange Pi）官方维护的 orange-pi-4.9-sun50iw9 分支 21。 |
|  | 全球搜索引擎 | Anbernic GPL source kernel RG35XX SP / "rg35xx sp" GPL linux source | **高文本命中**。大量结果均指向技术社区对安伯尼克未能履行 GPL 开源义务的声讨与讨论 18。 |
| **第五阶段** | 国内社区站 | 知乎、知乎、贴吧、Bilibili 检索内核编译与 H700 自编译 | **零源码命中**。国内开发者普遍采用逆向反编译 DTB、提取固件二进制 .ko 模块或移植主线内核来绕过原厂限制 17。 |
| **第六、七阶段** | Retro Handhelds | Reddit、Dingoonity、4PDA、Discord 频道检索 | **高讨论命中**。社区共识确认：安伯尼克官方未向任何开发者或普通用户提供 RG35XX-SP 的内核源码，其行为属于典型的 GPL 违规 18。 |

审计结果表明，**目前在公共网络空间中，不存在任何可用于直接克隆并编译的 Anbernic RG35XX-SP 原厂 4.9.170 内核源码存储库** 18。  
为了排除假阳性，本评估对检索过程中极易混淆的几个已知存储库和项目进行了技术鉴别与排除：

* HandsomeMod/linux-allwinner-4.9：此项目为 2021 年的早期快照。虽然其包含了全志 4.9 内核的大量基础结构，但由于发布时间远早于 RG35XX-SP 的硬件设计，其缺失了 H700 掌机所需的全部 AXP2202 定制化电源管理与电池子驱动，无法实现基本电源控制。  
* batocera-linux/batocera.linux：该项目属于现代主线化重构项目，使用的是 Mainline 6.x 内核（如 6.18 或 6.15 阶段），并非官方 4.9.170 BSP 遗留内核 24。  
* mporrato/alpine-h700：该脚本套件用于为 H700 芯片组制作 Alpine Linux 镜像，但其内核与模块完全直接剥离自官方 Stock OS 固件的预编译二进制包（factory.img），并不包含任何内核源码或编译流程 16。  
* Knulli h700-mainline 分支：属于针对全志 H700 平台的主线化移植 WIP（进行中）占位符，其目标是完全脱离旧版 4.9.170 代码体系 19。  
* linux-sunxi/linux-sunxi 与 torvalds/linux：前者为全志社区的主线镜像，后者为标准 Linux 主线内核。两者均仅包含经过合并的、符合内核规范的通用硬件驱动，不含全志早期 BSP 内部私有的、非标准的 disp2 显示核心及闭源驱动模块 19。

## **核心缺失驱动子系统的技术剖析**

由于官方源码不可得，第三方开发者在面对安伯尼克 H700 平台时，必须深入理解并解决以下两个核心专有硬件子系统的驱动缺失问题。

### **AXP2202 PMIC 电池与 USB 供电子系统**

安伯尼克 RG35XX-SP 的电源控制、锂电池充电逻辑、电量计量以及合盖休眠（通过霍尔效应磁吸开关实现）高度依赖于定制的 X-Powers AXP2202 电源管理单元 26。在 UART 日志中，可以观察到以下底层驱动调用：  
axp2202\_usb\_power: axp2202-acin device is not configed, not use vbus-det 5 axp2202\_battery: bat\_vol:3975 26 PM: pm\_system\_irq\_wakeup: 422 triggered axp2202\_irq\_chip 27  
在标准 Linux 内核中，AXP2202 的通用驱动仅提供了基础的电压调节和中断映射，并不包含针对特定电池物理曲线的电量估算机制 27。原厂 BSP 内核通过 axp2202-bat-power-supply 与 axp2202-usb-power-supply 两个专用驱动子系统，实现了对充电状态、放电截止电压、温度保护和电量计（Fuel Gauge）的高精度闭环控制 11。  
同时，合盖传感器的状态转换与休眠唤醒逻辑，由原厂开发人员直接硬编码写入了 PMIC 的工作队列中：  
\---\[allen\] hall\_press\_work\_func: state=1 26 \---\[allen\] hall\_switch\_isr: 711 26  
日志中出现的 \[allen\] 调试标记揭示了该部分代码是由安伯尼克的内部软件工程师（或其方案商技术代表 Allen）直接在全志 4.9.170 SDK 源码树中进行本地修改与打补丁生成的 11。这部分私有补丁从未在公开的任何全志开源分支中提交，导致通用内核在编译后无法支持 SP 掌机的合盖休眠和电量百分比读取。

### **Display Engine 2 (disp2) 驱动与 LCD 初始化**

全志处理器的视频输出由其专属的显示引擎（DE2.0）驱动模块 disp2（位于 drivers/video/fbdev/sunxi/disp2/）管理 28。在系统初始化期间，内核必须向 disp2 驱动注册特定 LCD 面板的初始化时序与寄存器配置 11：  
\===lcd\_panel\_uboot\_fj035fhd05\_v1\_init:L274,lcd\_type \= 0 11 allen\_boe\_lcd=1, str=boe 7  
这些信息表明，安伯尼克在其源码树中静态编译了至少两套屏幕面板的初始化参数（例如用于京东方 BOE 面板与老款面板的初始化序列） 7。由于这部分代码缺失，开发者在尝试自行编译 4.9.170 内核时，极易因屏幕初始化时序不匹配或无法正确加载 disp2 模块而面临开机绿灯亮起但屏幕黑屏的现象 26。

## **次优替代开源仓库的技术对齐**

鉴于原厂源码不可得，若开发者仍需要在 4.9.170 这一特定内核版本下进行开发，目前全网技术对齐度最高的开源代码参考点为香橙派（Orange Pi）官方发布的 H616 开发板内核。因为 H700 与 H616 / H618 处理器同属全志 sun50iw9 家族，其寄存器映射与底层 IP 核具有极高的同源性 12。  
该替代存储库的具体技术参数如下：

* **克隆 URL**：https://github.com/orangepi-xunlong/linux-orangepi 32  
* **目标分支**：orange-pi-4.9-sun50iw9 21  
* **最近提交日期**：2022 年底至 2023 年中（处于维护冻结状态） 8  
* **存储库体积估计**：约 1.5 GB  
* **AXP2202 子驱动支持情况**：该分支仅包含基础的 AXP2202 核心 PMU 支持，**完全缺失** axp2202-bat-power-supply 和 axp2202-usb-power-supply 两个针对掌机的电量计量与限流充电子系统驱动 26。

下表详细对比了 RG35XX-SP 原厂内核需求与香橙派替代源码的技术对齐度：

| 关键技术模块 | RG35XX-SP 目标 BSP 内核需求 | 香橙派替代源码 (orange-pi-4.9-sun50iw9) | 对齐度与可移植性评估 |
| :---- | :---- | :---- | :---- |
| **内核主版本** | 4.9.170 1 | 4.9.170 10 | **完全对齐**。基础内核结构及系统调用 ABI 完全一致。 |
| **显示子系统** | drivers/video/fbdev/sunxi/disp2/ 28 | 包含全志原生 disp2 驱动目录 28 | **高度一致**。提供了底层 DE2 控制框架，但需要手动添加安伯尼克特定的 LCD 初始化时序。 |
| **图形加速 (GPU)** | modules/gpu/mali-midgard/ 24 | 包含 Mali Midgard GPU 驱动源码 28 | **完全一致**。支持 Mali-G31 MP2 的内核空间驱动加载 11。 |
| **芯片组定义 (SoC)** | sun50iw9p1.dtsi 设备树 28 | 包含 H616 平台的 sun50iw9p1.dtsi 定义 33 | **完全一致**。SoC 级别的时钟、复位、总线定义完全通用。 |
| **电源控制 (PMIC)** | AXP2202 掌机专用供电与电池管理 26 | 仅包含 AXP2202 基础 PMU 通用寄存器定义 11 | **部分缺失**。必须手动补齐电池充电曲线逻辑与霍尔传感器（Lid Switch）中断响应代码 26。 |
| **板级设备树** | sun50i-h700-rg35xx-sp.dts 1 | 仅包含 sun50i-h616-orangepizero2.dts 等板级定义 31 | **完全缺失**。需要开发者利用反编译工具对官方固件中的 DTB 进行还原并移植 1。 |

## **开发者技术规避与替代方案**

在官方内核源码实质性缺席的背景下，复古掌机开源社区经过多轮探索，总结并确立了三条成熟的系统开发规避路线。

### **二进制内核注入与用户态重构**

这一技术方案被 MuOS、Stock OS 优化版（如 cbepx-me 发布的 StockOS MOD）等主流定制固件广泛采用 36。  
其基本原理是放弃重新编译内核的尝试，完整保留安伯尼克官方系统第一分区（BOOT 分区）中的预编译内核镜像 zImage（或 Image）、引导程序包（boot-package）以及存在于 /lib/modules/4.9.170 目录下的所有预编译驱动模块（.ko 文件） 17。  
在此基础上，开发者利用 qemu-user-static 架构模拟技术，在 PC 端挂载并构建全新的、高度定制化的 Linux 用户态根文件系统（Rootfs，如 Ubuntu 18.04/22.04 LTS 或极简 Alpine Linux） 16：

Bash

\# 挂载自定义根文件系统并利用 QEMU 模拟环境进入  
sudo mkdir /mnt/rg35xx  
sudo mount /dev/sdd5 /mnt/rg35xx/  
cd /mnt/rg35xx/  
sudo mount \-t proc /proc proc/  
sudo mount \-t sysfs /sys sys/  
sudo mount \-o bind /dev dev/  
sudo mount \-o bind /run run/  
sudo chroot. qemu-aarch64-static /bin/bash

(参考来源: 38)  
这种方法既确保了 100% 的硬件兼容性（PMIC、充电、合盖休眠、HDMI 音视频输出、Wi-Fi/蓝牙驱动均能完美运行），又赋予了开发者自由定制 EmulationStation 菜单、优化后台脚本以及升级用户态动态库的完整控制权 24。

### **树外内核模块交叉编译**

针对需要在掌机上扩展特定 USB 外设驱动（例如用于 M8 音乐合成器客户端的 cdc-acm 虚拟串口驱动，或特定 USB 声卡驱动）的开发者，可在纯净的 Linux 4.9.170 环境下进行单独驱动模块编译，而无需重构整个内核 13。  
具体操作步骤通常通过交叉编译脚本实现 15：

Bash

\# 1\. 准备纯净版 4.9.170 内核树  
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.9.170.tar.gz  
tar xzf linux-4.9.170.tar.gz  
cd linux-4.9.170

\# 2\. 拉取社区维护的、与原厂内核 ABI 严格对齐的配置文件  
wget \-O.config https://raw.githubusercontent.com/knulli-cfw/distribution/refs/heads/knulli-main/board/batocera/allwinner/h700/linux-sunxi64-legacy.config

\# 3\. 配置并生成底层依赖环境  
make ARCH=arm64 CROSS\_COMPILE=aarch64-linux-gnu- olddefconfig  
make ARCH=arm64 CROSS\_COMPILE=aarch64-linux-gnu- modules\_prepare

\# 4\. 仅编译所需的特定目标模块（以 cdc-acm 驱动为例）  
make ARCH=arm64 CROSS\_COMPILE=aarch64-linux-gnu- M=drivers/usb/class

(参考来源: 13)  
编译完成后，将生成的 cdc-acm.ko 文件拷贝至掌机存储卡中，在系统启动时通过 insmod 或 modprobe 动态强制加载入运行中的官方内核中 14。此方法避免了与原厂闭源驱动（如 disp2）的编译冲突，解决了特定外部设备的连接问题。

### **全面主线化移植工程**

作为最终的终极解决方案，复古硬件开源社区正在合力推进全志 H700 平台的主线化（Mainlining）移植，目标是彻底淘汰陈旧的全志 4.9.170 BSP 体系 21。目前，这一进程已在 ROCKNIX 与 Knulli 主线分支中取得实质性进展 19：

* **图形渲染**：采用 Linux 上游主线的开源 **Panfrost** GPU 驱动，完全取代了全志闭源的 Mali 核心驱动，在主线内核（如 6.12+）下实现了完整的 OpenGL / GLES 3.1 硬件图形加速 25。  
* **显示架构**：摒弃私有的 disp2 引擎，使用现代 Linux 的 DRM/KMS 显示驱动框架，利用 Sway 合成器和 EmulationStation 提供了流畅的 UI 交互 25。  
* **设备树重构**：通过对官方二进制 DTB 文件的深度逆向反编译，重新编写并向 Linux 内核主线提交了标准的设备树定义（如 sun50i-h700-anbernic-rg35xx-plus.dts 及 \-sp.dts 系列文件） 19。

主线化路线虽然目前在某些边缘细节（如极致低功耗深度睡眠模式、特定 HDMI 外部音频切换）上仍处于攻关阶段，但它从根本上消除了对安伯尼克原厂闭源内核的依赖，是让 RG35XX-SP 掌机获得长期生命力与现代软件技术栈支持的必经之路 24。

#### **引用的著作**

1. Anbernic RG35XXSP Data · GitHub, 访问时间为 五月 17, 2026， [https://gist.github.com/macromorgan/ea39d090ef75281194bb2f0a307bb0c7](https://gist.github.com/macromorgan/ea39d090ef75281194bb2f0a307bb0c7)  
2. LennartHennigs/Knulli-and-RGXX-Notes: How to install and set up Knulli on the Anbernic RG35XX or RG40XX. \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/LennartHennigs/Knulli-and-RGXX-Notes](https://github.com/LennartHennigs/Knulli-and-RGXX-Notes)  
3. RG35XX+ with Batocera on HDMI/BT : r/RG35XX\_Plus \- Reddit, 访问时间为 五月 17, 2026， [https://www.reddit.com/r/RG35XX\_Plus/comments/1apka7l/rg35xx\_with\_batocera\_on\_hdmibt/](https://www.reddit.com/r/RG35XX_Plus/comments/1apka7l/rg35xx_with_batocera_on_hdmibt/)  
4. Atari 2600+ Hardware \- Page 13, 访问时间为 五月 17, 2026， [https://forums.atariage.com/topic/357064-atari-2600-hardware/page/13/](https://forums.atariage.com/topic/357064-atari-2600-hardware/page/13/)  
5. Anbernic RG28XX \- 焊接UART \- 司徒的教學網站, 访问时间为 五月 17, 2026， [https://steward-fu.github.io/website/handheld/rg28xx\_uart.htm](https://steward-fu.github.io/website/handheld/rg28xx_uart.htm)  
6. 司徒的教學網站, 访问时间为 五月 17, 2026， [https://steward-fu.github.io/website/handheld/rg35xx-plus\_uart.htm](https://steward-fu.github.io/website/handheld/rg35xx-plus_uart.htm)  
7. Anbernic RG35XX Plusに UART をつける \- Zenn, 访问时间为 五月 17, 2026， [https://zenn.dev/tomoki\_imai/articles/b78a0441512b89](https://zenn.dev/tomoki_imai/articles/b78a0441512b89)  
8. TanixTX6SStockAndroidBootlog.txt \- GitHub Gist, 访问时间为 五月 17, 2026， [https://gist.github.com/LarsLinux93/de09c7bcbf2472b5fd9354de08083374](https://gist.github.com/LarsLinux93/de09c7bcbf2472b5fd9354de08083374)  
9. Tanix TX6s serial boot \- GitHub, 访问时间为 五月 17, 2026， [https://gist.github.com/heitbaum/c6b266a8e32dd14c8824ef7fcf067207](https://gist.github.com/heitbaum/c6b266a8e32dd14c8824ef7fcf067207)  
10. BTT SKR Pico isn't detected until manually reset · Issue \#11 · bigtreetech/CB1 \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/bigtreetech/CB1/issues/11](https://github.com/bigtreetech/CB1/issues/11)  
11. RG34XX \- GitHub Gist, 访问时间为 五月 17, 2026， [https://gist.github.com/macromorgan/3e12146a63b53c5b61e4e3c7d4547a0e](https://gist.github.com/macromorgan/3e12146a63b53c5b61e4e3c7d4547a0e)  
12. RG40XX-H \- GitHub Gist, 访问时间为 五月 17, 2026， [https://gist.github.com/macromorgan/c96590eb5bdad96771f26ef20bf20512](https://gist.github.com/macromorgan/c96590eb5bdad96771f26ef20bf20512)  
13. Building m8c for RG35XX Plus/H/SP on KNULLI (Scroll down for zip) \- GitHub Gist, 访问时间为 五月 17, 2026， [https://gist.github.com/Stanley00/c3541c91270cea966630ff7a4a6aaf53](https://gist.github.com/Stanley00/c3541c91270cea966630ff7a4a6aaf53)  
14. jamesMcMeex/m8c-rg35xx-knulli: A Docker platform to (hopefully) make it easier to compile a runnable executable of the M8 Tracker software for RG35XX\* devices running Knulli CFW \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/jamesMcMeex/m8c-rg35xx-knulli](https://github.com/jamesMcMeex/m8c-rg35xx-knulli)  
15. Building m8c for RG35XX Plus/H/SP on KNULLI (Scroll down for zip) \- Github-Gist, 访问时间为 五月 17, 2026， [https://gist.github.com/mnml/12f75bbf16eac4def15ba72cf1b11926](https://gist.github.com/mnml/12f75bbf16eac4def15ba72cf1b11926)  
16. mporrato/alpine-h700: Alpine Linux for the Allwinner H700 SoC \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/mporrato/alpine-h700](https://github.com/mporrato/alpine-h700)  
17. Switching out the kernel zImage for custom compiled Image · Issue \#2 · mporrato/alpine-h700 \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/mporrato/alpine-h700/issues/2](https://github.com/mporrato/alpine-h700/issues/2)  
18. Confused about the rg35xx sp and knulli \- Reddit, 访问时间为 五月 17, 2026， [https://www.reddit.com/r/RG35XX/comments/1goj932/confused\_about\_the\_rg35xx\_sp\_and\_knulli/](https://www.reddit.com/r/RG35XX/comments/1goj932/confused_about_the_rg35xx_sp_and_knulli/)  
19. Analogue snapping fix patch inclusion in cbepx-me · Issue \#75 \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/cbepx-me/Anbernic-H700-RG-xx-StockOS-Modification/issues/75](https://github.com/cbepx-me/Anbernic-H700-RG-xx-StockOS-Modification/issues/75)  
20. Quard/rg35xx-app-DinguxCommander \- Gitee, 访问时间为 五月 17, 2026， [https://gitee.com/QQxiaoming/rg35xx-app-DinguxCommander/blob/main/make\_native.sh](https://gitee.com/QQxiaoming/rg35xx-app-DinguxCommander/blob/main/make_native.sh)  
21. Is there a tutorial to build kernel for rg35xxh? · Issue \#205 \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/rg35xx-cfw/rg35xx-cfw.github.io/issues/205](https://github.com/rg35xx-cfw/rg35xx-cfw.github.io/issues/205)  
22. No aw859a-wifi.service · Issue \#47 · orangepi-xunlong/orangepi-build \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/orangepi-xunlong/orangepi-build/issues/47](https://github.com/orangepi-xunlong/orangepi-build/issues/47)  
23. GPL compliance? : r/SBCGaming \- Reddit, 访问时间为 五月 17, 2026， [https://www.reddit.com/r/SBCGaming/comments/1o50r8q/gpl\_compliance/](https://www.reddit.com/r/SBCGaming/comments/1o50r8q/gpl_compliance/)  
24. Anbernic RG35XX SP (anbernic-rg35xx-sp) \- postmarketOS Wiki, 访问时间为 五月 17, 2026， [https://wiki.postmarketos.org/wiki/Anbernic\_RG35XX\_SP\_(anbernic-rg35xx-sp)](https://wiki.postmarketos.org/wiki/Anbernic_RG35XX_SP_\(anbernic-rg35xx-sp\))  
25. Anbernic RG35XX SP \- RockNIX, 访问时间为 五月 17, 2026， [https://rocknix.org/devices/anbernic/rg35xx-sp/](https://rocknix.org/devices/anbernic/rg35xx-sp/)  
26. Black Screen / Green LED ON | RG35XX SP \- 2502 Pixie \- MustardOS Community Forum, 访问时间为 五月 17, 2026， [https://community.muos.dev/t/black-screen-green-led-on-rg35xx-sp/342](https://community.muos.dev/t/black-screen-green-led-on-rg35xx-sp/342)  
27. Can anyone give me a solution for MTP transfer drop : r/androidroot \- Reddit, 访问时间为 五月 17, 2026， [https://www.reddit.com/r/androidroot/comments/1oavoav/can\_anyone\_give\_me\_a\_solution\_for\_mtp\_transfer/](https://www.reddit.com/r/androidroot/comments/1oavoav/can_anyone_give_me_a_solution_for_mtp_transfer/)  
28. orangepi-build/external/packages/pack-uboot/sun50iw9/bin/dts/orangepizero2w-u-boot-current.dts at next \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/orangepi-xunlong/orangepi-build/blob/next/external/packages/pack-uboot/sun50iw9/bin/dts/orangepizero2w-u-boot-current.dts](https://github.com/orangepi-xunlong/orangepi-build/blob/next/external/packages/pack-uboot/sun50iw9/bin/dts/orangepizero2w-u-boot-current.dts)  
29. Documentation/devicetree/bindings/display/panel/anbernic,rg35xx-plus-panel.yaml · e7ed343658792771cf1b868df061661b7bcc5cef \- beim GitLab der FH Aachen\!, 访问时间为 五月 17, 2026， [https://git.fh-aachen.de/tb3838s/linux/-/blob/e7ed343658792771cf1b868df061661b7bcc5cef/Documentation/devicetree/bindings/display/panel/anbernic,rg35xx-plus-panel.yaml](https://git.fh-aachen.de/tb3838s/linux/-/blob/e7ed343658792771cf1b868df061661b7bcc5cef/Documentation/devicetree/bindings/display/panel/anbernic,rg35xx-plus-panel.yaml)  
30. Anbernic RG35XX-Plus minimal DT \- GitHub Gist, 访问时间为 五月 17, 2026， [https://gist.github.com/apritzel/f35dd4ab78c3df08b344b115751cd2b8](https://gist.github.com/apritzel/f35dd4ab78c3df08b344b115751cd2b8)  
31. Add support of Allwinner H616 (sun50iw9p1) · Issue \#1 · ZhangGaoxing/sunxi-gpio-driver, 访问时间为 五月 17, 2026， [https://github.com/ZhangGaoxing/sunxi-gpio-driver/issues/1](https://github.com/ZhangGaoxing/sunxi-gpio-driver/issues/1)  
32. uwe5622-aw \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/ChalesYu/uwe5622-aw](https://github.com/ChalesYu/uwe5622-aw)  
33. Tanix-TX6s-fdt-devicetree-from-android \- GitHub Gist, 访问时间为 五月 17, 2026， [https://gist.github.com/heitbaum/30676739ea278a4384ceb2f3486ce232](https://gist.github.com/heitbaum/30676739ea278a4384ceb2f3486ce232)  
34. \[PATCH v3 19/21\] arm64: dts: allwinner: Add Allwinner H616 .dtsi file \- Mailing Lists, 访问时间为 五月 17, 2026， [https://lists.infradead.org/pipermail/linux-arm-kernel/2021-January/632104.html](https://lists.infradead.org/pipermail/linux-arm-kernel/2021-January/632104.html)  
35. iot/src/devices/Gpio/Drivers/OrangePiZero2Driver.cs at main · dotnet/iot \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/dotnet/iot/blob/main/src/devices/Gpio/Drivers/OrangePiZero2Driver.cs](https://github.com/dotnet/iot/blob/main/src/devices/Gpio/Drivers/OrangePiZero2Driver.cs)  
36. Releases · cbepx-me/Anbernic-H700-RG-xx-StockOS-Modification \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/cbepx-me/Anbernic-H700-RG-xx-StockOS-Modification/releases](https://github.com/cbepx-me/Anbernic-H700-RG-xx-StockOS-Modification/releases)  
37. Regularly updated list of firmware compare for Anbernic H700 devices \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/symbuzzer/anbernic-h700-fw-comparing](https://github.com/symbuzzer/anbernic-h700-fw-comparing)  
38. RG35XX H Adventures \- GitHub Gist, 访问时间为 五月 17, 2026， [https://gist.github.com/Daft-Freak/96e254b768e91533783e3f115f1da6ea](https://gist.github.com/Daft-Freak/96e254b768e91533783e3f115f1da6ea)  
39. anbernic-h700-fw-comparing/README\_new.md at main \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/symbuzzer/anbernic-h700-fw-comparing/blob/main/README\_new.md](https://github.com/symbuzzer/anbernic-h700-fw-comparing/blob/main/README_new.md)  
40. sun50i-h700-anbernic-rg35xx-plus.dts \- torvalds/linux \- GitHub, 访问时间为 五月 17, 2026， [https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/allwinner/sun50i-h700-anbernic-rg35xx-plus.dts](https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/allwinner/sun50i-h700-anbernic-rg35xx-plus.dts)