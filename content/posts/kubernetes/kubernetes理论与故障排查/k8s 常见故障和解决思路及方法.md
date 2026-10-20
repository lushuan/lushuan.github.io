---
title: "k8s 常见故障和解决思路及方法"
subtitle: ""
date: 2024-01-13T12:06:37+08:00
lastmod: 2024-04-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes排障"]
slug: "k8s-common-failures-troubleshooting"
featuredImage: "/images/kubernetes/cover.jpg"
summary: "系统整理 Kubernetes 常见故障的排查思路与解决方法，重点覆盖 Pod 各种异常状态（Pending、ContainerCreating、ImagePullBackOff、CrashLoopBackOff、Error、Terminating/Unknown 等）的成因分析与诊断命令，并介绍存活/就绪探针等健康检查机制。"
---

Kubernetes 集群故障多集中在资源配置、应用代码、网络和存储四个方面，其中 Pod 状态异常是最常见的问题。本文系统整理各类 Pod 异常状态（Pending、ImagePullBackOff、CrashLoopBackOff、Terminating 等）的成因与排查思路，并介绍通用的诊断命令和健康检查探针机制。

<!--more-->

## 一、K8s 之连接异常——【集群故障】
Pod 问题思考思路

Kubernetes 集群中 Pod 一般哪些方面容易出现问题呢？
1. Kubernetes 资源配置错误：例如在部署 Deployment 和 Statefulset 时，资源清单书写有问题，导致 Pod 无法正常创建
2. 代码问题：应用程序代码在容器启动后失败，那就需要通过排查代码查找错误
3. 网络问题：网络插件部署有问题，导致 Pod 启动之后无法相互通信
4. 存储问题：Pod 挂载存储，但是需要共享的存储连接不上导致 Pod 启动异常

### Pod 状态异常排查问题集
1. Running
2. Pending
3. Waiting
4. ContainerCreating
5. ImagePullBackOff
6. CrashLoopBackOff
7. Error
8. Terminating
9. Unknown
10. Evicted

如何查看 pod 状态：
```
kubectl get pods <pod name> #查看pod运行状态
kubectl describe pods <pod name> #查看pod详细信息
```

Pod 状态异常排查思路

一般来说，无论 Pod 处于什么异常状态，都可以执行以下命令来查看 Pod 的状态：

1. 查看 Pod 的配置是否正确
```
kubectl get pod <pod-name> -o yaml
```
2. 查看 Pod 的详细信息 
```
kubectl describe pod <pod-name>
```
3. 查看 Pod 里对应容器的日志
```
kubectl logs <pod-name> [-c <container-name>]
```

### 部分节点无法启动 Pod：ContainerCreating 状态
诊断方法

1. 查看 pod
```
kubectl get pods -o wide | grep tomcat
NAME READY STATUS RESTARTS AGE IP NODE
tomcat 0/1 ContainerCreating 0 106s 10.244.1.4 k8s-node1
```
2. 查看 pod 详情信息
```
kubectl describe pods tomcat
```
3. 显示
```
error: code = Unknown desc = failed to set up sandbox container
"3f25ce8c6599a8b7824f8f9ba9cf2c078d9496dd840fb2305bf415929879f107" network for pod “tomcat": NetworkPlugin
cni failed to set up pod “tomcat" network: failed to set bridge addr: "cni0" already has an IP address different from
10.244.1.4/24
```
总结：pod 异常需要通过 describe 查看具体报错信息，根据具体报错信息再进行处理，以上是遇到同类问题的一种排查思路，问题各种各样，排查问题的思路却是可以有迹可循的

### Pod 状态异常排查问题集-pending 状态排查思路
Pending 是挂起状态：表示创建的 Pod 找不到可以运行它的物理节点，不能调度到相应的节点上运行，那么这种情况如何去排查呢？可以从两个层面分析问题：

**1. 物理节点层面分析**
- 查看节点资源使用情况：如 free -m 查看内存、top 查看 CPU 使用率、df –h 查看磁盘使用情况，这样就可以快
速定位节点资源情况，判断是不是节点有问题
- 查看节点污点：kubectl describe nodes master1（控制节点名字），如果节点定义了污点，那么 Pod 不能容
忍污点，就会导致调度失败，如下：
- Taints: node-role.kubernetes.io/master:NoSchedule
- 这个看到的是控制节点的污点，表示不允许 Pod 调度（如果 Pod 定义容忍度，则可以实现调度）

**1. Pod 本身分析 **
1. 在定义 pod 时，如果指定的 nodeName 是不存在的，那也会调度失败
2. 在定义 Pod 时，如果定义的资源请求比较大，导致物理节点资源不够也是不会调度的

### Pod 状态异常排查问题集- ImagePullBackOff 状态
问题分析
- 拉取镜像时间长导致超时
- 配置的镜像有错误：如镜像名字不存在等
- 配置的镜像无法访问：如网络限制无法访问 hub 上的镜像
- 配置的私有镜像仓库参数错误：如 imagePullSecret 没有配置或者配置错误
- dockerfile 打包的镜像不可用

### Pod 状态异常排查问题集- CrashLoopBackOff 状态
K8s 中的 pod 正常运行，但是里面的容器可能退出或者多次重启或者准备删除，导致出现这个状态，这个状态具有偶发性，可能上一秒还是 running 状态，但是突然就变成 CrashLoopBackOff 状态了。

### Pod 状态异常排查问题集- Error 状态
这种状态主要是 Pod 启动过程会出现：

原因分析：  
1）依赖的 ConfigMap、Secret、PV、StorageClass 等不存在；
2）请求的资源超过了管理员设置的限制，比如超过了 limits 等；
3）无法操作集群内的资源，比如开启 RBAC 后，需要为 ServiceAccount 配置权限。

排查思路：
```
# 1. describe #查看pod详细信息
kubectl describe pods <pod>
# 2. logs #查看pod日志
kubectl logs <pod> -c <container>
# 3. journalctl #查看kubelet日志
journalctl -f -u kubelet
# 4. tail –f #查看主机日志
tailf /var/log/messages
```
### Pod 状态异常排查问题集- Terminating 或 Unknown 状态
原因：Terminated 是因为容器里的应用挂了。

从 K8s v1.5 开始，Kubernetes 不会因为 Node 失联而删除其上正在运行的 Pod，而是将其标记为 Terminating 或 Unknown 状态。想要删除这些状态的 Pod 有 3 种方法：
1. 手动删除 node:
手动删除 kubectl delete nodes node 节点名字
2. 恢复正常可选择性删除： 
若 node 恢复正常，kubelet 会重新跟 kube-apiserver 通信
确认这些 Pod 的期待状态，进而再决定删除或者继续运行这些 Pod，如果删除可以通过 kubectl delete pods pod-name --grace-period=0 –force 强制删除
3. 使用参数重建容器  
Pod 行为异常，Pod 没有按预期的行为执行，比如没有运行 podSpec 里面设置的命令行参数，这一般是 podSpec YAML 文件内容有误，可以尝试使用 validate 参数重建容器，
比如（kubectl delete pod mypod 和 kubectl create -- validate -f mypod.yaml）；也可以查看创建后的 podSpec 是否是对的，比如（kubectl get pod mypod -o yaml）

Unknown 是 pod 状态不能通过 kubelet 汇报给 apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。

解决思路：
1. 查看 kubectl 状态
2. 查看节点状态
3. 查看 apiserver 是否异常
4. 查看 kubelet 日志
### Pod 健康检查：存活性探测&就绪性探测
**为什么需要探针？**

如果没有探针，K8s 无法知道应用是否还活着，只要 pod 还在运行，K8s 则认为容器是健康的。但实际上，pod 虽然运行了，但里面容器中的应用可能还没提供服务，那就需要做健康检查，健康检查就需要探针，常见的探针有如下几种：
1. httpGet
对容器的 ip 地址（指定的端口和路径）执行 HTTP get 请求
如果探测器收到响应，并且响应码是 2xx，则认为探测成功。如果服务器没有响应或者返回错误响
应码则说明探测失败，容器将重启。
2. tcpSocket  
探针与容器指定端口建立 TCP 连接，如果连接建立则探测成功，否则探测失败容器重启。
3. exec  
在容器内执行任意命令，并检查命令退出状态码，如果状态码为 0，则探测成功，否则探测失败
容器重启

LivenessProbe 和 ReadinessProbe 区别：  
- LivenessProbe 决定是否重启容器
- ReadinessProbe 主要来确定容器是否已经就绪：
只有当 Pod 里的容器部署的应用都处于就绪状态，也就是 pod 的 Ready 为 true（1/1）时，才会将请求转发给容器。

### CoreDNS 或者 kube-dns 经常重启和报错
问题描述：  
在 K8s 中部署服务，服务之间无法通过 DNS 解析，CoreDNS 一直处于 CrashLoopBackOff 状态
查看日志，报错如下：
```
plugin/loop: Loop (127.0.0.1:44222 -> :53) detected for zone
```
解决思路：  
CoreDNS 主要和 resolv.conf 打交道，会读取宿主机的 `/etc/resolv.conf` 中的 nameserver 内容，查询
resolv.conf 文件，如果里面存在本地回环如 127.0.0.1 或者 127.0.0.53 那么就容易造成死循环

解决步骤：  
修改 resolv.conf 文件：
将 nameserver 临时修改为 114.114.114.114 之后：`kubectl edit deployment coredns -n kube-system`
将 replicates 改为 0，从而停止已经启动的 CoreDNS pod 再将 replicates 改为 2，这样会触发 CoreDNS 重新读取系统配置，此时服务的状态为 Running

### kubectl 执行异常：无权限问题定位
描述
```
$ kubectl get node
> The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
这是因为安装 K8s 之后，默认是没有权限操作 K8s 资源的，可用如下方法解决
```
mkdir -p $HOME/.kube
sudo cp -ifr /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 二、K8s 之通信异常——【网络故障】
### 网络故障原因
Kubernetes 中的网络是非常重要的，整个 K8s 集群的通信调度都是通过网络插件实现的，Kubernetes 之上的网络通信主要有如下几种：

1. 容器间的通信：同一个 pod 内的多个容器间的通信，lo
2. pod 通信：从一个 pod Ip 到另一个 pod Ip
3. Pod 与 service 通信： pod ip 到 cluster ip
4. 外部通信： Service 与集群外部客户端的通信

通信需要依赖网络插件，如 Calico、Flannel、canal 等，通信过程最常见的故障如下

1. Pod 自身网络故障
2. Pod 间通信故障
3. Pod 和物理节点通信故障

## 三、K8s 之内部异常——【节点故障】
### 节点异常 1: 集群节点 NotReady
一般是该节点上的 kubelet 挂掉了，查看 kubelet 状态
```
systemctl status kubelet
```
如果挂掉需要重启一下 kubelet

扩展：为什么要重启 kubelet？
- 监视分配给该 Node 节点的 pods
- 挂载 pod 所需要的 volumes
- 下载 pod 的 secret
- 通过 docker/rkt 来运行 pod 中的容器
- 周期的执行 pod 中为容器定义的 liveness 探针
- 上报 pod 的状态给系统的其他组件
- 上报 Node 的状态

### 节点异常 2: pod 调度失败故障排查
在 K8s 集群中，pod 是最小单元，需要调度到具体的物理节点上运行，那么为了更好排查问题，了解 pod 的调度流程是重中之重，接下来介绍下 pod 工作和调度流程：

创建 pod 过程：

1、用户创建 pod 的信息通过 API Server 存储到 etcd 中，etcd 记录 pod 的元信息并将结果返回 API Server  
2、API Server 告知调度器请求资源调度分配，调度器通过计算，将优先级高的 node 与 pod 绑定并告知 API Server  
3、API Server 将此信息写入 etcd，得到 etcd 回复后调用 kubelet 创建 pod  
4、kubelet 使用 docker run 创建 pod 内的容器，得到反馈信息后将容器信息告知 API Server  
5、API Server 将收到的信息写入 etcd 并得到回馈  
6、此时使用 kubectl get pod 就可以查看到信息了  

调度方式：

1、当我们发送请求创建 Pod，调度器就会把 pod 调度到合适的物理节点，大致分为以下过程：  
Scheduler 根据预选策略和优选函数，选出合适的 node 节点调度  
2、可以通过定义 nodeName 和 nodeSelector 进行调度  
3、可以通过节点亲和性进行调度  

Pod 在调度的时候会找到一个合适的节点，所以如果节点资源不足；pod 定义了指定的节点，但是指定的节点出现故障，或者 pod 定义了亲和性，但是节点没有满足的条件，都会导致调度失败。

排查思路
1. describe pod 查看是否节点加了污点，pod 无法容忍，当然还要根据报错信息进行定位
2. 查看目标节点 kubelet 服务的状态

pod 调度失败内置污点

当某些条件为真时，节点控制器会自动为节点添加污点：

- node.kubernetes.io/ NoSchedule：如果一个 pod 没有声明容忍这个 Taint，则系统不会把该 Pod 调度到有这个 Taint 的 node 上
- node.kubernetes.io/NoExecute：定义 pod 的驱逐行为，以应对节点故障。
- node.kubernetes.io/not-ready：节点尚未准备好。这对应于 NodeConditionReady 为 False。
- node.kubernetes.io/unreachable：无法从节点控制器访问节点。这对应于 NodeConditionReady 为 Unknown。
- node.kubernetes.io/out-of-disk：节点磁盘不足。
- node.kubernetes.io/memory-pressure：节点有内存压力。
- node.kubernetes.io/disk-pressure：节点有磁盘压力。
- node.kubernetes.io/network-unavailable：节点的网络不可用。
- node.kubernetes.io/unschedulable：节点不可调度。

### 节点资源不足：Pod 超过节点资源限制
Pod 超过节点资源限制分两种情况：
情况一：Pod 数量太多超过物理节点限制，较少见
情况二：Pod 资源超过物理节点资源限制，相对频繁一些，requests 资源申请过多

### K8s 之内部异常【节点故障】 排查思路总结
节点状态查询：kubectl get nodes  
常见异常现象：1、节点状态 NotReady 2、调度到该节点的 pod 显示 Unknown、Pending 等  
常见故障：kubelet 进程异常、未安装 CNI 网络插件、Docker 异常、磁盘空间不足、内存不足、CPU 不足  

排查：
- `kubectl describe nodes 查看node节点信息`
- `kubectl describe pods <pod 名字> 查看pod节点信息`
- `systemctl status 查看kubelet状态`
- `Journalctl查看系统日志`
- `free查看内存`
- `top查看cpu`

## 四、K8s 之应用故障——【应用故障】

### 故障排查：Service 访问异常
Service 是什么？为什么容易出现问题？

在 Kubernetes 中，Pod 是有生命周期的，如果 Pod 重启 IP 很有可能会发生变化。如果我们的服务都是将 Pod 的 IP 地址写死，Pod 的挂掉或者重启，和刚才重启的 pod 相关联的其他服务将会找不到它所关联的 Pod，为了解决这个问题，在 Kubernetes 中定义了 service 资源对象，Service 定义了一个服务访问的入口，客户端通过这个入口即可访问服务背后的应用集群实例，Service 是一组 Pod 的逻辑集合，这一组 Pod 能够被 Service 访问到，通常是通过 Label Selector 实现的。

![service](/images/kubernetes/service-labelselecotr.png "service label selector")

情况一： service 和 pod labels 没有匹配
情况二：service 和 pod 正确关联，kube-proxy 异常

#### kube-proxy 导致 Service 访问异常
kube-proxy 是什么？

service 为一组相同的 pod 提供了统一的入口，起到负载均衡的效果。service 具有这样的功能，正是 kube-proxy 的功劳，kube-proxy 是一个代理，安装在每一个 K8s 节点，当我们暴露一个 service 的时候，kube-proxy 会在 iptables 中追加一些规则，为我们实现路由与负载均衡的功能。

背景：查看 Service、pod、endpoint 是否正常关联
```
$kubectl describe svc myapp-v1-service
Name: myapp-v1-service
Namespace: default
Selector: app=myapp,version=v1
Type: NodePort
IP: 10.98.1.137
Port: <unset> 80/TCP
TargetPort: 80/TCP
NodePort: <unset> 30080/TCP
Endpoints: 10.244.1.233:80,10.244.1.234:80
```
查看 iptables 是否正常创建规则
```
iptables -t nat -L|grep 10.98.1.137
KUBE-MARK-MASQ tcp -- !10.244.0.0/16 10.98.1.137 /* default/myapp-v1-service cluster IP */ tcp dpt:http
KUBE-SVC-J2PKJ4NHOU6XHVIG tcp -- anywhere 10.98.1.137 /* default/myapp-v1-service cluster IP */ tcp
dpt:http
```
通过上面可以看到之前创建的 service，会通过 kube-proxy 在 iptables 中生成一个规则，来实现流量路由。

通过上面可以看到，有一系列目标为 KUBE-SVC-xxx 链的规则，每条规则都会匹配某个目标 ip 与端口。也就是说访问某个
ip:port 的请求会由 KUBE-SVC-xxx 链来处理。这个目标 IP 其实就是 service ip。
```
$ iptables -t nat -L|grep KUBE-SVC-J2PKJ4NHOU6XHVIG
KUBE-SVC-J2PKJ4NHOU6XHVIG tcp -- anywhere anywhere /* default/myapp-v1-service */ tcp dpt:30080
KUBE-SVC-J2PKJ4NHOU6XHVIG tcp -- anywhere 10.98.1.137 /* default/myapp-v1-service cluster IP */ tcp dpt:http
```

```
$ iptables -t nat -L|grep KUBE-MARK-MASQ | grep 10.244.1.233
KUBE-MARK-MASQ all -- 10.244.1.233 anywhere /* default/myapp-v1-service */
$ iptables -t nat -L|grep KUBE-MARK-MASQ | grep 10.244.1.234
KUBE-MARK-MASQ all -- 10.244.1.234 anywhere /* default/myapp-v1-service */
```
通过上面分析可以知道，kube-proxy 在 service 请求中起到流量路由转发的作用：
kube-proxy 会为我们暴露的 service 追加 iptables 规则，具体流程如下：

1. 所有进出请求都会经过 KUBE-SERVICES 链
2. KUBE-SERVICES 链有我们发布的每一个服务，每个服务的链会对应到 DNAT 到 service 的 endpoint
3. KUBE-SERVICES 链最后的是 KUBE-NODEPORTS 链，会匹配请求本主机的一些端口，这就是我们通过 NODEPORT 类型发布服务能在集群外部访问的原因。

经过上面的分析，service 和 pod 正确关联，pod 也处于运行状态，那请求 service 还是访问不到 pod，90% 就是 kube-proxy 引起的。查看 kube-proxy 是否在机器上正常运行。

kube-proxy 如果在节点上运行，下一步，确认它有没有出现其他异常，比如连接主节点失败。要做到这一点，必须查看日志：如 /var/log/messages或者/var/log/kube-proxy.log（二进制安装的时候可以指定的），也可以使用 journalctl 访问日志、kube-proxy 如果是在 K8s 中以 pod 形式运行的，可以通过 kubectl logs <pod name> 查看日志。根据日志报错信息再进行进一步的排查

### 故障排查：Pod 删除失败
故障描述：在 K8s 中，可能会产生很多垃圾 pod，也就是有些 pod 虽然是 running 状态，可是其所调度到的 K8s node 节点已经从 K8s 集群删除了，但是 pod 还是在这个 node 上，没有被驱逐，针对这种情况就需要把 pod 删除。

删除 pod 的几种方法：
1. 删除 pod 所在的 namespace
2. 如果 pod 是通过控制器管理的，可以删除控制器资源
3. 如果 pod 是自主式管理，直接删除 pod 资源就可以了

但是我们常常遇见这样一种情况，我们在删除 pod 资源的时候，pod 会一直 terminate 状态，很长时间无法删除

解决方案：
可以通过如下方法强制删除：
加参数 `--force --grace-period=0，grace-period` 表示过渡存活期，默认
30s，在删除 POD 之前允许 POD 慢慢终止其上的容器进程，从而优雅退出，0 表示立即终止 POD
`kubectl delete pod <pod名字> -n --force --grace-period=0`
