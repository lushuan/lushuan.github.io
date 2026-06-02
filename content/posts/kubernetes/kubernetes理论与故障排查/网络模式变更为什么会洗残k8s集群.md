---
title: "虚拟机 IP 没变，为什么桥接转换 NAT 模式却“洗”残了我的 K8s 集群？"
subtitle: ""
date: 2026-06-01T12:06:37+08:00
lastmod: 2026-06-01T12:06:37+08:00
draft: false
toc:
enable: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes排障"]
---

## 1. 灾难降临：故障现场现象
本篇记录了网络模式变更，导致 k8s 集群异常的根因分析及修复全过程

### 1.1 核心异常现象
集群大脑彻底瘫痪
```
root@k8s-master01:~# kubectl get node
E0601 10:39:42.322540    1861 memcache.go:265] couldn't get current server API group list: Get "https://192.168.1.11:6443/api?timeout=32s": dial tcp 192.168.1.11:6443: connect: connection refused
E0601 10:39:42.323432    1861 memcache.go:265] couldn't get current server API group list: Get "https://192.168.1.11:6443/api?timeout=32s": dial tcp 192.168.1.11:6443: connect: connection refused
E0601 10:39:42.324365    1861 memcache.go:265] couldn't get current server API group list: Get "https://192.168.1.11:6443/api?timeout=32s": dial tcp 192.168.1.11:6443: connect: connection refused
E0601 10:39:42.326999    1861 memcache.go:265] couldn't get current server API group list: Get "https://192.168.1.11:6443/api?timeout=32s": dial tcp 192.168.1.11:6443: connect: connection refused
E0601 10:39:42.327232    1861 memcache.go:265] couldn't get current server API group list: Get "https://192.168.1.11:6443/api?timeout=32s": dial tcp 192.168.1.11:6443: connect: connection refused
The connection to the server 192.168.1.11:6443 was refused - did you specify the right host or port?
root@k8s-master01:~# cat /etc/os-release 
PRETTY_NAME="Ubuntu 22.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.4 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
root@k8s-master01:~# uname -a
Linux k8s-master01 5.15.0-124-generic #134-Ubuntu SMP Fri Sep 27 20:20:17 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```
### 1.2 环境上下文
- 基础设施：VMware 虚拟机
- 架构版本：Kubernetes v1.28/操作系统内核版本 5.15.0
- 诱发变更: 该k8s 集群是在2024年搭建的，当时用作自己的测试学习，当时还基于该环境整理一篇部署文档，[Ubuntu-server部署k8s1.28(containerd版本)集群
  ](https://blog.dazhongma.top/ubuntu-server%E9%83%A8%E7%BD%B2k8s1.28containerd%E7%89%88%E6%9C%AC%E9%9B%86%E7%BE%A4/),到今天差不多正好两年了，由于当时在家里搭建的时候选择的是基于bridge 桥接网络模式， 现在将当时在家里部署的台式机搬到了公司，切换到了wifi 无线网卡模式，算是搞了一个环境迁移。本意是想使用 client-go 做一些功能验证，既然有了环境就不想再搭建了，vmware 启动的时候才发现，在bridge 模式下虚拟机和宿主机网络双向不通。但是虚拟机 ip 地址并没有变化，只是将 VMnet8 网段和虚拟机网段调整成了同一网段。


## 2. 顺藤摸瓜：多维度排查与见招拆招 (The Exploration)

目前已经确认了的是，ip使用桥接方式，二层网络被隔离，其实就是 WIFI 无线驱动不支持桥接。 如果说现在是 ip 变了的话，那没有什么好说的，etcd 等组件和ip 进行了深入绑定，而ip 没有变，apiserver 组件的6443 端口依然没有被监听，这就感觉有点蹊跷了。 并且 k8s-apiserver 服务彻底挂掉了，没有启动。

**彻底排查与修复步骤**  
由于这是单 Master + 单 node 的测试环境，按照“网络 -> 运行时 -> 静态 Pod -> Kubelet”的顺序自底向上修复。

### step 1 网络: 检查并确认本地容器状态
因为 APIServer 是以容器形式跑在 containerd 中的，我们要看一看到底是什么在阻止它启动。
使用 crictl（CRI 客户端）查看核心组件容器是否在报错：
```shell
# 查看所有的容器（包括停止的）
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a
```
- 寻找： k8s_kube-apiserver、k8s_etcd 的容器。
- 如果它们处于 Exited 状态，拿到它们的 Container ID，看日志：
```shell
crictl --runtime-endpoint unix:///run/containerd/containerd.sock logs <CONTAINER_ID>
```
通常在这里能看到 APIServer 是因为绑定本地 IP 192.168.1.11 失败，还是因为 etcd 连不上而自杀。

### step 2 检查防火墙与系统路由
VMware 的 NAT 模式非常依赖宿主机的虚拟网卡状态,防火墙我记得在部署的时候是关掉的，这里做一下验证
```shell
# 1. 测试本地回环与本地 IP 能否握手：
ping 192.168.1.11
curl -k https://192.168.1.11:6443
# 2. 关闭可能冲突的防火墙：
ufw disable
iptables -F
```

### step 3 重置 Kubelet 的强行拉起逻辑
当确保网络底座和配置目录还原后，强制重启运行时和 kubelet：
```shell
systemctl restart containerd
systemctl restart docker
# 此时静候1分钟，给 containerd 一点时间去通过静态manifests拉起 apiserver
sleep 30
systemctl restart kubelet
```

### step 4 终极杀招（若证书/路由受损，通过 kubeadm 局部修复）
执行上述步骤后，crictl ps 里依然看不到 apiserver,或者提示证书因为 IP/路由变化过期,这里利用 kubeadm init phase 重新刷一遍控制平面组件，在重新刷一遍控制平面组件之前先处理一下证书

#### 🛠️ 第一步：紧急备份原有证书和配置
在做任何破坏性变更前，先对现有的关键配置进行备份，防止意外。
```shell
mkdir -p /root/k8s_cert_bak/manifests/
cp -r /etc/kubernetes/pki/ /root/k8s_cert_bak/
cp /etc/kubernetes/*.conf /root/k8s_cert_bak/
cp /etc/kubernetes/manifests/*.yaml /root/k8s_cert_bak/manifests/
```

#### 🛠️ 第二步：更新 Control Plane 核心证书
使用 kubeadm 命令直接为所有组件重新生成为期 1 年的证书。
```shell
kubeadm certs renew all
```

执行完后可以再次运行 `kubeadm certs check-expiration` 验证，此时除了 CA 之外的核心证书应该都显示恢复正常。
#### 🛠️ 第三步：重新生成集群组件的 .conf 配置文件
因为 admin.conf、controller-manager.conf、scheduler.conf 内嵌了老的客户端证书，必须重新生成它们。
```shell
# 进入配置目录
cd /etc/kubernetes/

# 删除旧的配置文件（已备份，放心删）
rm -f admin.conf kubelet.conf controller-manager.conf scheduler.conf

# 重新生成这些配置文件
kubeadm init phase kubeconfig all
```

#### 🛠️ 第四步：修复 Kubelet 证书并恢复运行
这一步是破局的关键。我们需要生成临时的 bootstrap-kubelet.conf 让 kubelet 能够成功引导并生成新的节点证书。
```shell
# 生成 kubelet 的引导配置文件
kubeadm init phase kubeconfig kubelet

# 将新生成的系统管理员管理凭证更新到当前用户的默认配置中（供 kubectl 使用）
mkdir -p $HOME/.kube
cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 重启 containerd 和 kubelet
systemctl restart containerd
systemctl restart kubelet
```

#### 🛠️ 第五步：强制刷新静态 Pod 状态
由于之前 etcd 和 apiserver 的容器状态被缓存，我们需要像之前说的那样，手动移走再移回 Manifest 文件，强迫 kubelet 根据新证书重建它们。
```shell
# 1. 移走静态 Pod 清单，让运行中的僵尸容器彻底退出
mkdir /tmp/k8s-yaml
mv /etc/kubernetes/manifests/*.yaml /tmp/k8s-yaml

# 2. 等待 15 秒左右，通过 crictl 确认之前的 etcd/apiserver 容器完全消失
sleep 15
crictl ps

# 3. 重新把清单移回来，触发重启验证
mv /tmp/k8s-yaml/*.yaml /etc/kubernetes/manifests/
```

#### 🛠️ 第六步：⏳ 状态验证
等待大约 1-2 分钟，让控制平面组件拉起。然后执行以下命令验证集群健康度：
```shell
# 1. 检查 containerd 里的核心容器是否变为 Running
crictl ps | grep -E "etcd|apiserver|scheduler|controller"

# 2. 检查节点及核心组件状态
kubectl get nodes
kubectl get pod -n kube-system
```
按照这个顺序走一遍，应该就能把这个大罢工的控制平面重新拉起来
### step 5 重置生产集群前的最后一步
检查 containerd 里的核心容器是否变为 Running，发现并没有，这个时候检查一下kubelet 的日志
```shell
journalctl -u kubelet -n 50 --no-pager
```
重点看： 是否依然在报 `bootstrap-kubelet.conf` 找不到，还是报类似于 `unauthorized、x509: certificate signed by unknown authority`，或者是找不到 Manifests 路径。

通过日志查看有一句报错
```shell
failed to "StartContainer" for "kube-apiserver" with ImagePullBackOff: "Back-off pulling image \"registry.k8s.io/kube-apiserver:v1.28.11\""
```
kubelet 已经成功加载了 Manifests，并且正在努力拉起 apiserver 和 etcd。但是，它尝试去官方镜像源 `registry.k8s.io` 重新拉取镜像，由于国内网络环境问题，拉取失败陷入了 ImagePullBackOff，这里可以使用[dockerproxy](https://dockerproxy.net/)代理获取相应的镜像，在拉取之前先检查一下本地是否有存留的镜像

#### 🛠️ 第一步：检查本地现有的 K8s 镜像
我们要确认本地实际存在的镜像叫什么名字（可能是国内代理源，也可能是老版本的 tag）。
```shell
crictl img | grep -E "apiserver|etcd"
```
会看到类似于： registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.28.11 或者类似的其他前缀

#### 🛠️ 第二步：修改 Manifest 里的镜像地址（一劳永逸）
由于 kubeadm certs renew 或 kubeconfig 刷新时，重置了静态 Pod 清单里的默认镜像中心（变成了原生的 registry.k8s.io），我们需要把它们改成能跑的地址。

方法 A：如果本地有阿里云或其他国内源的镜像
直接用 sed 批量替换 /etc/kubernetes/manifests/ 下的 4 个 YAML 文件。例如，如果本地镜像全是 registry.cn-hangzhou.aliyuncs.com/google_containers：
```shell
sed -i 's|registry.k8s.io|registry.cn-hangzhou.aliyuncs.com/google_containers|g' /etc/kubernetes/manifests/*.yaml
```
方法 B：直接修改成本地已有的精确 Tag
如果你刚才 crictl img 看到的镜像是带其他私有前缀，你可以手动编辑四个文件，或者给本地镜像打个 Tag 欺骗 kubelet：
```shell
# 举个例子：如果本地有阿里云镜像，直接 tag 成官方镜像名字
crictl tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.28.11 registry.k8s.io/kube-apiserver:v1.28.11
crictl tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.12-0 registry.k8s.io/etcd:3.5.12-0
crictl tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.28.11 registry.k8s.io/kube-controller-manager:v1.28.11
crictl tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.28.11 registry.k8s.io/kube-scheduler:v1.28.11
```

#### 🛠️ 第三步：触发重启验证
修改好镜像地址或者打完 Tag 后，再次刷新 kubelet 状态：
```shell
# 移走再移回，强制刷新容器生命周期（按之前定的 SOP 规范）
mv /etc/kubernetes/manifests/*.yaml /tmp/k8s-yaml
sleep 5
mv /tmp/k8s-yaml/*.yaml /etc/kubernetes/manifests/

# 观察容器是否开始 Running
sleep 10
# 请稍微等上 1-2 分钟，让容器内部完全初始化完毕，然后执行：
# 1. 看看现在的控制平面组件是不是还在 RUNNING，有没有变成 EXITED
crictl ps | grep -E "etcd|apiserver|scheduler|controller"

# 2. 检查 6443 端口起来了没
ss -ntlp | grep 6443

# 3. 尝试本地请求一下（如果是证书/认证问题，至少会返回 403 或者是 401，而不是 connection refused）
curl -k https://192.168.1.11:6443/version
```
执行结果
```shell
$ netstat -ntlp | grep 6443
tcp6       0      0 :::6443                 :::*                    LISTEN      4467/kube-apiserver 
$ kubectl get node
NAME           STATUS     ROLES    AGE   VERSION
k8s-master01   NotReady   <none>   84s   v1.28.11
$ curl -k https://192.168.1.11:6443/version
{
  "major": "1",
  "minor": "28",
  "gitVersion": "v1.28.0",
  "gitCommit": "855e7c48de7388eb330da0f8d9d2394ee818fb8d",
  "gitTreeState": "clean",
  "buildDate": "2023-08-15T10:15:54Z",
  "goVersion": "go1.20.7",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
可以发现节点是NotReady 的状态，到此刻算是解决了控制平面的问题，NotReady 也不要慌，这个是cni 网络插件的问题，后面再另外整理一篇文章进行回答。

#### 💡 几个最可能导致“死活不建立容器”的暗坑

根据以往的填坑经验，证书更新后静态 Pod 不出容器，通常是以下三个原因之一：

1. **`kubelet` 没有重启成功：** 有时候系统配置冲突，`systemctl restart kubelet` 可能卡住或者直接挂了。请用 `systemctl status kubelet` 确认它现在是 `active (running)` 状态。
2. **Kubelet 内部配错了静态 Pod 路径：**
   检查 `/var/lib/kubelet/config.yaml` 中，`staticPodPath` 这一项是否指向 `/etc/kubernetes/manifests`。
3. **系统时间严重漂移：**
   如果这台 Master 节点的系统时间跟证书生成的 UTC 时间有很大误差（比如主板电池没电或时区错乱），`kubelet` 会认为新证书“还未生效”（Not Before 限制），从而拒绝加载。可以用 `date` 命令肉眼看一眼时间对不对。

   
## 总结：为什么网络模式改变会导致 K8s 彻底瘫痪？
核心矛盾：网卡 MAC 地址与网关变了
在家里使用 桥接模式 时，虚拟机是直接向家里的路由器（物理网关）申请 IP 的，走的是物理网卡的混杂模式。
到了公司改用 NAT 模式 后，虚拟机的网络不再由公司无线路由器决定，而是由 VMware 软件虚拟出来的 NAT 网关和虚拟网卡（通常是 VMnet8） 来接管。

即使手动把虚拟机的 IP 依然固定为 192.168.1.11，其上游的默认网关、DNS 以及宿主机和虚拟机通信的虚拟网卡 MAC 物理路由全都变了。


###  问题核心原因及思考
由于切换了虚拟机网络模式，导致 kubelet 读取的集群配置文件失效 / 证书不匹配(过期)，kubelet 启动失败 → apiserver 起不来 → 集群挂了。 

因为是2年前搭建的集群，证书确实是失效了。这次的修复目的就不打算直接使用 reset 重置进行处理，虽然这样简单粗暴，但是还是想将其当成交付到现场的生产环境进行原地复活，虽然也是动了一场大手术，也方面后续遇到类似的问题进行借鉴。

还有就是涉及到删除配置文件和清单的操作，一定要是先备份再删除。

