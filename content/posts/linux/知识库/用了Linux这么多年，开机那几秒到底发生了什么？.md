---
title: "Linux启动流程全解析：从CentOS到Ubuntu再到麒麟V10服务器版"
subtitle: ""
date: 2024-03-28T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Linux"]
tags: ["Linux"]
---

## 写在前面

用Linux很多年了，从CentOS 6、7一路用到Ubuntu 22.04，最近又转战信创，开始部署银河麒麟Kylin V10**服务器版**。命令敲得越来越溜，`systemctl`、`journalctl`、`grub2-mkconfig` 这些张口就来，但有一天突然想到："Linux从按下电源键到你看见登录界面，到底经历了什么？"

我愣了一下，发现自己只能说出个大概。鸟哥的《私房菜》读过，入门够了，但入门这么久了，该往深处走走了。更重要的是，我发现自己虽然用过好几个发行版，却从来没有认真对比过它们在启动流程上的异同——尤其是麒麟V10服务器版，它和CentOS的血缘关系远比和Ubuntu近。

于是有了这篇文章。既是写给自己看的笔记，也希望能帮到和我一样"知其然，想知其所以然"的朋友。

------

## 一、先回答核心问题：启动流程到底一不一样？

### 一句话结论：**骨架完全相同，皮肉因发行版而异。**

不管你是CentOS 6/7、Ubuntu 22.04还是麒麟V10服务器版，Linux启动都遵循同一个五阶段模型：

```
固件（BIOS/UEFI）→ 引导加载器（GRUB）→ 内核+initramfs → init系统 → 服务与登录
```

这条主线从Linux诞生至今没有本质变化。**不同内核版本（3.x/4.x/5.x/6.x）之间也是如此**——架构不变，变的是驱动支持、安全特性和性能优化。

差异藏在每个阶段的"实现细节"里。下面逐阶段拆解，并在每一阶段明确标注三个系统的区别。

------

## 二、全景图：五个阶段的接力赛

```
按下电源键
    │
    ▼
┌──────────────────────────────┐
│ ① 固件阶段（BIOS / UEFI）    │ ← 主板上的"值班保安"
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ ② 引导加载器（GRUB）         │ ← "引路人"
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ ③ 内核启动（vmlinuz+initramfs）│ ← "真正的主角"
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ ④ 用户空间初始化（init系统）  │ ← "大管家"
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ ⑤ 服务启动 → 登录提示符      │ ← "你看到的世界"
└──────────────────────────────┘
```

------

## 三、阶段①：固件——BIOS与UEFI

### 通俗理解

按下电源键后，CPU通电执行的第一段代码不在硬盘上，而在主板固件芯片里。它像值班保安：先巡逻确认大楼没塌（硬件自检），再决定给谁开门（选择启动设备）。

### BIOS（传统）

- 执行POST（加电自检）：检测CPU、内存、显卡、硬盘。
- 按CMOS启动顺序找到第一个可启动设备。
- 读取该设备的**MBR（Master Boot Record，512字节）**：前446字节是引导程序，后64字节是分区表，最后2字节 `0x55AA` 是有效标志。
- 把控制权交给MBR里的引导程序（GRUB第一阶段）。

> 限制：最多4个主分区，磁盘寻址最大2TB。

### UEFI（现代）

- 使用GPT分区表（支持128个分区，磁盘可达9.4ZB）。
- 在磁盘上找**ESP分区**（EFI System Partition，FAT32格式，通常挂载在 `/boot/efi`），直接加载其中的 `.efi` 可执行文件。
- 支持Secure Boot：校验efi文件的数字签名。

### 三个系统的固件情况

| 项目        | CentOS 6 | CentOS 7      | Ubuntu 22.04 | 麒麟V10服务器版       |
| ----------- | -------- | ------------- | ------------ | --------------------- |
| 默认模式    | BIOS为主 | BIOS/UEFI并存 | UEFI为主     | UEFI为主（兼容BIOS）  |
| GPT支持     | 需手动   | 原生支持      | 原生支持     | 原生支持              |
| Secure Boot | 不支持   | 可选          | 默认启用     | **默认启用+国密扩展** |

> 麒麟V10服务器版在标准UEFI Secure Boot基础上，部分机型还支持国密算法的固件级验签，这是信创环境的特殊要求。

------

## 四、阶段②：引导加载器——GRUB

### 通俗理解

固件把接力棒交给GRUB。它的任务就一件：**让你选内核，然后把内核和initramfs加载到内存里**。

### GRUB做了什么

1. 显示启动菜单（多系统/多内核时弹出选择界面）。
2. 加载选定的 `vmlinuz`（压缩内核）和 `initramfs`（临时根文件系统）。
3. 传递内核启动参数（如 `root=`, `ro`, `quiet` 等）。
4. 把控制权交给内核。

### ⚠️ 关键区分：GRUB vs GRUB2

| 项目         | CentOS 6                           | CentOS 7 / 麒麟V10服务器版               | Ubuntu 22.04                          |
| ------------ | ---------------------------------- | ---------------------------------------- | ------------------------------------- |
| GRUB版本     | **GRUB Legacy (0.97)**             | **GRUB2 (2.x)**                          | GRUB2 (2.x)                           |
| 配置文件     | `/boot/grub/grub.conf`             | `/boot/grub2/grubenv`                    | `/boot/grub/grub.cfg`                 |
| 人类可读配置 | `/boot/grub/grub.conf`（直接编辑） | `/etc/default/grub` + `/etc/grub.d/*`    | `/etc/default/grub` + `/etc/grub.d/*` |
| 重建命令     | 直接编辑grub.conf                  | `grub2-mkconfig -o /boot/grub2/grub.cfg` | `update-grub`                         |
| UEFI efi路径 | 无（纯BIOS）                       | `/boot/efi/EFI/centos/grubx64.efi`       | `/boot/efi/EFI/ubuntu/grubx64.efi`    |
| GRUB密码保护 | 明文/MD5                           | PBKDF2加密                               | PBKDF2加密                            |
| 进入编辑模式 | 按p输入密码                        | 默认无需密码                             | 默认无需密码                          |

> **重要提醒**：如果你从CentOS 6升级到CentOS 7或麒麟V10服务器版，GRUB从Legacy变成了GRUB2，配置文件格式、重建命令、密码机制全部变了。这是很多人迁移时踩的第一个坑。

### 麒麟V10服务器版的GRUB特殊点

- 配置文件路径与CentOS 7完全一致：`/boot/grub2/grub.grubenv`
- 重建命令相同：`grub2-mkconfig -o /boot/grub2/grub.grubenv`
- 部分信创机型出厂预设了GRUB密码（防止篡改启动参数），编辑菜单需先认证
- 支持国密SM2/SM3签名的内核镜像校验（配合安全启动）

------

## 五、阶段③：内核启动——最核心的部分

### 通俗理解

GRUB把内核和"应急工具箱"塞进内存后说："兄弟，接下来交给你了。"内核从此接管整台机器。

### 5.1 vmlinuz是什么

- `vm` = Virtual Memory（历史命名）
- `linu` = Linux
- `z` = zlib压缩

它是一个被压缩的内核二进制文件，启动第一步就是自我解压。

### 5.2 initramfs（初始内存文件系统）

这是一个加载到内存中的**临时根文件系统**。为什么需要它？因为真正的根分区可能在LVM、RAID、加密卷上，或者需要特定驱动才能访问。initramfs提前打包了这些驱动和工具。

查看内容：

```bash
# CentOS 7 / 麒麟V10服务器版
lsinitrd /boot/initramfs-$(uname -r).img | head -30

# Ubuntu 22.04
lsinitramfs /boot/initrd.img-$(uname -r) | head -30
```

### 5.3 内核启动具体步骤

```
vmlinuz 解压
    │
    ▼
start_kernel()              ← 内核的 main()
    ├── 初始化CPU、内存管理、中断、调度器
    ├── 加载早期驱动（控制台、时钟、存储控制器）
    ├── 挂载 initramfs 为临时根文件系统
    ├── 启动 PID 1（init/systemd）
    │
    ▼
initramfs 中的初始化脚本执行
    ├── 探测硬件、加载存储模块（dm-mod, lvm2, raid等）
    ├── 组装真正的根文件系统
    ├── 以只读方式挂载真正的 /
    ├── switch_root：切换到真实根文件系统，释放initramfs
    │
    ▼
真实 / 下的 systemd 接管
```

### 5.4 不同内核版本的差异

| 变化点             | 说明                                 |
| ------------------ | ------------------------------------ |
| initrd → initramfs | 2.6以后统一用cpio格式，更灵活        |
| 压缩算法升级       | gzip → xz → zstd，解压更快           |
| 并行初始化         | 驱动异步探测，缩短启动时间           |
| 安全特性           | Lockdown模式、TPM度量、IMA完整性检查 |
| cgroup v2          | 资源管理更精细（影响systemd行为）    |

**但 `start_kernel()` → initramfs → PID 1 这条主线，从2.6到6.x从未改变。**

### 5.5 三个系统的内核对比如

| 项目          | CentOS 6   | CentOS 7    | Ubuntu 22.04          | 麒麟V10服务器版         |
| ------------- | ---------- | ----------- | --------------------- | ----------------------- |
| 默认内核      | 2.6.32     | 3.10.x      | 5.15.x                | 4.19.x / 5.10.x         |
| initramfs工具 | mkinitrd   | **dracut**  | initramfs-tools       | **dracut**              |
| 重建initramfs | `mkinitrd` | `dracut -f` | `update-initramfs -u` | `dracut -f`             |
| 内核源码基础  | RHEL 6     | RHEL 7      | Ubuntu HWE/LTS        | openEuler/CentOS        |
| 国密算法支持  | ❌          | ❌           | ❌                     | ✅ SM2/SM3/SM4内核级支持 |

> **关键发现**：麒麟V10服务器版的initramfs工具和CentOS 7一样都是 `dracut`，而Ubuntu用的是 `initramfs-tools`。这再次印证了麒麟V10服务器版与CentOS/openEuler的血缘关系。

------

## 六、阶段④：用户空间初始化——init系统的演进

### 通俗理解

内核搭好舞台后，"大管家"上场。它是PID 1，负责按预定义目标把所有服务拉起来。

### 6.1 init系统的三代变迁

| 时代   | init系统    | 代表                            | 特点                            |
| ------ | ----------- | ------------------------------- | ------------------------------- |
| 第一代 | SysVinit    | CentOS 6                        | `/etc/rc.d/rc*.d/` 脚本串行执行 |
| 第二代 | Upstart     | Ubuntu 14.04                    | 事件驱动，但仍兼容SysV          |
| 第三代 | **systemd** | CentOS 7+/Ubuntu 16.04+/麒麟V10 | 并行启动、依赖声明、单元化管理  |

> **你的经历正好跨越了三代**：CentOS 6是SysVinit，CentOS 7切到systemd，Ubuntu 22.04和麒麟V10服务器版都是systemd。

### 6.2 systemd的Target机制

systemd用Target替代了传统的runlevel：

```
default.target
    ├── graphical.target（图形界面，桌面版）
    │     └── multi-user.target（多用户命令行）
    │           └── basic.target（基础服务）
    │                 └── sysinit.target（系统初始化）
    │                       └── *.service / *.mount / *.socket
    │
    └── multi-user.target（服务器版默认）
          └── ...同上...
# 查看当前默认target
systemctl get-default

# 切换
systemctl set-default multi-user.target   # 改为命令行启动
systemctl set-default graphical.target    # 改为图形界面启动
```

### 6.3 三个系统的init系统对比

| 项目       | CentOS 6                      | CentOS 7               | Ubuntu 22.04          | 麒麟V10服务器版            |
| ---------- | ----------------------------- | ---------------------- | --------------------- | -------------------------- |
| init系统   | **SysVinit**                  | systemd 219            | systemd 249           | systemd 239                |
| 服务管理   | `service` / `chkconfig`       | `systemctl`            | `systemctl`           | `systemctl`                |
| 日志系统   | `/var/log/messages` + rsyslog | journald + rsyslog     | journald + rsyslog    | journald + rsyslog         |
| 网络配置   | ifcfg + network service       | NetworkManager / ifcfg | Netplan + NM/networkd | **NetworkManager / ifcfg** |
| 防火墙     | iptables                      | firewalld              | ufw/nftables          | **firewalld**              |
| MAC框架    | SELinux                       | SELinux                | AppArmor              | **SELinux**                |
| 默认target | runlevel 3/5                  | multi-user/graphical   | graphical             | **graphical**（服务器版）  |

> **重点**：麒麟V10服务器版的systemd版本（239）、SELinux、firewalld、ifcfg网络配置、dracut工具链，全部与CentOS 7/8对齐。如果你熟悉CentOS 7，迁移到麒麟V10服务器版几乎零学习成本。

------

## 七、阶段⑤：服务启动到登录

systemd按依赖链拉起所有服务后：

- **服务器版**（multi-user.target）：直接在tty1显示文字登录提示符
- **桌面版**（graphical.target）：启动显示管理器（GDM/LightDM），呈现图形登录界面

对于你使用的麒麟V10服务器版，看到的就是一行：

```
Kylin V10 Server login: _
```

------

## 八、你没提到但值得补充的内容

### 8.1 内核启动参数速查

在GRUB菜单按 `e` 编辑，常用参数：

| 参数                             | 作用                                  |
| -------------------------------- | ------------------------------------- |
| `root=/dev/mapper/kl-root`       | 指定根分区（LVM场景）                 |
| `ro` / `rw`                      | 初始只读/读写挂载                     |
| `single` / `rescue`              | 单用户/救援模式                       |
| `init=/bin/bash`                 | 跳过init直接进shell（忘记密码救命用） |
| `systemd.unit=multi-user.target` | 强制命令行启动                        |
| `selinux=0`                      | 临时禁用SELinux（排查问题时用）       |
| `rd.lvm.lv=kl/root`              | dracut专用：指定LVM逻辑卷             |
| `console=ttyS0,115200`           | 串口控制台输出（服务器远程调试必备）  |

### 8.2 启动性能分析三板斧

```bash
systemd-analyze                    # 总耗时
systemd-analyze blame              # 各服务耗时排名
systemd-analyze critical-chain     # 关键链依赖分析
systemd-analyze plot > boot.svg    # 可视化启动时序图
```

> CentOS 6（SysVinit）不支持这些命令，只有systemd才有。

### 8.3 启动日志排查

```bash
journalctl -b              # 本次启动全部日志
journalctl -b -p err       # 只看错误及以上
journalctl -u sshd -b      # 本次启动中sshd的日志
dmesg | tail -50           # 内核环形缓冲区（硬件/驱动问题首选）
cat /var/log/boot.log      # 部分发行版保留的传统启动日志
```

### 8.4 /boot目录解读

```bash
ls /boot/
# vmlinuz-4.19.xx-xx.ky10.aarch64   ← 压缩内核（麒麟ARM64示例）
# initramfs-4.19.xx-xx.ky10.aarch64.img ← initramfs
# config-4.19.xx-xx.ky10.aarch64    ← 内核编译配置
# System.map-4.19.xx-xx.ky10.aarch64 ← 符号表
# grub2/                             ← GRUB2配置和模块
# efi/                               ← UEFI引导文件
```

> 注意麒麟V10服务器版如果是ARM64架构（飞腾/鲲鹏处理器），内核文件名会带 `aarch64` 后缀，这与x86_64的CentOS/Ubuntu不同，但启动流程完全一致。

### 8.5 Secure Boot与国密

麒麟V10服务器版在信创环境下的特殊安全机制：

- **GRUB密码保护**：防止未授权修改启动参数
- **内核签名验证**：配合UEFI Secure Boot，只允许加载签名内核
- **国密算法支持**：SM2（非对称加密）、SM3（哈希）、SM4（对称加密）在内核和密码模块中原生支持
- **安全审计模块**：符合等保2.0三级要求的审计框架默认启用

这些不影响启动流程的骨架，但会在各阶段增加额外的校验步骤。

### 8.6 架构差异：x86_64 vs ARM64

麒麟V10服务器版常部署在飞腾、鲲鹏等ARM64平台上：

| 项目       | x86_64（CentOS/Ubuntu） | ARM64（麒麟V10）              |
| ---------- | ----------------------- | ----------------------------- |
| UEFI规范   | 标准UEFI                | UEFI + ACPI（或DTB）          |
| 内核镜像名 | vmlinuz-x.xx.x-x.x86_64 | vmlinuz-x.xx.x-x.ky10.aarch64 |
| EFI文件    | grubx64.efi             | **grubaa64.efi**              |
| 设备树     | 不需要                  | 可能需要DTB文件               |
| 启动流程   | 完全相同                | 完全相同                      |

> ARM64下UEFI可能通过Device Tree Blob（DTB）而非ACPI传递硬件信息，但对用户而言启动体验无异。

------

## 九、一张表总结：四个系统的启动流程对比

| 阶段          | CentOS 6               | CentOS 7               | Ubuntu 22.04          | 麒麟V10服务器版        |
| ------------- | ---------------------- | ---------------------- | --------------------- | ---------------------- |
| 固件          | BIOS                   | BIOS/UEFI              | UEFI                  | UEFI（+国密）          |
| GRUB          | Legacy 0.97            | GRUB2                  | GRUB2                 | GRUB2（+密码保护）     |
| GRUB配置      | `/boot/grub/grub.conf` | `/boot/grub2/grub.cfg` | `/boot/grub/grub.cfg` | `/boot/grub2/grub.cfg` |
| 默认内核      | 2.6.32                 | 3.10.x                 | 5.15.x                | 4.19/5.10              |
| initramfs工具 | mkinitrd               | dracut                 | initramfs-tools       | **dracut**             |
| init系统      | SysVinit               | systemd 219            | systemd 249           | systemd 239            |
| 服务管理      | service/chkconfig      | systemctl              | systemctl             | systemctl              |
| 网络配置      | ifcfg                  | ifcfg/NM               | Netplan               | **ifcfg/NM**           |
| MAC框架       | SELinux                | SELinux                | AppArmor              | **SELinux**            |
| 防火墙        | iptables               | firewalld              | ufw                   | **firewalld**          |
| 包管理        | yum                    | yum                    | apt                   | **yum/dnf**            |
| 默认target    | runlevel 3             | multi-user             | graphical             | **graphical**          |

> 可以清晰看到：**麒麟V10服务器版在几乎所有技术选型上都与CentOS 7/8对齐**，而非Ubuntu系。从CentOS 7迁移过来，启动流程的理解可以无缝复用。

------

## 十、故障排查速查表

| 症状                     | 可能阶段              | 排查方法                                      |
| ------------------------ | --------------------- | --------------------------------------------- |
| 按电源无反应             | 固件之前              | 检查电源、内存、BMC/IPMI                      |
| 卡在厂商Logo             | POST                  | 拔外设、清CMOS、查BMC日志                     |
| No bootable device       | 固件→引导             | 检查启动顺序、MBR/ESP完整性                   |
| `grub>` / `grub rescue>` | GRUB                  | 手动指定内核路径，Live USB修复                |
| Kernel panic             | 内核/initramfs        | 检查initramfs完整性、内核参数、`rd.break`调试 |
| 卡在某个 `Started ...`   | systemd服务           | `journalctl -b`、`systemctl --failed`         |
| 登录提示符不出现         | getty/display-manager | `systemctl status getty@tty1`                 |
| SELinux阻止服务          | MAC策略               | `ausearch -m avc`、`sealert`                  |

> 麒麟V10服务器版额外关注：`sealert` 分析SELinux拒绝、`kysec-status` 检查麒麟安全模块状态。

------

## 十一、写在最后

从CentOS 6的SysVinit，到CentOS 7的systemd，再到Ubuntu 22.04和麒麟V10服务器版，我跨越了Linux init系统的三个时代、两种包管理体系、x86和ARM两种架构。但每次按下电源键，机器内部跑的那套流程，本质上和二十年前一样：

**固件自检 → GRUB指路 → 内核解压 → initramfs探路 → systemd拉服务 → 登录提示符出现。**

理解了这条主线，不管以后换什么发行版、升什么内核、迁到什么架构，心里都有底。出了问题知道在哪个环节查，看到陌生报错能判断属于哪个阶段。

特别是麒麟V10服务器版，它和CentOS 7/8的技术血缘意味着：**你在CentOS上积累的启动流程知识，几乎可以原封不动地迁移过来**。唯一的增量，是信创环境带来的安全加固层（GRUB密码、国密验签、安全审计），它们叠加在标准流程之上，而非替换它。

这大概就是"入门之后，再往深处走一步"的意义。

------

> **参考与延伸阅读：**
>
> - `man bootup`（systemd官方启动流程文档）
> - `man dracut`、`man systemd`
> - 鸟哥《Linux私房菜：基础学习篇》第四版 第20章
> - kernel.org Documentation: `admin-guide/README.rst`
> - 麒麟软件知识库：[https://kb.kylinos.cn](https://kb.kylinos.cn/)
> - openEuler文档：[https://docs.openeuler.org](https://docs.openeuler.org/)