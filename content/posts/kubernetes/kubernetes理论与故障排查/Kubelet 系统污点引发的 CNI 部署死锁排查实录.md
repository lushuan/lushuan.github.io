---
title: "【Kubernetes】Kubelet 系统污点引发的 CNI 部署死锁排查实录"
subtitle: ""
date: 2026-06-02T12:06:37+08:00
lastmod: 2026-06-02T12:06:37+08:00
draft: false
toc:
enable: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes排障"]
---


# Kubelet 系统污点引发的 CNI 部署死锁排查实录

在修复好网络模式变更导致控制平面组件异常和证书过期的问题后，发现节点的状态还是 NotReady 的状态，本篇文档记录一下Node 节点为NotReady 的修复的全记录，为NotReady 的原因有很多，这里的出现该问题
的原因则很明显是 cni 网络插件没有部署，在部署好cni 网络插件后，再思索一下为什么k8s 调整了使用operator 部署。最后引申出这种部署方式的好处有哪些？

## 1. 灾难降临：故障现场与第一震撼现象 (The Incident)

### 1.1 核心异常现象

异常现象
```shell
$ kubectl get node
NAME           STATUS     ROLES    AGE   VERSION
k8s-master01   NotReady   <none>   84s   v1.28.11
```

### 1.2 环境上下文 (Environment Context)

- 基础设施：VMware 虚拟机
- 架构版本：Kubernetes v1.28/操作系统内核版本 5.15.0
- 诱发变更: 该k8s 集群是在2024年搭建的，当时用作自己的测试学习，当时还基于该环境整理一篇部署文档，[Ubuntu-server部署k8s1.28(containerd版本)集群
  ](https://blog.dazhongma.top/ubuntu-server%E9%83%A8%E7%BD%B2k8s1.28containerd%E7%89%88%E6%9C%AC%E9%9B%86%E7%BE%A4/),上一篇文档是记录解决切换网络模式解决控制平面组件异常的问题，也就是[网络模式变更为什么会洗残k8s集群](https://blog.dazhongma.top/%E7%BD%91%E7%BB%9C%E6%A8%A1%E5%BC%8F%E5%8F%98%E6%9B%B4%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BC%9A%E6%B4%97%E6%AE%8Bk8s%E9%9B%86%E7%BE%A4/),本篇记录了Node 节点NotReady 各种原因中的其中一个CNI 网络插件问题。

## 2. 顺藤摸瓜：多维度排查与见招拆招 (The Exploration)

### 排查思路：从现象到根因
节点变成 NotReady，本质上是 kubelet 向 apiserver 汇报心跳失败。可能的原因很多，但可以按照以下优先级逐一排查：

| 优先级 | 可能原因 | 典型表现 |
| --- | --- | --- |
| P0 | kubelet 进程死掉/崩溃 | 节点完全无响应 |
| P0 | 网络中断（节点与 apiserver） | kubelet 无法连接 apiserver |
| P1 | CNI 插件异常（Calico/Flannel） | 节点网络不通，Pod 互访失败 |
| P1 | Docker/containerd 服务异常 | 容器无法创建或启动 |
| P2 | 资源耗尽（磁盘、内存、PID） | kubelet 报配额不足错误 |
| P2 | 证书过期（kubelet client 证书） | kubelet 无法认证 apiserver |
| P3 | 内核参数被修改 | 网络、文件句柄等达到上限 |

当然这里已经确认了是CNI 插件的问题，就先按照解决CNI 插件问题的方式进行处理。在处理的过程中是如何遇到CNI部署死锁的问题，又是如何处理的？


### 2.1 部署 k8s集群部署网络插件 calico

常见的网络插件有多种，比如flannel、Calico和Cilium，本次实验使用Calico使用。

首先访问Calico帮助文档https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

#### master 节点
calico 部署使用的是operator 的方式，且不再和kube-system 共用一个namespace，另起了一个namesapce calico-system,引入了istio 相关组件服务，详情可以看下calico官网

配置插件需要的环境
```shell
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
```
下载客户端资源文件并修改pod 网段地址，因为我们的pod的网段和配置文件不一样

注意：部署operator 时需要连接外网加载该镜像`quay.io/tigera/operator:v1.32.5`,建议使用[dockerproxy](https://dockerproxy.net/)代理来处理

```shell
# 下载客户端资源文件
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml
# 修改pod的网段地址 podSubnet
sed -i 's/cidr: 192.168.0.0/cidr: 10.244.0.0/g' custom-resources.yaml
kubectl create -f custom-resources.yaml
```
查看集群 calico pod 创建过程，大约会耗时三分钟,部署成功后，所有node 节点为Ready 状态
```shell
watch kubectl get all -o wide -n calico-system
```
{{< admonition tip >}}
以上方法是在dockerhub 网络仓库正常访问情况下执行的，如果dockerhub 网络仓库访问异常可以使用离线部署的方式

2024.6 月份docker hub 被墙了后，就不能方便的下载 docker 镜像了，这里使用离线的方式进行部署。
{{< /admonition >}}

calico v3.27 相关镜像列表
```
docker.io/calico/cni:v3.27.2
docker.io/calico/kube-controllers:v3.27.2
docker.io/calico/node:v3.27.2
docker.io/calico/pod2daemon-flexvol:v3.27.2
docker.io/calico/typha:v3.27.2
# 下面三个镜像 release-v3.27.2 没有包含进来
docker.io/calico/csi:v3.27.2
docker.io/calico/apiserver:v3.27
docker.io/calico/node-driver-registrar:v3.27.2
```

手动镜像导入
```shell
sudo ctr -n k8s.io images ls|grep calico

sudo ctr -n k8s.io images import calico-cni.tar  
sudo ctr -n k8s.io images import calico-dikastes.tar  
sudo ctr -n k8s.io images import calico-flannel-migration-controller.tar  
sudo ctr -n k8s.io images import calico-kube-controllers.tar  
sudo ctr -n k8s.io images import calico-node.tar  
sudo ctr -n k8s.io images import calico-pod2daemon.tar  
sudo ctr -n k8s.io images import calico-typha.tar
# # 下面三个镜像 release-v3.27.2 没有包含进来,需要手动导入
sudo ctr -n k8s.io images import csi.tar
sudo ctr -n k8s.io images import apiserver.tar
sudo ctr -n k8s.io images import node-driver-registrar.tar
```

参考：[calico_3.27 quick install 官网](https://docs.tigera.io/calico/3.27/getting-started/kubernetes/quickstart)


## 顺藤摸瓜：多维度排查与见招拆招
上面的部署k8s cni calico 网络插件是一个常规流程，但是由于我之前部署的结构是单master + 单node 节点，cni 网络插件被我部署到了node 节点上，现在在master 节点上还需要移除污点调度
```shell
kubectl taint nodes k8s-master01 node.kubernetes.io/not-ready:NoSchedule-
```
此刻发现部署的过程中，创建了`tigera-operator`和`calico-system`
```shell
$kubectl get ns
NAME               STATUS   AGE
default            Active   28h
kube-node-lease    Active   28h
kube-public        Active   28h
kube-system        Active   28h
tigera-operator    Active   26h
```
此刻发现tiger-operator pod 异常重启
```shell

$ kubectl get node
NAME           STATUS     ROLES    AGE    VERSION
k8s-master01   NotReady   <none>   144m   v1.28.11
$ kubectl -n tigera-operator get pod
NAME                               READY   STATUS    RESTARTS      AGE
tigera-operator-7595bb75f4-g7rpl   1/1     Running   3 (32s ago)   53s
$ kubectl -n tigera-operator get pod
NAME                               READY   STATUS   RESTARTS      AGE
tigera-operator-7595bb75f4-g7rpl   0/1     Error    3 (36s ago)   57s
$ ctr -n k8s.io images |grep calico
$ kubectl -n tigera-operator describe pod tigera-operator-7595bb75f4-g7rpl
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  3m5s                  default-scheduler  Successfully assigned tigera-operator/tigera-operator-7595bb75f4-g7rpl to k8s-master01
  Normal   Pulled     76s (x5 over 3m5s)    kubelet            Container image "quay.io/tigera/operator:v1.32.5" already present on machine
  Normal   Created    76s (x5 over 3m5s)    kubelet            Created container tigera-operator
  Normal   Started    76s (x5 over 3m5s)    kubelet            Started container tigera-operator
  Warning  BackOff    48s (x10 over 2m58s)  kubelet            Back-off restarting failed container tigera-operator in pod tigera-operator-7595bb75f4-g7rpl_tigera-operator(d1d08a82-188e-435f-8644-7b7bf437fd77)
```

从 describe 的 Events 来看，镜像 `quay.io/tigera/operator:v1.32.5` 已经在本地了（already present on machine），容器也能成功 Started。但是它刚启动就挂了（Status: Error），而且在 57 秒内重启了 3 次，直接陷入了 CrashLoopBackOff。

既然容器能跑起来再挂掉，说明不是 K8s 调度的锅，而是 Operator 进程内部由于缺少某些初始化条件或者配置冲突崩了。

目前是已经部署了tiger-operator 但是pod 异常重启

### 核心排查命令查看 pod 更多信息
对于这种启动即崩溃（Exit Code 通常是 1 或 137/255）的 Operator，我们需要看它到底在临死前喊了什么。请执行以下命令抓取它的崩尸日志：
```shell
kubectl logs -n tigera-operator tigera-operator-7595bb75f4-g7rpl
```

### 💡 常见病因预测

根据之前咱们硬抬控制平面的背景，Operator 此时崩溃通常由于以下两个原因：

#### 1. 证书不信任或连接 APIServer 超时（概率最大）

因为你之前降级过 APIServer，而测试环境的本地 `~/.kube/config` 或者 Pod 内部挂载的 `serviceaccount` 凭证，可能与新拉起来的控制平面在通信或认证上存在 gap（比如证书时间戳或者 CA 没对上）。

* **日志特征：** 会看到 `request handshaking`、`x509: certificate signed by unknown authority` 或者 `timeout` 相关的报错。

#### 2. 残留的 CRD 导致元数据冲突

如果这个环境之前装过旧版的 Calico 或 Tigera，集群里可能残留了旧的 CRD（自定义资源类型）。新版的 Operator 启动时去校验或加载这些 CRD，发现字段不匹配，直接 panic 崩溃。

* **日志特征：** 会看到 `Failed to delete old CRD`、`unable to convert` 或者 `panic: schema` 等字样。

---

### 🛠️ 关于 `ctr` 命令的冷知识（为什么 grep 不到？）

你刚才执行 `ctr -n k8s.io images | grep calico` 没结果是完全正常的：

* 因为通过 Operator 部署时，第一阶段**只会拉取 Operator 自身的镜像**（即 `tigera/operator`），此时它还没来得及去拉真正的 `calico-node`、`calico-cni` 镜像呢！所以本地暂时确实没有带 `calico` 关键字的镜像。

### 继续查看日志信息
```shell
$ kubectl logs -n tigera-operator tigera-operator-7595bb75f4-g7rpl

2026/06/01 05:54:19 [INFO] Version: v1.32.5

2026/06/01 05:54:19 [INFO] Go Version: go1.21.6 X:boringcrypto

2026/06/01 05:54:19 [INFO] Go OS/Arch: linux/amd64

2026/06/01 05:54:22 [ERROR] Get "https://10.96.0.1:443/api?timeout=32s": dial tcp 10.96.0.1:443: connect: no route to host
```

大白于天下！报错非常清晰：
`dial tcp 10.96.0.1:443: connect: no route to host`

`10.96.0.1` 是 Kubernetes 内部默认的 **ClusterIP 服务（kubernetes.default）**。Tigera Operator 作为一个普通的 Pod，启动后的第一件事就是通过这个 Service IP 去找 APIServer 握手并同步状态。

但是，因为集群**现在还没有任何 CNI 网络插件**，宿主机上的 `kube-proxy` 和系统的路由表/iptables 还没有完全就绪或无法转发，导致 Pod 内部根本找不到去往 `10.96.0.1` 的路由，直接报 `no route to host`。

这就再次陷入了经典的“先有鸡还是先有蛋”的死锁：

> Operator 需要网络通了才能去创建 Calico；而网络要通，又必须等 Operator 起来去创建 Calico。

## 🛠️ 破局方案

既然通过 ClusterIP 走不通，我们需要**绕过内部 Service，让 Operator 直接通过宿主机的物理网络（APIServer 的物理 IP）进行通信**。

请按照以下两步修改 Operator 的部署配置：

### 第一步：修改 Operator 部署文件（改为 HostNetwork）

我们需要将 Tigera Operator 的 Deployment 改为使用**宿主机网络模式（hostNetwork: true）**。这样它就会直接用 `k8s-master01` 的物理网络，不再通过 `10.96.0.1` 绕路。

先将tigera-operator.yaml 进行备份，再进行如下修改，在 `spec.template.spec` 标签下，加入 `hostNetwork: true` 和 `dnsPolicy: ClusterFirstWithHostNet`：
```yaml
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: tigera-operator
    spec:
      hostNetwork: true                 # <-- 添加这一行，让它共享宿主机网络
      dnsPolicy: ClusterFirstWithHostNet # <-- 添加这一行，确保主机网络下能解析K8s域名
      containers:
      - name: tigera-operator
        image: quay.io/tigera/operator:v1.32.5
        # ... 其余保持不变 ...

```
保存并重新应用：
```bash
kubectl apply -f tigera-operator.yaml

```

### 第二步：检查环境变量（可选）

有些版本的 Tigera Operator 在配置中会显式通过环境变量传入 APIServer 的地址。如果在日志里发现它依然固执地去连 `10.96.0.1`，可以在刚才的 Deployment 容器环境变量（`env`）中，强行指定为你物理 Master 节点的 IP：

```yaml
        env:
        - name: KUBERNETES_SERVICE_HOST
          value: "192.168.1.11" # <-- 换成你 master01 的真实物理内网 IP
        - name: KUBERNETES_SERVICE_PORT
          value: "6443"

```

### 🎯 期待的效果

完成上述修改后，旧的 Pod 会被销毁，新的 Pod 会直接带着宿主机的网络权限拉起。此时它再去调 APIServer 就能一路畅通。

执行以下命令盯着它合体：

```bash
watch -n 2 "kubectl get pods -n tigera-operator && kubectl get nodes"

```

一旦 Operator 成功挺住不崩，它就会立刻在后台把真正的 `calico-node` 等组件一股脑生出来，节点很快就会刷成 **`Ready`**！

### 一波三折并非期待的效果
在49 行到 51 行这一段
```shell
envFrom:
- configMapRef:
    name: kubernetes-services-endpoint
    optional: true
```
Tigera Operator 默认会去读取一个叫 kubernetes-services-endpoint 的 ConfigMap。如果这个 ConfigMap 存在或者有默认残留配置，它里面记录的依然是内核内置的 KUBERNETES_SERVICE_HOST: 10.96.0.1。这就是为什么你开了 hostNetwork，它在启动时依然固执地去冲 10.96.0.1，结果因为没有 CNI 路由再次撞墙。

### 🛠️ 怎么改？

既然我们要大力出飞砖，直接在 `env` 标签下显式注入**物理控制平面 IP**，强行覆盖掉 ConfigMap 的脑残默认行为。

请直接使用 `kubectl edit` 或者修改你的 YAML 重新 apply，将 `spec.template.spec.containers[0].env` 改为如下配置：

```yaml
      containers:
      - command:
        - operator
        env:
        - name: KUBERNETES_SERVICE_HOST
          value: "192.168.1.11"               # <-- 强制指定为你 master01 的真实物理 IP
        - name: KUBERNETES_SERVICE_PORT
          value: "6443"                       # <-- 强制指定控制平面的安全端口
        - name: WATCH_NAMESPACE
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: OPERATOR_NAME
          value: tigera-operator
        - name: TIGERA_OPERATOR_INIT_IMAGE_VERSION
          value: v1.32.5
        # 下面的 envFrom 可以留着也可以删掉，因为上面的显式 env 优先级更高
        envFrom:
        - configMapRef:
            name: kubernetes-services-endpoint
            optional: true

```



### 🎯 修改后的观察

改完后，原本处于 `CrashLoopBackOff` 的 Pod 会瞬间被销毁重建。接着通过日志盯死它：

```bash
# 1. 看看新的 Pod 名字叫什么
kubectl get pod -n tigera-operator

# 2. 盯死它的启动日志
kubectl logs -f -n tigera-operator deployment/tigera-operator

```
这一次它拿到物理 IP 之后就能顺利完成三方握手，不再报 `no route to host` 了，Calico 的各个工作负载马上就会跟着被派发下来！

### 最终期待的效果
```shell
$ kubectl get pod -n tigera-operator
NAME                               READY   STATUS    RESTARTS   AGE
tigera-operator-66448b59f6-mmgtw   1/1     Running   0          18s

$ kubectl get ns
NAME              STATUS   AGE
calico-apiserver  Active   19s
calico-system     Active   19s
default           Active   154m
kube-node-lease   Active   154m
kube-public       Active   154m
kube-system       Active   154m
tigera-operator   Active   42m

$ kubectl -n calico-system get pod
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7f8d5cfff9-cfvts   1/1     Running   0          40s
calico-node-nv9jr                          0/1     Running   0          41s
calico-typha-5b5bb85ffb-fm9pm              1/1     Running   0          41s
csi-node-driver-xfmz8                      2/2     Running   0          41s

$ kubectl get pod -n tigera-operator
NAME                               READY   STATUS    RESTARTS   AGE
tigera-operator-66448b59f6-mmgtw   1/1     Running   0          60s

$ kubectl get node
NAME           STATUS   ROLES    AGE    VERSION
k8s-master01   Ready    <none>   155m   v1.28.11
```


## 📝 故障复盘报告：K8s 控制平面瘫痪与 CNI 复合死锁救援

###  1. 故障全景摘要

* **故障定义**：Kubernetes 控制平面静态 Pod（APIServer/etcd）版本/镜像不匹配导致集群脑死亡；恢复控制平面后，又因缺乏 CNI 网络插件触发 Kubelet 保护性污点，导致 Tigera Operator 陷入“网络不通无法部署 CNI，无 CNI 网络无法接通”的**先有鸡还是先有蛋**的绝对死锁。
* **影响面**：整个集群控制平面整体瘫痪，`kubectl` 彻底失效，节点处于 `NotReady` 状态。
* **最终结果**：通过底层运行时介入、静态 Pod 镜像降级、网络降维（HostNetwork）及物理端点强行注入，集群核心组件及 Calico 网络全面恢复，节点状态回归 **`Ready`**。

---

### 2. 故障演进与解决全历程（Step-by-Step）

#### 🚨 阶段一：控制平面“脑死亡”，集群失联

##### 1.1 异常现象

执行任意 `kubectl` 命令（如 `kubectl get nodes`）直接报连接超时或拒绝连接；控制平面的安全端口 `6443` 处于未监听状态。

```bash
root@k8s-master01:/etc/kubernetes# netstat -ntlp | grep 6443
# 输出为空，APIServer 未启动

```

##### 1.2 原因剖析

在对集群进行组件调整或升级时，`/etc/kubernetes/manifests/` 下的静态 Pod YAML 文件中指定的组件镜像版本（如 `v1.28.11`）与本地实际存在的镜像、或国内加速源的 Tag 未对齐，导致宿主机 `kubelet` 无法正常拉起控制平面容器。

##### 1.3 解决步骤（底层介入与降级）

由于 `kubectl` 失效，排查必须下沉到容器运行时（Containerd）：

1. **查看底层死活**：使用 `crictl ps -a` 发现核心组件容器疯狂退出，通过 `crictl logs` 确认为镜像/配置引发的崩溃。
2. **镜像降级与对齐**：修改 `/etc/kubernetes/manifests/kube-apiserver.yaml` 等文件，将镜像版本果断降级至本地稳妥的 `v1.28.0`（阿里云镜像源）。
3. **验证大脑复活**：
```bash
root@k8s-master01:/etc/kubernetes# netstat -ntlp | grep 6443
tcp6       0      0 :::6443                 :::* LISTEN      4467/kube-apiserver 

```


核心端口恢复监听，`curl -k https://192.168.1.11:6443/version` 成功拿到 `v1.28.0` 响应，控制平面救活。

---

#### 🧱 阶段二：节点“假死”，网络插件真空

##### 2.1 异常现象

控制平面虽然活了，但执行 `kubectl get node` 发现节点死死卡在 `NotReady`：

```bash
root@k8s-master01:/etc/kubernetes# kubectl get node
NAME           STATUS     ROLES    AGE   VERSION
k8s-master01   NotReady   <none>   84s   v1.28.11

```

##### 2.2 原因剖析

在整个容器运行时重置或故障期间，原本的 Calico 网络插件容器被全部清理。Kubelet 启动后，检测到集群内**没有任何可用的 CNI 网络插件**，无法为 Pod 分配 IP 建立网络平面。
为了自保，Kubelet 会自动在节点上打上系统污点：`node.kubernetes.io/not-ready:NoSchedule`。

##### 2.3 解决步骤（明确现状）

1. 检查本地是否有 CNI 容器运行：`crictl ps | grep calico` 输出完全为空。
2. 检查系统命名空间：`kubectl get pods -n kube-system` 中完全没有网络插件的身影，确认此时集群处于**零 CNI 插件真空状态**。

---

#### 🔄 阶段三：Tigera Operator 陷入“鸡蛋死锁”

##### 3.1 异常现象

尝试通过 Tigera Operator 重新部署 Calico 网络。部署后发现 Operator 的 Pod 疯狂报错崩溃（`CrashLoopBackOff`）：

```bash
root@k8s-master01 kuberntest-install>$ kubectl -n tigera-operator get pod
NAME                               READY   STATUS   RESTARTS      AGE
tigera-operator-7595bb75f4-g7rpl   0/1     Error    3 (36s ago)   57s

```

利用诊断利器 `kubectl logs -n tigera-operator <pod-name>` 抓取临终日志，暴露核心报错：

```text
[ERROR] Get "https://10.96.0.1:443/api?timeout=32s": dial tcp 10.96.0.1:443: connect: no route to host

```

##### 3.2 原因剖析（核心死锁逻辑）

1. Tigera Operator 作为一个普通 Pod 启动，默认根据内部配置或 ConfigMap，尝试通过 Kubernetes 内置的集群服务虚拟 IP（**ClusterIP: `10.96.0.1**`）去和 APIServer 握手通迅。
2. 然而，因为此时 Calico 还没起来，由 `kube-proxy` 维护的虚拟转发网络和底层网络平面**根本不存在**，宿主机内核路由表找不到去往 `10.96.0.1` 的路径，直接抛出 `no route to host`。
3. **死锁达成**：Operator 需要网络通了才能去创建 Calico；而网络要通，又必须等 Operator 成功运行去创建 Calico。

---

### 3. 终极破局：网络降维与强行指路

为了打破上述死锁，必须让 Operator **绕过尚未建立的集群内部网络，直接走宿主机的物理网络**向 APIServer 汇报。

### 🛠️ 解决步骤

1. **网络降维（共享宿主机网络）**：
   修改 Operator 部署配置，在 `spec.template.spec` 下加入 `hostNetwork: true` 和 `dnsPolicy: ClusterFirstWithHostNet`。使其直接使用 `k8s-master01` 宿主机的网络命名空间。
2. **强行指路（覆盖 ClusterIP）**：
   由于 Operator 会默认读取残留的 ConfigMap 重定向到 `10.96.0.1`，我们在 Deployment 的环境变量（`env`）中注入高优先级的物理端点配置，强行指定为 Master 的真实内网物理 IP：
```yaml
env:
- name: KUBERNETES_SERVICE_HOST
  value: "192.168.1.11"               # 强行指向物理 IP
- name: KUBERNETES_SERVICE_PORT
  value: "6443"                       # 强行指向物理端口

```


3. **合体验证**：
   重新配置后，新生的 Operator 容器秒级成功运行（`READY 1/1 Status: Running`），并立刻向下派发 Calico 核心组件。
4. **大功告成**：
   几秒钟内，`calico-node` 容器在本地拉起，CNI 网络平面成功建立。Kubelet 检测到网络就绪，自动撤销 `NotReady` 系统污点，节点瞬间回归 **`Ready`** 状态！
```bash
root@k8s-master01 kuberntest-install>$ kubectl get node
NAME           STATUS   ROLES    AGE    VERSION
k8s-master01   Ready    <none>   155m   v1.28.11

```



---

### 4. 核心经验沉淀（DevOps 避坑指南）

1. **不可盲目依赖 CNI Operator 的默认行为**：在曾经重置或故障修复的集群中，Operator 默认去读的 `kubernetes-services-endpoint` ConfigMap 极易引发 ClusterIP 路由撞墙。在初始化或抢修集群时，显式在 YAML 里注入物理 `KUBERNETES_SERVICE_HOST**` 是防止网络死锁的黄金标准。
2. **善用底层不失联工具链**：当控制平面断网时，`kubectl` 就是睁眼瞎。此时必须形成条件反射：看静态 Pod 状态用 `crictl ps -a` / `crictl logs`；看 Pod 临终遗言用 `kubectl logs -p**`。
3. **版本兼容定力**：Kubelet 二进制本身具备极强的向下兼容性（高版本 Kubelet 兼容低版本控制平面）。当遇到升级故障时，优先保证控制平面清单（Manifests）里的镜像能匹配并在本地稳妥运行，不盲目扩大变更面，才是化解危机的高级 DevOps 素养。