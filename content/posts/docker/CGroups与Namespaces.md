---
title: '拆解Docker黑盒：用原生Linux命令手写一个"容器"'
subtitle: ""
date: 2023-05-13T16:25:58+08:00
lastmod: 2024-06-26T16:25:58+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Docker"]
tags: ["Docker"]
---

# 

## 引言：容器到底是什么？

当我们敲下 `docker run nginx` 的那一刻，一个"容器"就出现了。它看起来像虚拟机——有独立的进程树、独立的主机名、甚至独立的网络。但它比虚拟机轻量得多，启动只需几百毫秒。

容器不是魔法，也不是某种全新的技术。**容器的本质，就是 Linux 内核提供的两项基础能力的组合：**

- **Namespaces（命名空间）**：负责 **"隔离"**——让进程看到独立的世界（独立的进程号、主机名、网络栈、文件系统视图）。
- **CGroups（控制组）**：负责 **"限制"**——给进程划定资源边界（最多用多少内存、多少 CPU）。

Docker、containerd、runc……这些上层工具做的事情，归根结底就是帮你调用一系列 Linux 系统调用，把 Namespaces 和 CGroups 组装起来。

**本文将不依赖任何容器工具**，仅使用 `unshare`、`cgcreate`、`cgset` 等原生命令，一步步手动还原这个过程。读完之后，你会发现 `docker run` 的底层没有任何黑魔法。

> **环境说明**：本文基于 **CGroups v1**（CentOS 7 / Ubuntu 18.04~20.04 默认版本）。如果你的系统是纯 CGroups v2（如较新的 Fedora），部分路径和命令会有差异，文末附有简要说明。

---

## 一、先建立全局认知

在动手之前，先用一张概念图把全局拎清楚：

```
┌─────────────────────────────────────────────────────────┐
│                    宿主机 Linux 内核                      │
│                                                         │
│   ┌─────────────────────┐   ┌────────────────────────┐  │
│   │    Namespaces       │   │      CGroups           │  │
│   │   （隔离 · 看到什么） │   │   （限制 · 能用多少）   │  │
│   │                     │   │                        │  │
│   │  · PID  → 进程号    │   │  · memory → 内存上限   │  │
│   │  · UTS  → 主机名    │   │  · cpu    → CPU 配额   │  │
│   │  · NET  → 网络栈    │   │  · pids   → 进程数上限  │  │
│   │  · MNT  → 文件系统   │   │  · blkio  → 磁盘 IO   │  │
│   │  · IPC  → 共享内存   │   │                        │  │
│   │  · USER → 用户映射   │   │                        │  │
│   └─────────────────────┘   └────────────────────────┘  │
│                                                         │
│              两者组合 = 一个"容器"进程                      │
└─────────────────────────────────────────────────────────┘
```

接下来，我们先玩 **Namespaces**，再玩 **CGroups**，最后把它们粘在一起。

---

## 二、Namespaces：隔离的世界

### 2.1 从一个问题出发

在宿主机上执行：

```bash
ps aux | head -5
```

你会看到 PID 1 是 `systemd`（或 `init`），后面跟着成百上千个进程。现在想象一下：**如果一个进程只能看到自己，看不到宿主机的其他进程，它是不是就像运行在一台独立的"机器"上？**

这就是 **PID Namespace** 要做的事。

### 2.2 PID Namespace：独立的进程编号空间

Linux 提供了 `unshare` 命令，可以在不创建新进程树的前提下，让当前 shell 进入新的命名空间。

```bash
# 进入新的 PID 命名空间
# --pid       : 创建新的 PID Namespace
# --fork      : unshare 后 fork 一个子进程（PID NS 要求第一个进程 PID=1）
# --mount-proc: 重新挂载 /proc，使其反映新 NS 的进程视图（关键！）
# bash        : 在新环境中启动一个 shell
unshare --pid --fork --mount-proc bash
```

> ⚠️ **为什么必须加 `--mount-proc`？**
> 如果不加，新 shell 中的 `/proc` 仍然指向宿主机的进程信息，`ps` 命令依然会列出所有宿主机进程。`--mount-proc` 会在新 Namespace 中重新挂载一个干净的 `/proc`，让 `ps` 只能看到本 Namespace 内的进程。

进入新 shell 后，验证一下：

```bash
# 在新 Namespace 中查看进程
ps aux
```

**对比观察：**

| 观察项 | 宿主机 | 新 PID Namespace |
|:---|:---|:---|
| `ps aux` 输出 | 数百个进程，PID 1 是 systemd | 仅 2~3 个进程，PID 1 是 bash |
| 自身 shell 的 PID | 如 12345 | **1** |

```bash
# 在新 Namespace 中确认自己的 PID
echo $$
# 输出: 1  ← 你就是这个世界的"init"

# 再开一个后台进程，它的 PID 是 2
sleep 3600 &
echo $!
# 输出: 2
```

**退出新 Namespace：**

```bash
exit
```

> 📌 **小结**：PID Namespace 并没有创建新的内核，它只是给进程换了一套"编号规则"。新 Namespace 里的 PID 1 在宿主机上其实有一个完全不同的 PID（比如 29831），只是它自己不知道。

### 2.3 UTS Namespace：独立的主机名

```bash
# 进入新的 UTS Namespace
unshare --uts bash
```

```bash
# 修改主机名（不影响宿主机）
hostname my-container
hostname
# 输出: my-container
```

回到宿主机验证：

```bash
exit
hostname
# 输出: 仍然是原来的主机名
```

### 2.4 Mount Namespace：独立的文件系统视图

Mount Namespace 是最特殊的一个——它隔离的不是某个简单的标识符，而是**整个文件系统的挂载点视图**。

```bash
# 进入新的 Mount Namespace
unshare --mount bash
```

```bash
# 在新 NS 中创建一个 tmpfs 挂载
mkdir /tmp/my_isolated_fs
mount -t tmpfs tmpfs /tmp/my_isolated_fs

# 查看挂载点
mount | grep my_isolated_fs
# 输出: tmpfs on /tmp/my_isolated_fs type tmpfs ...
```

回到宿主机：

```bash
exit
mount | grep my_isolated_fs
# 无输出 ← 宿主机完全不知道这个挂载的存在
```

> 📌 这就是为什么 Docker 容器能拥有独立的 `/`。容器运行时通过 `pivot_root` 系统调用将进程的根目录切换到容器镜像的 rootfs，配合 Mount Namespace，进程就"以为"自己运行在一套独立的文件系统上。

### 2.5 Network Namespace：独立的网络栈

```bash
# 进入新的 Network Namespace
unshare --net bash
```

```bash
# 查看网络接口
ip addr
# 只有 lo，且状态为 DOWN，没有 eth0
```

```bash
exit
ip addr
# 宿主机一切正常
```

> 真实的容器网络远比这复杂（需要 veth pair、bridge、iptables 等），这里只演示隔离效果。

### 2.6 组合多个 Namespace

实际容器中，以上 Namespace 是**同时生效**的。`unshare` 支持一次指定多个：

```bash
unshare \
  --pid \
  --uts \
  --mount \
  --net \
  --fork \
  --mount-proc \
  bash
```

进入后依次验证：

```bash
echo $$            # PID = 1
hostname my-box    # 可以随意改主机名
ip addr            # 只有 lo
mount | head -5    # 独立的挂载视图
```

**此刻，你已经拥有了一个"看起来像独立机器"的进程环境。但它还没有资源限制——它可以吃光宿主机的内存和 CPU。这就轮到 CGroups 出场了。**

---

## 三、CGroups：资源的牢笼

### 3.1 CGroups 是什么？

CGroups（Control Groups）是 Linux 内核提供的**资源限制、优先级分配和统计机制**。它把一组进程归入一个"组"，然后对整个组施加资源上限。

先确认你的系统使用的是 CGroups v1：

```bash
# 如果以下路径存在，说明是 v1
ls /sys/fs/cgroup/
# 应看到: memory/  cpu/  pids/  blkio/  devices/  ...（每个子系统一个目录）

# 如果看到的是 cgroup.controllers 等文件，则是 v2
cat /sys/fs/cgroup/cgroup.controllers 2>/dev/null
```

> ⚠️ **以下所有 CGroups 操作均基于 v1。** v2 采用统一层级树，路径和写入方式完全不同，文末有简要对照。

### 3.2 准备一个"吃资源"的进程

为了演示限制效果，我们先制造一个内存消耗大户：

```bash
# 启动一个持续申请内存的后台进程（每次申请 1MB，共 500 次 ≈ 500MB）
bash -c 'arr=(); for i in $(seq 1 500); do arr+=("$(head -c 1M /dev/zero | base64)"); sleep 0.1; done' &

# 记下它的 PID
EAT_PID=$!
echo "吃内存进程 PID: $EAT_PID"
```

### 3.3 限制内存：200MB 上限

```bash
# 安装 libcgroup-tools（如果没有 cgcreate 命令）
# CentOS: yum install -y libcgroup-tools
# Ubuntu: apt install -y cgroup-tools

# 创建一个名为 mycontainer 的 memory 控制组
cgcreate -g memory:/mycontainer

# 设置内存上限为 200MB
cgset -r memory.limit_in_bytes=209715200 mycontainer
# 等价于: echo 209715200 > /sys/fs/cgroup/memory/mycontainer/memory.limit_in_bytes

# 查看当前限制是否生效
cgget -r memory.limit_in_bytes mycontainer
# 输出: memory.limit_in_bytes: 209715200
```

### 3.4 把进程"关进"CGroup

```bash
# 将吃内存的进程加入该控制组
echo $EAT_PID > /sys/fs/cgroup/memory/mycontainer/tasks

# 验证：查看该控制组中有哪些进程
cat /sys/fs/cgroup/memory/mycontainer/tasks
# 应输出 $EAT_PID
```

### 3.5 观察 OOM：精准击杀

由于进程不断申请内存，而 CGroup 上限只有 200MB，很快内核的 **OOM Killer** 就会介入：

```bash
# 等待几秒后检查
dmesg | grep -i "killed process" | tail -3
```

你会看到类似输出：

```
Memory cgroup out of memory: Killed process 29845 (bash) total-vm:...
```

**关键观察：**

- 被杀掉的**只有** CGroup 内的进程。
- 宿主机上的其他进程（包括你当前的终端）**完全不受影响**。

```bash
# 确认进程已被杀死
ps -p $EAT_PID
# 输出: 无此进程
```

> ⚠️ **安全提示**：不要将 PID 1（systemd）或关键系统服务写入 CGroup 再设一个极小的内存上限，否则可能导致系统关键进程被 OOM Kill。

### 3.6 限制 CPU

```bash
# 创建 CPU 控制组
cgcreate -g cpu:/mycontainer

# 限制为 0.5 个 CPU 核心
# cpu.cfs_period_us = 100000 (100ms 为一个周期，默认值)
# cpu.cfs_quota_us  = 50000  (每 100ms 最多用 50ms)
cgset -r cpu.cfs_period_us=100000 mycontainer
cgset -r cpu.cfs_quota_us=50000 mycontainer

# 启动一个 CPU 死循环并加入控制组
bash -c 'while true; do :; done' &
CPU_PID=$!
echo $CPU_PID > /sys/fs/cgroup/cpu/mycontainer/tasks

# 在另一个终端观察 CPU 占用（应为 ~50%）
top -p $CPU_PID
```

### 3.7 清理 CGroup

```bash
# 先杀掉残留进程
kill $CPU_PID 2>/dev/null

# 删除控制组
cgdelete -r memory:/mycontainer
cgdelete -r cpu:/mycontainer

# 验证已删除
ls /sys/fs/cgroup/memory/ | grep mycontainer
# 无输出
```

> ⚠️ **清理顺序**：必须先 `kill` 掉 CGroup 内的所有进程，再执行 `cgdelete`。否则删除会失败（`Device or resource busy`）。

---

## 四、粘合：用脚本"手搓"一个容器

前面我们分别体验了 Namespaces 和 CGroups。现在把它们组合起来，写一个最小化的"容器启动脚本"：

```bash
#!/bin/bash
# mini-container.sh —— 一个极简"容器"启动脚本

CONTAINER_NAME="mycontainer"
MEM_LIMIT="209715200"   # 200MB
CPU_QUOTA="50000"       # 0.5 核

echo "[1/4] 创建 CGroups..."
cgcreate -g memory:/$CONTAINER_NAME
cgcreate -g cpu:/$CONTAINER_NAME
cgset -r memory.limit_in_bytes=$MEM_LIMIT $CONTAINER_NAME
cgset -r cpu.cfs_quota_us=$CPU_QUOTA $CONTAINER_NAME
cgset -r cpu.cfs_period_us=100000 $CONTAINER_NAME

echo "[2/4] 进入新的 Namespaces 并启动容器进程..."
unshare --pid --uts --mount --net --fork --mount-proc bash -c '
  hostname '$CONTAINER_NAME'
  echo "  容器主机名: $(hostname)"
  echo "  容器 PID 1: $$"
  echo "  容器网络:"
  ip addr
  echo "  容器已就绪，按 Ctrl+C 退出"
  sleep infinity
' &
CONTAINER_PID=$!

echo "[3/4] 将容器进程加入 CGroups..."
sleep 0.5   # 等待 unshare 完成 fork
echo $CONTAINER_PID > /sys/fs/cgroup/memory/$CONTAINER_NAME/tasks
echo $CONTAINER_PID > /sys/fs/cgroup/cpu/$CONTAINER_NAME/tasks

echo "[4/4] 容器 $CONTAINER_NAME (PID: $CONTAINER_PID) 已启动！"
echo ""
echo "验证 CGroup 限制:"
echo "  内存上限: $(cat /sys/fs/cgroup/memory/$CONTAINER_NAME/memory.limit_in_bytes) bytes"
echo "  CPU 配额: $(cat /sys/fs/cgroup/cpu/$CONTAINER_NAME/cpu.cfs_quota_us) / $(cat /sys/fs/cgroup/cpu/$CONTAINER_NAME/cpu.cfs_period_us)"
```

运行它：

```bash
chmod +x mini-container.sh
sudo ./mini-container.sh
```

输出示例：

```
[1/4] 创建 CGroups...
[2/4] 进入新的 Namespaces 并启动容器进程...
  容器主机名: mycontainer
  容器 PID 1: 1
  容器网络:
1: lo: <LOOPBACK> mtu 65536 ...
[3/4] 将容器进程加入 CGroups...
[4/4] 容器 mycontainer (PID: 31207) 已启动！

验证 CGroup 限制:
  内存上限: 209715200 bytes
  CPU 配额: 50000 / 100000
```

**清理脚本：**

```bash
#!/bin/bash
# cleanup.sh
CONTAINER_NAME="mycontainer"

echo "停止容器进程..."
kill $(cat /sys/fs/cgroup/memory/$CONTAINER_NAME/tasks) 2>/dev/null
sleep 1

echo "删除 CGroups..."
cgdelete -r memory:/$CONTAINER_NAME 2>/dev/null
cgdelete -r cpu:/$CONTAINER_NAME 2>/dev/null

echo "清理完成。"
```

---

## 五、回归 Docker：从手动到自动

现在，让我们把刚才的每一步和 `docker run` 对应起来：

| 我们手动做的事 | Docker / runc 对应的事 |
|:---|:---|
| `unshare --pid --uts --mount --net` | `clone()` 系统调用 + `CLONE_NEWPID \| CLONE_NEWUTS \| ...` |
| `--mount-proc` + `pivot_root` | 挂载容器镜像的 rootfs（overlayfs），切换根目录 |
| `cgcreate` + `cgset` + 写入 `tasks` | 通过 cgroup 文件系统设置资源限制 |
| `hostname mycontainer` | 写入 `/proc/1/uts/hostname` |
| 我们的 `mini-container.sh` | **runc**（OCI 运行时） |
| 如果加上镜像拉取、网络管理、持久化存储… | **containerd / Docker Engine** |

你会发现，Docker 并没有发明任何新的内核机制。**它的全部工作，就是把 Namespaces 和 CGroups 这两个 Linux 原语，按照 OCI 规范自动化地组装起来。**

理解了这一点，你就理解了：
- 为什么容器比虚拟机轻（没有 Guest OS，共享宿主内核）；
- 为什么容器逃逸本质上是 Namespace 或 CGroup 的突破；
- 为什么 Kubernetes 的 `resources.limits` 最终就是写 CGroup 文件。

---

## 六、CGroups v1 与 v2 速查对照

如果你的系统是 CGroups v2，主要差异如下：

| 对比项 | v1 | v2 |
|:---|:---|:---|
| 层级结构 | 每个子系统独立层级（`/sys/fs/cgroup/memory/`、`/sys/fs/cgroup/cpu/`） | **统一层级**（`/sys/fs/cgroup/` 下只有一棵树） |
| 限制内存 | `memory.limit_in_bytes` | `memory.max` |
| 限制 CPU | `cpu.cfs_quota_us` + `cpu.cfs_period_us` | `cpu.max`（格式：`quota period`） |
| 加入进程 | 写入 `tasks` | 写入 `cgroup.procs` |
| 确认方式 | `ls /sys/fs/cgroup/` 有多个子系统目录 | `cat /sys/fs/cgroup/cgroup.controllers` 有输出 |

> 在 CGroups v2 系统上，Docker / containerd 会自动适配，无需手动干预。但如果你想手动实验，需要按上表调整路径和文件名。

---

## 七、总结

```
docker run nginx
    │
    ├── clone(CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS | CLONE_NEWNET | ...)
    │       → Namespaces：隔离进程视图
    │
    ├── pivot_root → overlayfs
    │       → Mount NS：隔离文件系统
    │
    ├── cgroup.procs ← $PID
    │   echo 200M > memory.max
    │   echo "50000 100000" > cpu.max
    │       → CGroups：限制资源用量
    │
    └── execve("/docker-entrypoint.sh")
            → 容器进程启动
```

容器不是黑盒。拆开它，里面只有 Linux 内核用了二十多年的老手艺：**Namespace 给你一间独立的房间，CGroup 给你划好房间的面积上限。** Docker 做的事情，就是帮你把门和墙自动砌好。

希望下次你敲下 `docker run` 时，脑海里浮现的不再是一个魔法指令，而是 `clone`、`pivot_root`、`cgroup.procs` 这些朴素而精妙的系统调用。

---

*本文所有命令均可在 CentOS 7/8、Ubuntu 18.04/20.04（CGroups v1）上直接复现。生产环境请勿随意操作 CGroups，以免误杀关键进程。*

## 参考
1. https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/
2. https://www.infoq.cn/article/docker-resource-management-cgroups/