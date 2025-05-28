---
title: "Linux系统Nvidia驱动部署"
subtitle: ""
date: 2025-01-11T12:06:37+08:00
lastmod: 2025-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["GPU"]
tags: ["GPU"]
---
## 背景
部署通义千问大模型`qwen2.5-7b-awq`基础环境预安装

## 服务器配置
- 操作环境：openEuler 22.03 (欧拉-LTS-SP1)
- cpu: 40c        memory: 186G  disk: 2T
- GPU物理机，显卡型号为NVIDIA Tesla T4 (Tesla T4 被广泛应用于云端推理服务、虚拟桌面基础设施(VDI)、视频转码等领域)

查看 GPU 型号
```
$ ls  /proc/driver/nvidia/gpus/ 
0000:3b:00.0
$ cat 0000\:3b\:00.0/information 
Model:           Tesla T4
IRQ:             218
GPU UUID:        GPU-e42994d9-5240-0163-fbf5-deb9d57a348c
Video BIOS:      90.04.38.00.03
Bus Type:        PCIe
DMA Size:        47 bits
DMA Mask:        0x7fffffffffff
Bus Location:    0000:3b:00.0
Device Minor:    0
GPU Firmware:    570.124.06
GPU Excluded:    No
```

### 检测主机是否为虚拟化环境
`systemd-detect-virt` 是一个专门用于检测虚拟化环境的工具。

```bash
systemd-detect-virt
```
- 如果是物理机，输出为：
```
none
```
- 如果是虚拟机，输出可能是：
```
kvm
vmware
virtualbox
xen
```

## 1. 检查仓库源
```
# debian
$ sudo apt install -y g++ gcc gcc-c++ make build-essential pciutils  bzip2 dkms
# redhat
$ sudo yum install -y g++ gcc gcc-c++  make  pciutils  bzip2 dkms
```
查看是否已安装gcc和picutils
```
# 验证系统是否有GCC编译环境
$ gcc --version
$ lspci
```
查看内核版本和GPU
```
$ uname -a
# 查找与 NVIDIA 相关的硬件设备显卡
$ lspci | grep -i nvidia
```
### 内核版本检查
Redhat
```
$ uname -r
3.10.0-1160.119.1.el7.x86_64
$ yum list installed|grep kernel
kernel-headers-3.10.0-1160.108.1.el7.x86_64
kernel-devel-3.10.0-1160.71.1.el7.x86_64
kernel-3.10.0-1160.119.1.el7.x86_64
kernel-3.10.0-1160.el7.x86_64
kernel-tools-libs-3.10.0-1160.119.1.el7.x86_64
kernel-tools-3.10.0-1160.119.1.el7.x86_64
```
{{< admonition type=note >}}
查看内核版本是否匹配，如果匹配则跳过重新安装，不匹配需要重新安装进行匹配
{{< /admonition >}}


在线安装
```
# optional可选
sudo yum remove kernel-devel-$(uname -r) kernel-headers-$(uname -r)
# 安装
sudo yum install kernel-devel-$(uname -r) kernel-headers-$(uname -r)
```
离线安装下载centos源码包
```
devel地址：http://rpmfind.net/linux/rpm2html/search.php?query=kernel-devel
headers地址：http://rpmfind.net/linux/rpm2html/search.php?query=kernel-headers
```
安装
```
$ rpm -ivh kernel-devel-3.10.0-1160.119.1.el7.x86_64.rpm --nodeps --force
$ rpm -ivh kernel-rpm -ivh kernel-headers-3.10.0-1160.119.1.el7.x86_64.rpm --force
```



## 2. 检查显卡
查看显卡信息和驱动信息
```
# 查看主显卡是哪一个

# 为了查看nvidia 显卡，可以安装nvidia-detect,并运行此软件
# debian
$ apt-get install nvidia-detect
# Redhat
$ lspci |grep -i vga
03:00.0 VGA compatible controller: ASPEED Technology, Inc. ASPEED Graphics Family (rev 41)
# 查找与 VGA（视频图形阵列）兼容的显示控制器的信息
$ lspci -s 03:00.0 -v
03:00.0 VGA compatible controller: ASPEED Technology, Inc. ASPEED Graphics Family (rev 41) (prog-if 00 [VGA controller])
        DeviceName:  Onboard IGD
        Subsystem: ASPEED Technology, Inc. ASPEED Graphics Family
        Flags: bus master, medium devsel, latency 0, IRQ 17, NUMA node 0
        Memory at 98000000 (32-bit, non-prefetchable) [size=64M]
        Memory at 9c000000 (32-bit, non-prefetchable) [size=128K]
        I/O ports at 2000 [size=128]
        Expansion ROM at 000c0000 [virtual] [disabled] [size=128K]
        Capabilities: [40] Power Management version 3
        Capabilities: [50] MSI: Enable- Count=1/2 Maskable- 64bit+
        Kernel driver in use: ast
        Kernel modules: ast
```

## 3. 安装
安装先检查有没有 cuda 支持的GPU
```
$ lspci | grep -i nvidia
3b:00.0 3D controller: NVIDIA Corporation TU104GL [Tesla T4] (rev a1)
```
### Nouveau开源图形驱动禁用
后面安装时的告警信息，提前预知
```
WARNING: The Nouveau kernel driver is currently in use by your system.  This driver is incompatible with
the NVIDIA driver，and must be disabled before proceeding.
```
警告：您的系统当前正在使用Nouveau内核驱动程序。此驱动程序与NVIDIA驱动程序不兼容，必须先禁用才能继续。

解释：  
Nouveau是由第三方为NVIDIA显卡提供的开源图形驱动，但它与NVIDIA官方提供的驱动可能不兼容，尤其在运行特定的游戏或应用程序时可能会出现性能问题。如果系统报告Nouveau驱动正在使用中，可能意味着你的系统没有使用NVIDIA官方提供的驱动，这可能会降低图形性能或者导致系统不稳定。

解决方法：  
1. 禁用Nouveau驱动：  
Nouveau 是一个开源的图形驱动程序，用于支持 NVIDIA 显卡在 Linux 系统上的使用。它是由社区开发的，旨在提供对 NVIDIA 显卡的开源驱动支持，以便在 Linux 系统上实现图形加速和其他相关功能。在安装专有的 NVIDIA 驱动之前，通常需要禁用 Nouveau 驱动

创建文件/etc/modprobe.d/blacklist-nouveau.conf，并添加以下内容：
```
# 查看nouveau 是否被禁用，如果没有任何输出，则说明 Nouveau 已被成功禁用
$ lsmod | grep nouveau 
nouveau              2424832  0
mxm_wmi                16384  1 nouveau
video                  61440  1 nouveau
ttm                   118784  3 drm_vram_helper,drm_ttm_helper,nouveau
drm_kms_helper        286720  5 drm_vram_helper,ast,nouveau
drm                   638976  7 drm_kms_helper,drm_vram_helper,ast,drm_ttm_helper,ttm,nouveau
i2c_algo_bit           16384  3 igb,ast,nouveau
wmi                    36864  2 mxm_wmi,nouveau

# 从未手动禁用 Nouveau 驱动程序（开源的 NVIDIA 显卡驱动），则该文件不会存在，直接添加配置内容即可
$ sudo vi /etc/modprobe.d/blacklist-nouveau.conf
#将nvidiafb注释掉:
#blacklist nvidiafb

# 添加以下内容
blacklist nouveau
options nouveau modeset=0
```
禁用Nouveau驱动后更新initramfs：

修改模块黑名单后，需要更新 initramfs 以确保更改生效

```
# debian
sudo update-initramfs -u
# redhat
sudo dracut --force
```

备份当前的镜像并建立新的镜像
```
#备份当前的镜像
$ sudo mv /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak
#建立新的镜像
$ sudo dracut /boot/initramfs-$(uname -r).img $(uname -r)
# 重启,不重启 Nouveau 模块可能仍然被加载到内核中，影响nvidia 驱动的安装
$ sudo reboot
#最后输入上面的命令验证
$ lsmod | grep nouveau
```

2. 要卸载nouveau驱动并清除相关配置文件，请执行以下命令：(可选)
禁用掉即可
```
# debian
sudo apt-get remove --purge nouveau
# redhat
sudo yum t remove --purge nouveau
```

### 安装Nvidia驱动
GPU 云服务器正常工作需安装正确的基础设施软件，对 NVIDIA 系列 GPU 而言，有两个层次的软件包需要安装：

- 驱动 GPU 工作的硬件驱动程序。
- 上层应用程序所需要的库。
若把 NVIDIA GPU 用作通用计算，需要安装 GeForce Driver + CUDA。

####  处理 Secure Boot（可选）
若系统启用 Secure Boot，需为 NVIDIA 内核模块签名或禁用 Secure Boot（需进入 BIOS 设置）。

禁用 Secure Boot 会降低系统对恶意内核代码的防护能力，但允许加载未签名的驱动（如 NVIDIA 驱动）

 **方法1. 检查是否启用 Secure Boot**
```shell
# 安装 mokutil（如果未安装）
$ sudo yum install mokutil -y
# 查看 Secure Boot 状态
$ sudo mokutil --sb-state
SecureBoot disabled
Platform is in Setup Mode
```
- 输出结果：  
  - **`SecureBoot enabled`** 表示已启用。  
  - **`SecureBoot disabled`** 表示已禁用。

**方法2.检查 UEFI 变量文件**
```bash
# 确认系统使用 UEFI 启动（非传统 BIOS）
ls /sys/firmware/efi

# 查看 Secure Boot 状态
cat /sys/firmware/efi/efivars/SecureBoot-* 2>/dev/null
```
- 输出 **`1`** 表示启用，**`0`** 表示禁用。

##### 禁用 Secure Boot
Secure Boot 是 UEFI 固件的功能，禁用需进入主板的 **BIOS/UEFI 设置界面**。以下是通用步骤：

**步骤 1：重启并进入 BIOS/UEFI 设置**
1. 重启计算机，在开机自检界面按下 **特定按键**（常见按键：`F2`、`F10`、`Del`、`Esc`，具体取决于主板型号）。  
   - 对于虚拟机（如 VMware/VirtualBox），需在启动时按 **`Esc` 或 `F2`** 进入设置。

** 步骤 2：找到 Secure Boot 选项**
1. 在 BIOS/UEFI 界面中，导航到 **`Security`** 或 **`Boot`** 选项卡。  
2. 查找 **`Secure Boot`** 选项（可能位于 `Advanced` 或 `Authentication` 子菜单中）。

**步骤 3：禁用 Secure Boot**
1. 将 **`Secure Boot`** 设为 **`Disabled`**。  
2. 保存设置并退出（通常按 **`F10`** 或选择 **`Save & Exit`**）。

**步骤 4：验证禁用结果**
重启后回到 CentOS 7，再次运行检查命令（如 `sudo mokutil --sb-state`），确认输出为 **`SecureBoot disabled`**。



####  安装驱动后缀分为.run和.rpm
1.打开 NVIDIA 驱动下载链接 [Advanced Driver Search | NVIDIA](https://www.nvidia.cn/drivers/lookup/) 。

2.选择支持 RPM 或者RUN的操作系统，并获取该包的下载链接。例如：选择 CentOS 7.x， 得到下载链接：[Download NVIDIA, GeForce, Quadro, and Tesla Drivers](https://www.nvidia.cn/geforce/drivers/) 可选安装

3. 安装NVIDIA官方驱动
关闭图形界面以避免冲突：
```
sudo systemctl isolate multi-user.target
```

运行安装脚本：

确定你的显卡型号。
从NVIDIA官网下载对应显卡的驱动程序。
安装驱动请查看以下文章：

```
$ chmod +x NVIDIA-Linux-x86_64-(版本号).run
$ sudo ./NVIDIA-Linux-x86_64-570.124.06.run
1: Multiple kernel module types are available for this system. Which would you like to use? - 选择 NVIDIA Proprietary
2. Building kernel modules  进度条...
3. WARNING: nvidia-installer was forced to guess the X library path '/usr/lib64' and X module path '/usr/lib64/xorg/modules'; these paths were not queryable from the system.  If X fails
   to find the NVIDIA X driver module, please install the `pkg-config` utility and the X.Org SDK/development package for your distribution and reinstall the driver. 警告可以忽略
4. Install NVIDIA's 32-bit compatibility libraries? 根据需要自行选择,这里选择兼容
5.  Would you like to register the kernel module sources with DKMS? This will allow DKMS to automatically build a new module, if your kernel changes later. 选择YES
6. Registering the kernel modules with DKMS: 进度条...
7. Installation of the NVIDIA Accelerated Graphics Driver for Linux-x86_64 (version: 570.124.06) is now complete.  安装完成

```
- 按提示操作，选择以下选项：
  - `Yes` 安装 DKMS（需提前安装 `dkms` 包）。前面步骤已经提前预安装
  - `Yes` 覆盖已有的 Nouveau 配置。
  - `Yes` 安装 32 位兼容库（如需）。

备注：
1. DKMS: 
DKMS（Dynamic Kernel Module Support）是一个框架，用于在内核更新时自动重新编译和安装内核模块。
```
Would you like to register the kernel module sources with DKMS? This will allow DKMS to automatically build a new module when a new kernel is installed.
```
- 这是询问您是否希望将 NVIDIA 驱动模块注册到 DKMS（Dynamic Kernel Module Support）中。
- 如果选择“是”，DKMS 将自动为未来的内核版本重新编译和安装 NVIDIA 驱动模块。
2. X.Org 
是否修改 X.Org 配置文件
```
Would you like to run the nvidia-xconfig utility to automatically update your X configuration file so that the NVIDIA X driver is used when you restart X?
```
- 如果您计划使用图形界面，建议选择“是”。这将确保 X Server 启动时使用 NVIDIA 驱动。
3. 32位兼容库
如果您在一个 64 位系统上安装驱动，安装程序可能会询问是否安装 32 位兼容库：
```
Install NVIDIA's 32-bit compatibility libraries?
```
如果您需要运行 32 位应用程序（例如某些游戏或旧版软件），请选择“是”。

### 安装报错
```
ERROR: Unable to find the kernel source tree for the currently running kernel.  Please make sure you have installed the kernel source files for your kernel and that they are properly  
  configured; on Red Hat Linux systems, for example, be sure you have the 'kernel-source' or 'kernel-devel' RPM installed.  If you know the correct kernel source files are 
  installed, you may specify the kernel source path with the '--kernel-source-path' command line option.
```

查看一下内核版本是否一致

### 验证安装

重新启动系统
```
$ sudo reboot
# 查看显卡信息，验证驱动是否已成功安装
# 检查主机 GPU 和 NVIDIA 驱动是否正常工作，返回 GPU 的详细信息（如型号、驱动版本、显存使用情况等），说明驱动安装成功
$ nvidia-smi # 显示 GPU 状态
Tue Mar 25 16:23:30 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.124.06             Driver Version: 570.124.06     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla T4                       Off |   00000000:3B:00.0 Off |                    0 |
| N/A   70C    P0             32W /   70W |       1MiB /  15360MiB |      4%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+

# 检查 NVIDIA 模块是否加载
$ lsmod | grep nvidia  # 确认模块已加载

# 查看 GPU 资源的使用情况
$ nvidia-smi pmon -s u -d 1
# gpu         pid   type     sm    mem    enc    dec    jpg    ofa    command 
# Idx           #    C/G      %      %      %      %      %      %    name 
    0      43043     C      0      0      -      -      -      -    funasr-wss-serv
    0      43043     C      0      0      -      -      -      -    funasr-wss-serv
```

{{< admonition type=note >}}
需要注意的是，第一次安装显卡驱动的话，是不用重启服务器的，后续更新驱动版本的话，则是需要的。但是建议第一次安装驱动之后，最好还是重启下，防止意外情况的出现和发生。
{{< /admonition >}}

### 安装 CUDA（可选）
CUDA (Compute Unified Device Architecture) 是显卡厂商 NVIDIA 推出的运算平台。 CUDA™ 是一种由 NVIDIA 推出的通用并行计算架构，该架构使 GPU 能够解决复杂的计算问题。 它包含了 CUDA 指令集架构（ISA）以及 GPU 内部的并行计算引擎。 开发人员现在可以使用 C 语言, C++ , FORTRAN 来为 CUDA™ 架构编写程序，所编写出的程序可以在支持 CUDA™ 的处理器上以超高性能运行。
GPU 云服务器采用 NVIDIA 显卡，需要安装 CUDA 开发运行环境。

1.[CUDA驱动下载](https://developer.nvidia.com/cuda-75-downloads-arch) 
2.选择操作系统和安装包，以CentOS 7为例

```
Installation Instructions:
`sudo rpm -i cuda-repo-rhel7-7-5-local-7.5-18.x86_64.rpm`
`sudo yum clean all`
`sudo yum install cuda`
```
在/usr/local/cuda/samples/1_Utilities/deviceQuery目录下，执行make命令，可以编译出deviceQuery程序。
```
$ cd /usr/local/cuda/samples/1_Utilities/deviceQuery
$ make 
```
执行deviceQuery正常显示设备信息，此刻认为CUDA安装正确。

{{< admonition type=note >}}
如果使用rpm文件报错，则考虑使用run文件进行安装(runfile(local))
{{< /admonition >}}


```
# 下载run文件进行安装
sh cuda_*.run
```
建议最好不要使用GUI图形化界面操作，容易报错。


####  驱动程序和 CUDA 的关系：

可以把 **驱动程序（Driver）** 和 **CUDA** 想象成 **“汽车引擎”** 和 **“赛车配件包”** 的关系：

- **驱动程序（Driver）**：  
  就像汽车的 **引擎**，它让 GPU 硬件能够被操作系统识别并基础运行（如显示画面、基础计算）。没有引擎，汽车根本无法启动。

- **CUDA**：  
  则像一套 **赛车专用配件包**（如涡轮增压、高性能变速箱、赛用轮胎等），它提供了开发高性能计算程序所需的工具和接口（如并行计算库、编译器）。只有安装了 CUDA，你才能利用 GPU 的算力做深度学习、科学计算等复杂任务。

---

####  是否需要安装 CUDA？取决于你的需求
1. **如果只用 GPU 显示图形（如日常办公、游戏）**：  
   安装驱动程序就够了，不需要 CUDA。

2. **如果要做 GPU 计算（如深度学习、科学计算）**：  
   必须安装 CUDA，因为：  
   - 深度学习框架（PyTorch、TensorFlow）依赖 CUDA 调用 GPU 算力。  
   - CUDA 提供了 `nvcc` 编译器、`cuBLAS` 等加速库，是开发 GPU 程序的基石。

---

####  驱动和 CUDA 的版本依赖关系
- **CUDA 版本依赖驱动程序版本**：  
  每个 CUDA 版本需要 **最低版本的驱动程序** 支持（见 [NVIDIA CUDA 文档](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)）。例如：  
  - CUDA 12.x 需要驱动版本 ≥ 525.60.13  
  - CUDA 11.x 需要驱动版本 ≥ 450.80.02  

  你安装的驱动版本 **570.124.06** 支持大部分 CUDA 版本（具体需查兼容性表）。

- **建议安装方式**：  
  1. 先安装驱动（已装好）。  
  2. 再根据需求安装 **CUDA Toolkit**（例如通过 NVIDIA 官网或包管理器）。  

---

####  如何安装 CUDA？
1. **查看驱动支持的 CUDA 版本**：  
   ```bash
   nvidia-smi  # 输出顶部会显示支持的 CUDA 版本（例如：CUDA Version: 12.2）
   ```

2. **从 NVIDIA 官网下载对应 CUDA Toolkit**：  
   [CUDA Toolkit 下载页面](https://developer.nvidia.com/cuda-toolkit-archive)  
   选择与驱动兼容的版本，按官方文档安装（通常用 `runfile` 或包管理器）。

3. **验证安装**：  
   ```bash
   nvcc --version  # 查看 CUDA 编译器版本
   nvidia-smi      # 确认驱动和 CUDA 版本兼容
   ```

---

####  总结
- **驱动程序**：让 GPU 能被系统使用（基础运行）。  
- **CUDA**：让开发者能利用 GPU 做高性能计算（功能扩展）。  
- **关系**：驱动是“汽油”，CUDA 是“赛车改装套件”——想飙车？两者缺一不可！

## 安装 NVIDIA container toolkit 配套工具
**NVIDIA Container Toolkit介绍：**
NVIDIA Container Toolkit(容器工具包)使用户能够构建和运行 GPU 加速的容器，该工具包括一个容器运行时库和实用程序，用于自动配置容器以利用 NVIDIA GPU。

**使用背景**
在使用大模型的时候，很多情况需要经常更新模型或者使用平台，所以时常更新代码或环境。
但是若在实体机更新导致环境混乱，维护不便，所以使用 docker 环境进行更新。
使用 docker 运行模型的时候要让其支持 GPU 需要使用 nvidia-container-toolkit 工具。

NVIDIA Container Toolkit 为容器化 GPU 加速工作提供了高效且便捷的解决方案，是深度学习和科学计算等场景的核心工具之一。


**核心功能**
1. GPU 加速支持：

- 将主机的 NVIDIA GPU 映射到容器中，使得容器能够直接访问 GPU 资源。

2. 轻松管理 CUDA 环境：

- 自动加载必要的 NVIDIA 驱动程序、CUDA 库和运行时，简化环境配置。

3. 与容器平台集成：

- 支持 Docker 和其他容器运行时（如 containerd）。

4. 灵活的 GPU 分配：

- 允许用户指定容器可以访问的 GPU 数量或特定 GPU。

**组件结构**
1. NVIDIA Container Runtime：

- Docker 运行时的扩展，用于在容器中加载 GPU 相关组件。

2. NVIDIA Container CLI：

- 提供了命令行工具 nvidia-container-cli，可以用于调试和管理 GPU 映射。

3. 支持的库和工具：

- 自动为容器挂载 libcuda.so 和其他相关库，使其支持 GPU 加速。

**前提条件**
- NVIDIA 驱动程序：确保主机已安装支持的 NVIDIA 驱动程序。
- Docker：安装 Docker，版本需为 19.03 或更高。


检查`nvidia-container-toolkit`  NVIDIA 容器工具包是否安装
```
# debian
$ dpkg -l | grep nvidia-container-toolkit
# redhat
$ rpm -l | grep nvidia-container-toolkit
# 添加 NVIDIA 容器运行时的官方存储库
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
# 在线安装，在线安装异常可以尝试离线安装
sudo yum install -y nvidia-container-toolkit
```

离线下载安装
```shell
$ wget https://nvidia.github.io/libnvidia-container/stable/rpm/x86_64/libnvidia-container1-1.14.6-1.x86_64.rpm
$ wget https://nvidia.github.io/libnvidia-container/stable/rpm/x86_64/libnvidia-container-tools-1.14.6-1.x86_64.rpm
$ wget https://nvidia.github.io/libnvidia-container/stable/rpm/x86_64/nvidia-container-toolkit-base-1.14.6-1.x86_64.rpm
$ wget https://nvidia.github.io/libnvidia-container/stable/rpm/x86_64/nvidia-container-toolkit-1.14.6-1.x86_64.rpm

#手动安装 RPM 包,如果您是手动安装 .rpm 文件，确保版本匹配后执行以下命令：
$ sudo rpm -ivh libnvidia-container1-1.14.6-1.x86_64.rpm
$ sudo rpm -ivh libnvidia-container-tools-1.14.6-1.x86_64.rpm
$ sudo rpm -ivh nvidia-container-toolkit-base-1.14.6-1.x86_64.rpm
$ sudo rpm -ivh nvidia-container-toolkit-1.14.6-1.x86_64.rpm
```
测试
```
$ nvidia-ctk --version
NVIDIA Container Toolkit CLI version 1.14.6
commit: 5605d191332dcfeea802c4497360d60a65c7887e

$ nvidia-container-cli --version
cli-version: 1.14.6
lib-version: 1.14.6
build date: 2024-02-27T20:52+0000
build revision: d2eb0afe86f0b643e33624ee64f065dd60e952d4
build compiler: gcc 4.8.5 20150623 (Red Hat 4.8.5-44)
build platform: x86_64

build flags: -D_GNU_SOURCE -D_FORTIFY_SOURCE=2 -DNDEBUG -std=gnu11 -O2 -g -fdata-sections -ffunction-sections -fplan9-extensions -fstack-protector -fno-strict-aliasing -fvisibility=hidden -Wall -Wextra -Wcast-align -Wpointer-arith -Wmissing-prototypes -Wnonnull -Wwrite-strings -Wlogical-op -Wformat=2 -Wmissing-format-attribute -Winit-self -Wshadow -Wstrict-prototypes -Wunreachable-code -Wconversion -Wsign-conversion -Wno-unknown-warning-option -Wno-format-extra-args -Wno-gnu-alignof-expression -Wl,-zrelro -Wl,-znow -Wl,-zdefs -Wl,--gc-sections

# 查看nvidia-container-toolkit版本
$ nvidia-container-runtime --version
NVIDIA Container Runtime version 1.14.6
commit: 5605d191332dcfeea802c4497360d60a65c7887e
spec: 1.2.0

runc version 1.1.12
commit: v1.1.12-0-g51d5e94
spec: 1.0.2-dev
go: go1.21.11
libseccomp: 2.5.3

$ nvidia-container-toolkit --version
NVIDIA Container Runtime Hook version 1.14.6
commit: 5605d191332dcfeea802c4497360d60a65c7887e
```
{{< admonition type=note >}}
- docker v20+ 以上用 nvidia-container-toolkit 代替 nvidia-docker2
- nvidia-container-runtime现在叫 nvidia-container-toolkit
{{< /admonition >}}

## 容器运行时配置
### Docker 配置
```shell
$ nvidia-ctk runtime configure --runtime=docker
INFO[0000] Loading config from /etc/docker/daemon.json  
INFO[0000] Wrote updated config to /etc/docker/daemon.json 
INFO[0000] It is recommended that docker daemon be restarted. 
```

这个命令实际上是对 docker 的配置文件/etc/docker/daemon.json进行修改，内容如下：
```
# 新增配置：
{
  ...
  "storage-driver": "overlay2",
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
  ...
}
```
修改后重启docker
```
$ sudo systemctl daemon-reload 
$ systemctl restart docker containerd
# 验证容器运行时
$ docker info | grep "Runtimes"
 Runtimes: io.containerd.runc.v2 io.containerd.runtime.v1.linux nvidia runc
```
### 在 docker-compose.yml 中添加支持 GPU
仅供参考
```
services:
  ollama:
    image: ollama/ollama:0.5.4
    container_name: ${CONTAINER_NAME}
    restart: unless-stopped
    ports:
      - ${PANEL_APP_PORT_HTTP}:11434
    networks:
      - 1panel-network
    tty: true
    privileged: true
    volumes:
      - ./data:/root/.ollama
    labels:
      createdBy: "Apps"
    # 添加 GPU 支持
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]
    # 如果使用 NVIDIA Container Toolkit，可以添加以下环境变量
    environment:
      NVIDIA_VISIBLE_DEVICES: all
      NVIDIA_DRIVER_CAPABILITIES: "compute,utility"

networks:
  1panel-network:
    external: true
```

### Containerd 配置
`# nvidia-ctk runtime configure --runtime=containerd`

这个命令实际上是对 containerd 的配置文件 /etc/containerd/config.toml 进行修改，内容如下：
```
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "nvidia"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/bin/nvidia-container-runtime"
```

### 在线或者离线使用国外代理
```shell
# 在线
cd /etc/yum.repos.d/
wget https://mirrors.ustc.edu.cn/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo
sed -i  's#nvidia.github.io#mirrors.ustc.edu.cn#g' nvidia-container-toolkit.repo
yum makecache
yum install -y nvidia-container-toolkit

# 离线
$ cat >>nvidia-container-toolkit.repo<<EOF
[nvidia-container-toolkit]
name=nvidia-container-toolkit
baseurl=https://mirrors.ustc.edu.cn/libnvidia-container/stable/rpm/\$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://mirrors.ustc.edu.cn/libnvidia-container/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt

[nvidia-container-toolkit-experimental]
name=nvidia-container-toolkit-experimental
baseurl=https://mirrors.ustc.edu.cn/libnvidia-container/experimental/rpm/$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=0
gpgkey=https://mirrors.ustc.edu.cn/libnvidia-container/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
yum install -y nvidia-container-toolkit
```
### nvidia docker, nvidia docker2, nvidia container toolkits区别
在docker容器中用GPU时，查阅了网上许多教程，教程之间概念模糊不清，相互矛盾，过时的教程和新的教程混杂在一起。主要原因是Nvidia为docker容器的支持发生了好几代变更，api发生了不少变化。

**省流版总结**  
凡是使用了命令nvidia docker或者在docker中引入了--runtime=nvidia参数的都是过时教程，最新方法只需要下载nvidia-container-toolkits，在docker中引入--gpus参数即可。

#### nvidia docker(不推荐)
nvidia docker是NVIDIA第一代支持docker容器内使用GPU资源的项目。运行时用nvidia-docker命令。

根据nvidia docker在github ( https://github.com/NVIDIA/nvidia-docker )上的描述，已经不再使用了

#### nvidia docker2(不推荐)
nvidia docker2是NVIDIA第二代支持docker容器内使用GPU资源的项目。运行时用nvidia-docker命令，且需要指定参数--runtime=nvidia.

根据 github (https://github.com/NVIDIA/nvidia-docker#backward-compatibility) 的描述，一代和二代之间有如下兼容性。
```
Backward compatibility To help transitioning code from 1.0 to 2.0, a bash script is provided in /usr/bin/nvidia-docker for backward compatibility. It will automatically inject the --runtime=nvidia argument and convert NV_GPU to NVIDIA_VISIBLE_DEVICES.
```
也就是说，在二代中，既可以使用nvidia docker命令，这会自动引入参数--runtime=nvidia也可以使用docker命令，手动指定参数--runtime=nvidia

#### nvidia-container-toolkits(推荐)
根据github这是最新的支持方案，如帖子描述，nvidia docker2 被Nvidia container toolkits取代，无需指定--runtime参数，只需要传递--gpus参数

## k8s 部署nvidia设备插件(扩展)
```
$ kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta4/nvidia-device-plugin.yml
# 或者安装 NVIDIA GPU Operator，适用K8S版本 1.23版本以上
# 添加 NVIDIA Helm repository
$ helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update
$ helm install -n gpu-operator --create-namespace gpu-operator nvidia/gpu-operator \
  --set driver.enabled=false \
  --set dcgmExporter.enabled=true \
  --set migManager.enabled=true \
  --set driver.vgpu.enabled=true \
  --set migStrategy=mixed
```
### k8s 调度GPU 任务测试
**测试一`gpu.yaml`**

```
apiVersion: v1
kind: Pod
metadata:
  name: tf-pod
spec:
  containers:
  - name: tf-container
    image: tensorflow/tensorflow:2.14.0-gpu
    command: [ "/bin/sh" ]
    args: [ "-c", "tail -f" ]
    resources:
      limits:
      cpu: 200m
      memory: 512Mi
      nvidia.com/gpu: 1 # requesting 1 GPU
```

**测试二 `cat cuda-vectoradd.yaml`**

```
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vectoradd
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda-vectoradd
    image: "nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04"
    resources:
       limits:
         nvidia.com/gpu: 1  # requesting 1 GPU
  nodeSelector:
    accekerator: nvidia-rtx4090
```
**测试三**

```
$ docker run -d  --gpus all--name pytorch  swr.cn-south-1.myhuaweicloud.com/myj-prod-local-cluster-repository/pytorch:cuda11.8
```
`docker exec -it pytorch  bash  执行 python3,调用cuda 计算显卡`

**测试四**
进pod 里面执行nvida-smi 是否有输出，有输出说明成功了


## 排错答疑
{{< admonition type=question >}}
1. `could not select device driver "" with capabilities: [[gpu]]`
- 通常是由于 Docker 没有正确识别到 GPU，或者 NVIDIA Docker 配置不正确。可能是 `nvidia-container-toolkit` 工具包没有安装，docker配置文件中的默认运行时没有添加nvidia配置

2. `Is there a difference between: nvidia-docker run and docker run --runtime=nvidia ?`  
- `docker run --runtime=nvidia is only available since nvidia-docker v2.`
{{< /admonition >}}



## 参考
1. [下载安装NVIDIA驱动程序](https://www.nvidia.cn/Download/index.aspx?lang=cn)
2. [kernel 内核源码包](https://rpmfind.net/linux/rpm2html/search.php?query=kernel-devel)
3. [Installation Guide Linux :: CUDA Toolkit Documentation](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html) 这里查看受支持的Linux版本
4. [NVIDIA Container Toolkit 介绍及安装 by Unbuntu ](https://blog.csdn.net/gs80140/article/details/144581141)
5. [欧拉openEuler 22.03 (LTS-SP1)安装NVIDIA Driver](https://www.cnblogs.com/boradviews/p/18567780)
6. [Installing the NVIDIA Container Toolkit 官网](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
7. [Openeuler v22.03 AI 大模型服务镜像使用指南](https://docs.openeuler.org/zh/docs/22.03_LTS_SP4/docs/AI/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E6%9C%8D%E5%8A%A1%E9%95%9C%E5%83%8F%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97.html)
8. [nvidia-container-toolkit github](https://github.com/NVIDIA/nvidia-container-toolkit)
9. [NVIDIA Container 运行时库 USTC 中科大官方镜像](https://mirrors.ustc.edu.cn/help/libnvidia-container.html)
10. [CUDA  和GPU 驱动型兼容性](https://docs.nvidia.com/deploy/cuda-compatibility/)

{{< admonition type=info >}}
本文档所提供的指引和参考主要基于特定实践设备的操作经验。由于不同设备在硬件配置、软件版本、使用场景等方面可能存在差异，因此，当您在使用其他设备时，所遇到的问题可能与此文档所述有所不同。尽管如此，大部分设备的安装方法和基本步骤仍然保持相似。
{{< /admonition >}}

