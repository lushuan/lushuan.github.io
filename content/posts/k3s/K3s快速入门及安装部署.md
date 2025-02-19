---
title: "K3s 快速入门及安装部署"
subtitle: ""
date: 2024-09-23T12:06:37+08:00
lastmod: 2024-12-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["K3s"]
tags: ["K3s"]
---
## 背景
为物联网和边缘计算构建的经过认证的Kubernetes发行版，是Rancher旗下的产品

Great for:

- Edge
- IoT
- CI
- Development
- ARM
- Embedding k8s
- Situations where a PhD in k8s clusterology is infeasible

## 发展历程
微型kubernetes发行版
1. CNCF认证的Kubernetes发行版
2. 50MB左右二进制包，500MB左右内存消耗
3. 单一进程包含Kubernetes master,Kubelet,和 containerd
4. 支持SQLite/Mysql/PostgreSQL/DQlite和etcd
5. 同时为x86 64,Arm64,和 Armv7 平台发布

变更
1. 多合一：将基础服务合并到了单一的进程中，如api-server,controller manager、scheduler和cloud-controller-manager合到了单一的进程中,简化了运维，降低了复杂度


## 基本架构
角色
- k3s server
  - api-server
  - controller-manager
  - scheduler
  - etcd(option)
  - containerd
  - k3s-agent
- k3s agent
  - k3s-agent
  - containerd|docker(optional)
  - 

![k3s-arch](/images/k3s/k3s-arch.png "K3s 架构")

技术亮点
- 单进程架构简化部署
- 移除各种非必须代码，减小资源占用
- TLS证书管理
- 内置Containerd
- 内置自运行rootfs
- 内置Helm Chart管理机制
- 内置L4/L7 LB支持

其它
- 默认使用containerd
- 默认命名空间
  - kube-system
  - kube-public
  - kube-node-release
- k3s 资源文件目录：`/var/lib/rancher/k3s`
## 安装部署
### server 部署
```shell
$ mkdir /root/k3s && cd /root/k3s
# 1. k3s镜像文件,指定安装版本为 v1.24.9
$ wget https://github.com/k3s-io/k3s/releases/download/v1.24.9-rc1%2Bk3s2/k3s-airgap-images-amd64.tar.gz
$ sudo mkdir -p /var/lib/rancher/k3s/agent/images/
$ sudo cp /root/k3s/k3s-airgap-images-amd64.tar /var/lib/rancher/k3s/agent/images/

# 2. k3s安装脚本
$ wget https://rancher-mirror.rancher.cn/k3s/k3s-install.sh

# 3. k3s二进制文件
$ wget https://github.com/k3s-io/k3s/releases/download/v1.24.9-rc1%2Bk3s2/k3s
$ sudo chmod a+x /root/k3s/k3s /root/k3s/k3s-install.sh
$ sudo cp /root/k3s/k3s /usr/local/bin/

# 4. 安装k3s
$ INSTALL_K3S_SKIP_DOWNLOAD=true /root/k3s/k3s-install.sh

# 5. 验证
$ crictl images
IMAGE                                        TAG                    IMAGE ID            SIZE
docker.io/rancher/klipper-helm               v0.7.4-build20221121   6f2af12f2834b       252MB
docker.io/rancher/klipper-lb                 v0.4.0                 3449ea2a2bfa7       9.21MB
docker.io/rancher/local-path-provisioner     v0.0.23                9621e18c33880       37.7MB
docker.io/rancher/mirrored-coredns-coredns   1.9.4                  a81c2ec4e946d       49.8MB
docker.io/rancher/mirrored-library-busybox   1.34.1                 827365c7baf13       5.09MB
docker.io/rancher/mirrored-library-traefik   2.9.4                  288889429becf       136MB
docker.io/rancher/mirrored-metrics-server    v0.6.2                 25561daa66605       70.2MB
docker.io/rancher/mirrored-pause             3.6                    6270bb605e12e       686kB

$ kubectl get pod -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   local-path-provisioner-687d6d7765-h8xvr   1/1     Running     0          2m25s
kube-system   coredns-7b5bbc6644-nbvnf                  1/1     Running     0          2m25s
kube-system   helm-install-traefik-crd-zfzqv            0/1     Completed   0          2m25s
kube-system   metrics-server-667586758d-5gvfd           1/1     Running     0          2m25s
kube-system   svclb-traefik-07d6e08f-gbvwd              2/2     Running     0          107s
kube-system   traefik-64b96ccbcd-tbcth                  1/1     Running     0          107s
kube-system   helm-install-traefik-tvcc8                0/1     Completed   2          2m25s

# server 节点获取k3s token
$ mynodetoken=`awk -F ':' '{print $4}'  /var/lib/rancher/k3s/server/token`
```
### agent 部署
```shell
# 1. 准备k3s 二进制
$ mkdir /root/k3s && cd /root/k3s
$ wget https://github.com/k3s-io/k3s/releases/download/v1.24.9-rc1%2Bk3s2/k3s
$ wget https://rancher-mirror.rancher.cn/k3s/k3s-install.sh
# 需保证各主机名称不一样进入 /etc/hostname 和 /etc/hosts 修改即可
$  INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.24.9-rc1+k3s2" K3S_URL=https://192.168.143.101:6443 K3S_TOKEN=7983aa3ee5b8a70afa61a79e2e6476d6 /root/k3s/k3s-install.sh
# 在线安装
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://192.168.143.101:6443 \
  K3S_TOKEN=7983aa3ee5b8a70afa61a79e2e6476d6 INSTALL_K3S_VERSION="v1.24.9-rc1+k3s2" \
  sh -

# 2. 查看状态
$ systemctl status k3s-agent
● k3s-agent.service - Lightweight Kubernetes
   Loaded: loaded (/etc/systemd/system/k3s-agent.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-09-23 17:05:24 CST; 8s ago
     Docs: https://k3s.io
  Process: 4125 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
  Process: 4121 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
  Process: 4116 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service 2>/dev/null (code=exited, status=0/SUCCESS)
 Main PID: 4131 (k3s-agent)
    Tasks: 24
   Memory: 300.4M
   CGroup: /system.slice/k3s-agent.service
           ├─4131 /usr/local/bin/k3s agent
           └─4159 containerd 
# 3. 切到server 节点
$ kubectl get node
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   25m   v1.24.9-rc1+k3s2
node1    Ready    <none>                 64s   v1.24.9-rc1+k3s2
```


## K3s 各名字空间介绍
在Kubernetes中，包括K3s，命名空间（Namespaces）提供了一种机制来划分集群资源。它们对于组织和隔离工作负载非常有用，特别是在多用户环境中。以下是K3s中常见的几个命名空间及其作用：

1. **`default`**：
   - 这是Kubernetes集群中的默认命名空间。如果你没有特别指定一个命名空间，那么你的Pod、服务等资源将会被创建在这个命名空间中。

2. **`kube-system`**：
   - 该命名空间包含了由Kubernetes系统自身创建的所有组件，如核心DNS服务（CoreDNS）、各种控制器和服务代理（kube-proxy）。此外，如果使用了某些网络插件或存储插件，相关的Pod也会出现在这个命名空间中。

3. **`kube-public`**：
   - 这个命名空间是一个特殊的命名空间，旨在供所有用户（包括那些未认证的用户）读取。它通常用于放置所有人都可以访问的信息，例如集群引导令牌（bootstrap tokens）。在标准配置下，此命名空间内只有一个ConfigMap对象，名为`cluster-info`，它包含了关于API服务器的信息。

4. **`kube-node-lease`**：
   - 这个命名空间是从Kubernetes 1.16版本开始引入的，用来存放节点租约（Node Lease）对象。这些对象允许节点定期更新其活跃状态，从而优化了节点健康检查的过程，使得集群能够更快地检测到不健康的节点。

5. **其他命名空间**：
   - 用户可以根据需要创建更多的命名空间来组织不同的应用或者团队的工作负载。这有助于资源管理和配额控制，并且可以在不同项目之间提供一定程度的隔离。

6. **特定于K3s的命名空间**：
   - 在一些情况下，K3s可能会引入额外的命名空间以支持特定的功能或集成。例如，当启用某些附加组件时，可能会自动创建相应的命名空间来托管这些组件。

  
### 其他名字空间
1. catle-system  
Rancher的"catle-system"命名空间通常用于存储Rancher控制平面组件的资源，如Webhook服务(rancher-webhookpod)和与安全相关的配置，比如TLS证书(cattle-webhook-tls)。这些组件是Rancher管理集群自动化部署和操作的核心部分。

当你提到洲除1oca1集群catt1e-system命名空间中的特定Pod和TLS密钥时，这意味着你可能正在清理测试环境或者准备进行维护操作。以


## 生产环境参考
- k3s 版本v1.22
- 数据库：etcd
- 容器运行时：containerd v1.5.9-k3s1

### 1. server
1. 服务状态
```shell
$ systemctl status k3s
● k3s.service - Lightweight Kubernetes
   Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2024-05-13 23:00:30 CST; 8 months 10 days ago
     Docs: https://k3s.io
 Main PID: 25364 (k3s-server)
    Tasks: 84
   Memory: 938.8M
   CGroup: /system.slice/k3s.service
           ├─17092 /var/lib/rancher/k3s/data/8307e9b398a0ee686ec38e18339d1464f75158a8b948b059b564246f4af3a0a6/bin/containerd-shim-runc-v2 -namespace k8s.io -id f11d857c503976e830b667a4...
           ├─25364 /usr/local/bin/k3s server
           ├─25379 containerd 

$ systemctl status etcd
● etcd.service - Etcd Server
   Loaded: loaded (/usr/lib/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2022-05-20 09:50:06 CST; 2 years 8 months ago
 Main PID: 63172 (etcd)
    Tasks: 11
   Memory: 413.0M
   CGroup: /system.slice/etcd.service
           └─63172 /opt/etcd/etcd --config-file=/data/etcd/2379/conf/etcd.conf
```
2. 版本
```shell
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.6+k3s1", GitCommit:"3228d9cb9a4727d48f60de4f1ab472f7c50df904", GitTreeState:"clean", BuildDate:"2022-01-25T01:27:44Z", GoVersion:"go1.16.10", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.6+k3s1", GitCommit:"3228d9cb9a4727d48f60de4f1ab472f7c50df904", GitTreeState:"clean", BuildDate:"2022-01-25T01:27:44Z", GoVersion:"go1.16.10", Compiler:"gc", Platform:"linux/amd64"}
$ kubectl -n kube-system get all
NAME                                 READY   STATUS    RESTARTS   AGE
pod/coredns-96cc4f57d-7k4gr          1/1     Running   0          2y247d
pod/metrics-server-ff9dbcb6c-vl8hl   1/1     Running   0          2y200d

NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                        AGE
service/kube-dns         ClusterIP   10.43.0.10     <none>        53/UDP,53/TCP,9153/TCP         2y248d
service/kubelet          ClusterIP   None           <none>        10250/TCP,10255/TCP,4194/TCP   2y247d
service/metrics-server   ClusterIP   10.43.99.234   <none>        443/TCP                        2y248d

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns          1/1     1            1           2y248d
deployment.apps/metrics-server   1/1     1            1           2y248d

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-96cc4f57d          1         1         1       2y248d
replicaset.apps/metrics-server-ff9dbcb6c   1         1         1       2y248d
```
3. 容器运行时
ctr 是 containerd 的客户端命令行工具
```shell
$ ctr --help
NAME:
   ctr - 
        __
  _____/ /______
 / ___/ __/ ___/
/ /__/ /_/ /
\___/\__/_/

containerd CLI


USAGE:
   ctr [global options] command [command options] [arguments...]

VERSION:
   v1.5.9-k3s1
$ ctr version
Client:
  Version:  v1.5.9-k3s1
  Revision: 
  Go version: go1.16.10

Server:
  Version:  v1.5.9-k3s1
  Revision: 
  UUID: 521e28e5-d9b4-4ae9-9dfd-48f742329485
``` 
4. 相关镜像
```shell
$ ctr images list -q|grep -v sha256|grep -v fluent
docker.io/rancher/mirrored-coredns-coredns:1.8.6
docker.io/rancher/mirrored-metrics-server:v0.5.2
docker.io/rancher/mirrored-pause:3.6
docker.io/rancher/pause:3.1
docker.io/rancher/rancher-agent:v2.6.6
```
systemd 启动配置
```shell
$ cat /etc/systemd/system/k3s.service
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
Wants=network-online.target
After=network-online.target

[Install]
WantedBy=multi-user.target

[Service]
Type=notify
EnvironmentFile=-/etc/default/%N
EnvironmentFile=-/etc/sysconfig/%N
EnvironmentFile=-/etc/systemd/system/k3s.service.env
KillMode=process
Delegate=yes
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s
ExecStartPre=/bin/sh -xc '! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service'
ExecStartPre=-/sbin/modprobe br_netfilter
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/k3s \
    server \
        '--disable' \
        'traefik' \
        '--disable' \
        'servicelb' \
        '--disable' \
        'local-storage' \
        '--datastore-endpoint=http://192.168.47.24:2379,http://192.168.47.25:2379,http://192.168.47.26:2379' \
        '--tls-san' \
        '192.168.47.251' \
        '--token' \
        'secret' \
        '--kubelet-arg=container-log-max-files=5' \
        '--kubelet-arg=container-log-max-size=100Mi' \
        '--kube-proxy-arg' \
        'proxy-mode=ipvs' \
        '--kubelet-arg' \
        'eviction-hard=nodefs.available<10%' \
```

这个 K3s 启动命令配置了多个参数来定制其行为，下面是对每个参数的解释：

- `server`：表示启动的是 K3s 的服务器（控制平面）组件。

- `--disable traefik`：禁用内置的 Traefik Ingress Controller。K3s 默认安装 Traefik 作为集群入口控制器，但如果你已经有其他入口控制器或不需要它，可以使用此选项禁用。

- `--disable servicelb`：禁用服务负载均衡器（Service Load Balancer）。在多节点集群中，K3s 可以为没有外部负载均衡器的服务创建一个简单的软件负载均衡器，但你可以选择禁用它。

- `--disable local-storage`：禁用本地存储功能。K3s 提供了一个简易的本地存储插件，如果不需要此功能，可以通过该参数禁用。

- `--datastore-endpoint=http://192.168.47.24:2379,http://192.168.47.25:2379,http://192.168.47.26:2379`：指定外部 etcd 集群的地址列表，用于替代默认的嵌入式数据库。这使得高可用性成为可能，并且可以将数据持久化存储在专门的 etcd 集群中。

- `--tls-san 192.168.47.251`：为 TLS 证书添加额外的主题备用名称（Subject Alternative Name）。这对于确保 API 服务器能够正确地验证客户端请求非常重要，特别是在使用内部 IP 或者自定义域名时。

- `--token secret`：设置用于加入新节点到集群的令牌。这个令牌是所有希望加入集群的节点都需要提供的共享密钥。

- `--kubelet-arg=container-log-max-files=5` 和 `--kubelet-arg=container-log-max-size=100Mi`：这些参数传递给 kubelet，用来限制容器日志文件的数量和大小。在这个例子中，设置了每个容器最多保留5个日志文件，每个文件的最大大小为100MB。

- `--kube-proxy-arg proxy-mode=ipvs`：将 kube-proxy 的代理模式设置为 IPVS 模式。IPVS 是一种更高效的网络负载均衡机制，适用于较大规模的集群。

- `--kubelet-arg eviction-hard=nodefs.available<10%`：传递给 kubelet 的参数，用来设置驱逐策略。这里配置的是当节点文件系统的可用空间低于10%时触发驱逐操作，以防止磁盘满导致的问题。

通过以上参数，你可以看到这个命令是用来配置一个高度定制化的 K3s 服务器实例，它与外部 etcd 集群集成，启用了特定的日志管理策略，并调整了某些组件的行为以适应特定的需求。

5. 镜像仓库地址
```shell
$ pwd
/etc/rancher/k3s
$ cat registries.yaml 
mirrors:
  hub.xxx.yyyy.com:
    endpoint:
      - "https://hub.xxx.yyyy.com"
configs:
  "https://hub.xxx.yyyy.com":
    auth:
      username: lushaun
      password: password
    tls:
      insecure_skip_verify: true
```
6. node 注册到server passwd
Agent 将使用节点集群 Secret 以及随机生成的节点密码注册到 Server，密码存储在 /etc/rancher/node/password 中。Server 会将各个节点的密码存储为 Kubernetes Secret，后续的任何尝试都必须使用相同的密码
```shell
$ pwd
/etc/rancher/node
$ cat password 
a04fffdf1effc0e03f593a3xxxxxxxxx
```
7. 查看k3s 当前配置
```
$ /usr/local/bin/k3s check-config

Verifying binaries in /var/lib/rancher/k3s/data/8307e9b398a0ee686ec38e18339d1464f75158a8b948b059b564246f4af3a0a6/bin:
- sha256sum: good
- links: good

System:
- /sbin iptables v1.4.21: older than v1.8
- swap: disabled
- routes: default CIDRs 10.42.0.0/16 or 10.43.0.0/16 already routed

Limits:
- /proc/sys/kernel/keys/root_maxkeys: 1000000

modprobe: FATAL: Module configs not found.
info: reading kernel config from /boot/config-3.10.0-1062.el7.x86_64 ...

Generally Necessary:
- cgroup hierarchy: cgroups V1 mounted, cpuset|memory controllers status: good
- CONFIG_NAMESPACES: enabled
- CONFIG_NET_NS: enabled
- CONFIG_PID_NS: enabled
- CONFIG_IPC_NS: enabled
- CONFIG_UTS_NS: enabled
- CONFIG_CGROUPS: enabled
- CONFIG_CGROUP_CPUACCT: enabled
- CONFIG_CGROUP_DEVICE: enabled
- CONFIG_CGROUP_FREEZER: enabled
- CONFIG_CGROUP_SCHED: enabled
- CONFIG_CPUSETS: enabled
- CONFIG_MEMCG: enabled
- CONFIG_KEYS: enabled
- CONFIG_VETH: enabled (as module)
- CONFIG_BRIDGE: enabled (as module)
- CONFIG_BRIDGE_NETFILTER: enabled (as module)
- CONFIG_IP_NF_FILTER: enabled (as module)
- CONFIG_IP_NF_TARGET_MASQUERADE: enabled (as module)
- CONFIG_NETFILTER_XT_MATCH_ADDRTYPE: enabled (as module)
- CONFIG_NETFILTER_XT_MATCH_CONNTRACK: enabled (as module)
- CONFIG_NETFILTER_XT_MATCH_IPVS: enabled (as module)
- CONFIG_IP_NF_NAT: enabled (as module)
- CONFIG_NF_NAT: enabled (as module)
- CONFIG_POSIX_MQUEUE: enabled
- CONFIG_DEVPTS_MULTIPLE_INSTANCES: enabled

Optional Features:
- CONFIG_USER_NS: enabled
  (RHEL7/CentOS7: User namespaces disabled; add 'user_namespace.enable=1' to boot command line) (fail)
- CONFIG_SECCOMP: enabled
- CONFIG_CGROUP_PIDS: enabled
- CONFIG_MEMCG_KMEM: enabled
- CONFIG_RESOURCE_COUNTERS: missing
- CONFIG_BLK_CGROUP: enabled
- CONFIG_BLK_DEV_THROTTLING: enabled
- CONFIG_CGROUP_PERF: enabled
- CONFIG_CGROUP_HUGETLB: enabled
- CONFIG_NET_CLS_CGROUP: enabled
- CONFIG_NETPRIO_CGROUP: enabled
- CONFIG_CFS_BANDWIDTH: enabled
- CONFIG_FAIR_GROUP_SCHED: enabled
- CONFIG_RT_GROUP_SCHED: enabled
- CONFIG_IP_NF_TARGET_REDIRECT: enabled (as module)
- CONFIG_IP_SET: enabled (as module)
- CONFIG_IP_VS: enabled (as module)
- CONFIG_IP_VS_NFCT: enabled
- CONFIG_IP_VS_PROTO_TCP: enabled
- CONFIG_IP_VS_PROTO_UDP: enabled
- CONFIG_IP_VS_RR: enabled (as module)
- CONFIG_EXT4_FS: enabled (as module)
- CONFIG_EXT4_FS_POSIX_ACL: enabled
- CONFIG_EXT4_FS_SECURITY: enabled
- Network Drivers:
  - "overlay":
    - CONFIG_VXLAN: enabled (as module)
      Optional (for encrypted networks):
      - CONFIG_CRYPTO: enabled
      - CONFIG_CRYPTO_AEAD: enabled
      - CONFIG_CRYPTO_GCM: enabled (as module)
      - CONFIG_CRYPTO_SEQIV: enabled
      - CONFIG_CRYPTO_GHASH: enabled (as module)
      - CONFIG_XFRM: enabled
      - CONFIG_XFRM_USER: enabled
      - CONFIG_XFRM_ALGO: enabled
      - CONFIG_INET_ESP: enabled (as module)
      - CONFIG_INET_XFRM_MODE_TRANSPORT: enabled (as module)
- Storage Drivers:
  - "overlay":
    - CONFIG_OVERLAY_FS: enabled (as module)

STATUS: 1 (fail)
```
### 2. agent
```shell
$ systemctl status k3s-agent
● k3s-agent.service - Lightweight Kubernetes
   Loaded: loaded (/etc/systemd/system/k3s-agent.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2024-05-13 23:03:57 CST; 8 months 10 days ago
     Docs: https://k3s.io
 Main PID: 68145 (k3s-agent)
    Tasks: 120
   Memory: 1.2G
   CGroup: /system.slice/k3s-agent.service
```
k3s-agent.service 配置文件信息
```shell
# cat /etc/systemd/system/k3s-agent.service
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
Wants=network-online.target
After=network-online.target

[Install]
WantedBy=multi-user.target

[Service]
Type=exec
EnvironmentFile=-/etc/default/%N
EnvironmentFile=-/etc/sysconfig/%N
EnvironmentFile=-/etc/systemd/system/k3s-agent.service.env
KillMode=process
Delegate=yes
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s
ExecStartPre=/bin/sh -xc '! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service'
ExecStartPre=-/sbin/modprobe br_netfilter
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/k3s \
    agent \
        '--kubelet-arg' \
        'container-log-max-files=5' \
        '--kubelet-arg' \
        'container-log-max-size=100Mi' \
```

## 用户案例
选择K3s的原因
1. k3s与Kubernetes使用习惯完全一致:上手无代价
2. 与其他同类产品比，无功能绑定
3. 兼容Arm设备
4. 通过导入Rancher进行统一纳管
5. 无厂商锁定，无需绑定硬件
6. 100% 开源，合作方式灵活
7. 出错重启就可以了，不像k8s 出错重启也不一定能解决，哈哈

## Q/A
1. 使用k8s 替换k3s 有什么问题？
如果只是普通的使用一些workload 是没有什么问题的，如果workload 中的一些应用使用了一些诸如自定义的cloud provider 是不适配的，一般应用于iot之类的场景

## 参考
1. [k3s官网](https://k3s.io/)
2. [Rancher Product](https://www.rancher.cn/products)
3. [K3s Document](https://docs.k3s.io/zh/architecture)
4. [Kubernetes Document](https://kubernetes.io/docs/home/)
5. [K3s Github](https://github.com/k3s-io/k3s/)
6. [从0到1基础入门 全面了解k3s](https://www.bilibili.com/video/BV1g7411G7By/?spm_id_from=333.337.search-card.all.click&vd_source=7e8c74e6ef5f4645e4247ef4120b7971)
7. [一文搞定全场景K3s离线安装](https://mp.weixin.qq.com/s/pfUM6tr2HDeFyJExFVAc4Q)
8. [k3s安装指定版本以及离线安装（docker)](https://blog.csdn.net/weixin_45056021/article/details/142848836)
9. [11-K3S 安装-仪表盘及卸载K3s](https://blog.csdn.net/weixin_40274679/article/details/130155847)