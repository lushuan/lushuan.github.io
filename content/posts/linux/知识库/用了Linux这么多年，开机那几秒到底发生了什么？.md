---
title: "用了Linux这么多年，开机那几秒到底发生了什么？——从CentOS到Ubuntu再到Kylin V10，一次讲透Linux启动流程"
subtitle: ""
date: 2024-03-28T12:06:37+08:00
draft: true
toc:
  enable: true
weight: false
categories: ["Linux"]
tags: ["Linux"]
---

## 写在前面

用Linux很多年了，从CentOS 6、7一路用到Ubuntu 22.04，最近又转战信创，开始折腾银河麒麟Kylin V10。命令敲得越来越溜，`systemctl`、`journalctl`、`grub2-mkconfig`这些张口就来，但有一天突然问了一句："Linux从按下电源键到你看见登录界面，到底经历了什么？"

我愣了一下，发现自己只能说出个大概。当初鸟哥的《私房菜》读过，入门够了，但入门这么久了，该往深处走走了。

于是有了这篇文章。既是写给自己看的笔记，也希望能帮到和我一样"知其然，想知其所以然"的朋友。

------

## 一、先画一张全景图

把Linux启动想象成一场**接力赛**，每一棒把控制权交给下一棒：

```
按下电源键
    │
    ▼
┌─────────────────────────────┐
│  ① 固件阶段（BIOS / UEFI）  │  ← 主板上的"老祖宗"
└─────────────┬───────────────┘
              ▼
┌─────────────────────────────┐
│  ② 引导加载器（GRUB2）      │  ← "引路人"
└─────────────┬───────────────┘
              ▼
┌─────────────────────────────┐
│  ③ 内核启动（vmlinuz+initramfs）│ ← "真正的主角登场"
└─────────────┬───────────────┘
              ▼
┌─────────────────────────────┐
│  ④ 用户空间初始化（systemd） │  ← "大管家"
└─────────────┬───────────────┘
              ▼
┌─────────────────────────────┐
│  ⑤ 服务启动 → 登录界面      │  ← "你看到的世界"
└─────────────────────────────┘
```

这五个阶段，**不管你是CentOS、Ubuntu还是麒麟V10，大框架完全一致**。差异藏在细节里，后面会逐一对比。

------

## 二、阶段①：固件——BIOS与UEFI

### 通俗理解

你按下电源键，CPU通电后第一件事不是找操作系统，而是执行主板上那块**固件芯片**里的代码。它就像一个值班保安：先巡逻一圈确认大楼没塌（硬件自检），然后看看该给谁开门（选择启动设备）。

### BIOS（传统）

- 执行 **POST**（Power-On Self-Test）：检测CPU、内存、显卡、硬盘等。
- 按CMOS中设定的启动顺序，找到第一个可启动设备。
- 读取该设备的**第0个扇区（MBR，Master Boot Record，512字节）**。
    - 前446字节：引导程序代码（Boot Loader）
    - 接下来64字节：分区表
    - 最后2字节：魔术数字 `0x55AA`（告诉BIOS"这是个有效的引导扇区"）
- 把控制权交给MBR里的引导程序（也就是GRUB的第一阶段）。

> **限制**：MBR只有512字节，分区表最多4个主分区，磁盘寻址最大2TB。

### UEFI（现代）

- 同样执行POST，但用**UEFI固件**代替传统BIOS。
- 使用**GPT分区表**（支持128个分区，磁盘可达9.4ZB）。
- 不再读MBR，而是在磁盘上找一个特殊的**ESP分区**（EFI System Partition，FAT32格式，通常挂载在 `/boot/efi`），里面放着一堆 `.efi` 可执行文件。
- 固件直接加载并执行 `/boot/efi/EFI/ubuntu/grubx64.efi`（或对应发行版的efi文件）。
- 支持 **Secure Boot**：固件会校验efi文件的数字签名，没签名的不让你跑。

> **你的三个系统**：CentOS 7时代很多机器还是BIOS+MBR；Ubuntu 22.04和麒麟V10默认走UEFI+GPT，但都兼容BIOS模式。

------

## 三、阶段②：引导加载器——GRUB2

### 通俗理解

固件把接力棒交给了GRUB（GRand Unified Bootloader）。GRUB的任务就一件事：**让你选择启动哪个内核，然后把内核文件加载到内存里**。它是"引路人"。

### GRUB做了什么

1. **显示菜单**（如果配置了多系统或多内核，会弹出选择界面）。
2. 加载你选定的**内核文件** `vmlinuz`（一个被压缩的Linux内核）。
3. 加载**初始内存文件系统** `initramfs`（或叫 `initrd`），这是一个临时的迷你文件系统，包含启动早期必需的驱动和工具。
4. 把控制权交给内核，同时传递**内核启动参数**（`/boot/grub2/grub.cfg` 或 `/boot/grub/grub.cfg` 里写的那些 `linux /vmlinuz root=/dev/sda2 ro quiet splash` 之类）。

### 不同发行版的GRUB差异

| 项目             | CentOS 7                                 | Ubuntu 22.04                       | 麒麟 V10                               |
| ---------------- | ---------------------------------------- | ---------------------------------- | -------------------------------------- |
| GRUB配置文件路径 | `/boot/grub2/grub.cfg`                   | `/boot/grub/grub.cfg`              | `/boot/grub2/grubenv(是一个软链接)`    |
| 默认配置文件     | `/etc/default/grub`                      | `/etc/default/grub`                | `/etc/default/grub`                    |
| 重新生成命令     | `grub2-mkconfig -o /boot/grub2/grub.cfg` | `update-grub`                      | `update-grub`                          |
| UEFI下efi文件    | `/boot/efi/EFI/centos/grubx64.efi`       | `/boot/efi/EFI/ubuntu/grubx64.efi` | `/boot/efi/EFI/kylin/grubx64.efi`      |
| GRUB密码保护     | 默认无                                   | 默认无                             | **服务器版默认有**（root/Kylin123123） |

> 麒麟V10服务器版进入GRUB编辑需要输入账号密码，这是信创安全加固的一个体现。

------

## 四、阶段③：内核启动——最核心的部分

### 通俗理解

GRUB把内核（vmlinuz）和"应急工具箱"（initramfs）塞进内存，然后说："兄弟，接下来交给你了。"内核从这一刻起接管整台机器。

### 4.1 vmlinuz 是什么

- `vm` = Virtual Memory（历史原因，表示支持虚拟内存的内核）
- `linu` = Linux
- `z` = zlib压缩

它就是一个**被压缩的内核二进制文件**。内核启动的第一步就是**自我解压**。

### 4.2 initramfs（初始内存文件系统）

这是一个**临时根文件系统**，被加载到内存中。为什么需要它？因为真正的根文件系统（`/`）可能在LVM、RAID、加密分区上，或者需要特定驱动才能访问。initramfs里提前放好了这些驱动和工具。

你可以这样查看它的内容：

```bash
# 查看当前initramfs里有什么
lsinitramfs /boot/initrd.img-$(uname -r) | head -30    # Ubuntu
lsinitrd /boot/initramfs-$(uname -r).img | head -30    # CentOS 7/麒麟
```

里面通常有：

- 关键存储驱动（如 `nvme`、`ahci`、`dm-mod`、`lvm2`）
- 文件系统驱动（如 `ext4`、`xfs`）
- `systemd` 或 `dracut` 的早期初始化脚本
- 加密相关工具（如果用了LUKS）

### 4.3 内核启动的具体步骤

```
vmlinuz 解压
    │
    ▼
start_kernel()          ← 内核的 main() 函数
    ├── 初始化CPU、内存管理、中断
    ├── 加载早期驱动（控制台、时钟）
    ├── 挂载 initramfs 为临时根文件系统
    ├── 启动第一个用户空间进程（init）
    │
    ▼
initramfs 里的 init 脚本执行
    ├── 探测硬件、加载必要模块
    ├── 组装真正的根文件系统（LVM activate、RAID assemble等）
    ├── 挂载真正的 / 分区
    ├── switch_root：切换到真正的根文件系统，释放initramfs
    │
    ▼
真正的 / 下的 init（即 systemd）接管
```

### 4.4 不同内核版本对应的启动流程差异大吗？

**大框架不变，细节在进化。** 从2.6到5.x再到6.x：

| 变化点                           | 说明                                     |
| -------------------------------- | ---------------------------------------- |
| initramfs 取代 initrd            | 2.6以后统一用cpio格式的initramfs，更灵活 |
| systemd 成为默认init             | 从2010年代开始，取代了SysVinit和Upstart  |
| 内核自解压算法升级               | 从gzip到xz、zstd，解压更快               |
| 早期用户空间（early init）更复杂 | 支持Secure Boot验证、TPM度量等           |
| 启动速度优化                     | 并行初始化、异步探测等                   |

但 `start_kernel()` → 挂载initramfs → 启动PID 1 这条主线，**从2.6到6.x没有本质变化**。

------

## 五、阶段④：用户空间初始化——systemd的天下

### 通俗理解

内核把舞台搭好了，现在轮到"大管家"systemd上场。它是**PID 1**，是所有用户空间进程的祖先。它的任务是：按照预定义的"目标"（target），把该启动的服务全部拉起来。

### 5.1 为什么是systemd？

| 时代 | init系统                       | 代表发行版                        |
| ---- | ------------------------------ | --------------------------------- |
| 远古 | SysVinit（`/etc/rc.d/rc*.d/`） | CentOS 6及更早                    |
| 过渡 | Upstart                        | Ubuntu 14.04及更早                |
| 现代 | **systemd**                    | CentOS 7+、Ubuntu 16.04+、麒麟V10 |

> CentOS 7已经是systemd了。如果你用过CentOS 6，会记得 `chkconfig` 和 `service` 命令，那就是SysVinit时代。

### 5.2 systemd的启动逻辑：Target（目标）

systemd用**Target**代替了传统的runlevel：

```
default.target（通常是 graphical.target 或 multi-user.target）
    │
    ├── graphical.target（图形界面）
    │     └── multi-user.target（多用户命令行）
    │           └── basic.target（基础服务）
    │                 └── sysinit.target（系统初始化）
    │                       └── 各种 .service、.mount、.socket 单元
    │
    └── 最终你看到的：登录管理器（GDM/LightDM/UKUI登录）
```

查看当前默认target：

```bash
systemctl get-default
# graphical.target  → 进图形界面
# multi-user.target → 进命令行（服务器常见）
```

### 5.3 systemd启动服务的特点

- **并行启动**：不像SysVinit一个个排队，systemd根据依赖关系并行拉起服务，所以快。
- **依赖声明式**：每个 `.service` 文件里写清楚 `After=`、`Requires=`、`Wants=`。
- **按需启动**：socket activation、path activation等机制，服务可以等第一个连接来了再启动。

### 5.4 三个系统的systemd差异

| 项目            | CentOS 7                                              | Ubuntu 22.04                              | 麒麟 V10             |
| --------------- | ----------------------------------------------------- | ----------------------------------------- | -------------------- |
| systemd版本     | 219                                                   | 249                                       | 249（与Ubuntu同源）  |
| 默认target      | graphical.target（桌面）/ multi-user.target（服务器） | graphical.target                          | graphical.target     |
| 日志工具        | `journalctl`（systemd-journald）                      | 同左                                      | 同左                 |
| 服务管理        | `systemctl`                                           | 同左                                      | 同左                 |
| 网络管理        | NetworkManager                                        | Netplan + NetworkManager/systemd-networkd | NetworkManager       |
| 桌面/登录管理器 | GDM（GNOME）                                          | GDM（GNOME）                              | **UKUI**（自研桌面） |

> 麒麟V10的systemd与Ubuntu基本同源（都基于Debian系），但桌面环境换成了自研的UKUI，安全策略更严格（如GRUB密码、安全模块等）。

------

## 六、阶段⑤：服务启动到你看见登录界面

systemd按照依赖链把所有服务拉起来后：

1. **显示管理器**（Display Manager）启动：
    - CentOS 7 / Ubuntu 22.04：`gdm`
    - 麒麟V10：`lightdm` 或 UKUI自带的登录管理器
2. 你看到**图形登录界面**。
3. 输入用户名密码后，启动**桌面会话**（GNOME / UKUI）。
4. 用户空间的shell、文件管理器、终端模拟器等你熟悉的一切，都是在这个会话里跑的。

如果是服务器（`multi-user.target`），你看到的直接就是 **tty1 的文字登录提示符**。

------

## 七、那些你可能没注意到的细节

### 7.1 Secure Boot（安全启动）

UEFI固件只允许加载**经过数字签名**的引导程序和内核。Ubuntu和麒麟V10的官方ISO都带有Canonical/麒麟的签名，所以能过Secure Boot。如果你自己编译了内核，要么签名，要么关Secure Boot。

### 7.2 内核启动参数（你排查问题时的救命稻草）

在GRUB菜单按 `e` 编辑，常见参数：

| 参数                             | 作用                                   |
| -------------------------------- | -------------------------------------- |
| `root=/dev/sda2`                 | 指定根分区                             |
| `ro` / `rw`                      | 初始以只读/读写方式挂载根分区          |
| `quiet`                          | 不显示内核启动信息                     |
| `splash`                         | 显示启动画面                           |
| `single` / `rescue`              | 进入单用户/救援模式                    |
| `init=/bin/bash`                 | 直接进bash（忘了root密码时的救命操作） |
| `systemd.unit=multi-user.target` | 跳过图形界面                           |

> 麒麟V10服务器版做这个操作前要先输入GRUB密码。

### 7.3 initramfs 的生成工具

| 发行版       | 工具                        | 重建命令              |
| ------------ | --------------------------- | --------------------- |
| CentOS 7     | dracut                      | `dracut -f`           |
| Ubuntu 22.04 | initramfs-tools             | `update-initramfs -u` |
| 麒麟 V10     | initramfs-tools（同Ubuntu） | `update-initramfs -u` |

### 7.4 启动性能分析

```bash
# 总启动耗时
systemd-analyze

# 各阶段耗时分解
systemd-analyze blame

# 启动链依赖图（SVG）
systemd-analyze critical-chain

# 绘制启动流程图（SVG文件）
systemd-analyze plot > boot.svg
```

这三个系统都支持，排查"开机慢"特别好用。

### 7.5 查看启动日志

```bash
journalctl -b          # 本次启动的所有日志
journalctl -b -p err   # 只看错误级别以上
dmesg                  # 内核环形缓冲区消息（硬件、驱动相关）
```

### 7.6 /boot 目录里到底有什么

```bash
ls /boot/
# 你会看到：
# vmlinuz-5.15.0-xx-generic    ← 压缩内核
# initrd.img-5.15.0-xx-generic ← initramfs
# config-5.15.0-xx-generic     ← 内核编译配置
# System.map-5.15.0-xx-generic ← 内核符号表
# grub/ 或 grub2/              ← GRUB配置和模块
# efi/                         ← UEFI引导文件（UEFI模式下）
```

------

## 八、不同发行版、不同内核版本，启动流程是否有共同点？

### 总结一句话：**骨架相同，皮肉有别。**

**相同的部分（骨架）：**

- 固件 → 引导加载器 → 内核+initramfs → systemd → 服务 → 登录
- 这条主线从Linux诞生至今没变过，不管是CentOS、Ubuntu、麒麟，还是Fedora、Arch、openSUSE。

**不同的部分（皮肉）：**

- 固件层：BIOS还是UEFI，取决于主板和安装方式
- 引导层：GRUB配置路径、是否加密码保护、是否启用Secure Boot
- 内核层：内核版本号不同（CentOS 7默认3.10，Ubuntu 22.04默认5.15，麒麟V10默认4.19/5.4/5.10），但启动逻辑一致
- init层：systemd版本号不同，但机制相同
- 服务层：网络管理方案不同、桌面环境不同、安全策略不同

**内核版本的影响：**

- 3.x → 4.x → 5.x → 6.x：启动流程的**架构没变**
- 变化的是：支持的新硬件驱动更多、initramfs生成工具更新、安全特性增强（如Lockdown模式）、启动速度优化
- 你不需要因为升级内核而重新学习启动流程

------

## 九、一张图总结：三个系统的启动流程对比

```
                CentOS 7              Ubuntu 22.04           麒麟 V10
                ────────              ────────────           ─────────
固件            BIOS/UEFI             BIOS/UEFI              BIOS/UEFI
引导            GRUB2                 GRUB2                  GRUB2（带密码保护）
内核            3.10.x                5.15.x                 4.19/5.4/5.10
initramfs工具   dracut                initramfs-tools        initramfs-tools
init系统        systemd 219           systemd 249            systemd 249
桌面            GNOME 3               GNOME 42               UKUI（自研）
网络管理        NetworkManager        Netplan+NM             NetworkManager
安全特性        SELinux               AppArmor               SELinux + 自研安全模块
包管理          yum                   apt                    apt（兼容deb）
```

------

## 十、实用排查速查表

遇到启动问题，按这个顺序排查：

| 症状                           | 可能阶段    | 排查方法                                  |
| ------------------------------ | ----------- | ----------------------------------------- |
| 按电源没反应                   | 固件之前    | 检查电源、内存条                          |
| 卡在厂商Logo                   | 固件POST    | 拔掉外设，清CMOS                          |
| `No bootable device`           | 固件→引导   | 检查启动顺序，MBR/ESP是否完好             |
| 进入 `grub>` 或 `grub rescue>` | GRUB        | 手动指定内核路径，或用Live USB修复        |
| 内核panic（满屏英文）          | 内核        | 检查initramfs是否完整，内核参数是否正确   |
| 卡在 `Started ...` 某一行      | systemd服务 | `journalctl -b`、`systemctl --failed`     |
| 黑屏但有光标                   | 显示管理器  | `systemctl restart gdm`（或lightdm）      |
| 登录后桌面闪退                 | 桌面会话    | `~/.xsession-errors`，`journalctl --user` |

------

## 十一、写在最后

从CentOS 6、7到Ubuntu 22.04再到麒麟V10，我换了三个发行版，内核从3.10换到了5.15，桌面从GNOME换到了UKUI。但每次按下电源键，机器内部跑的那套流程，**本质上和十年前一样**：固件自检 → 
GRUB指路 → 内核解压 → initramfs探路 → systemd管家拉服务 → 你看见登录框。

理解了这条主线，不管以后换什么发行版、升什么内核版本，你心里都有底。出了问题，你知道该在哪个环节去查；看到陌生的报错，你能判断它属于哪个阶段。

这大概就是"入门之后，再往深处走一步"的意义。


------

> **参考与延伸阅读：**
>
> - `man bootup`（systemd官方启动流程文档）
> - `man systemd`、`man dracut`、`man update-initramfs`
> - 鸟哥《Linux私房菜：基础学习篇》第四版 第20章
> - kernel.org Documentation: `admin-guide/README.rst`
