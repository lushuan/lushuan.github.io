---
title: "k8s 常见故障和解决思路及方法"
subtitle: ""
date: 2024-01-13T12:06:37+08:00
lastmod: 2024-04-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes排障"]
---

## K8S 之连接异常——【集群故障】
Pod问题思考思路

Kubernetes集群中Pod一般哪些方面容易出现问题呢？
1. Kubernetes资源配置错误：例如在部署Deployment和Statefulset时，资源清单书写有问题，导致Pod无法正常创建
2. 代码问题：应用程序代码在容器启动后失败，那就需要通过排查代码查找错误
3. 网络问题：网络插件部署有问题，导致Pod启动之后无法相互通信
4. 存储问题：Pod挂载存储，但是需要共享的存储连接不上导致Pod启动异常

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

如何查看pod状态：
```
kubectl get pods <pod name> #查看pod运行状态
kubectl describe pods <pod name> #查看pod详细信息
```

Pod状态异常排查思路

一般来说，无论 Pod 处于什么异常状态，都可以执行以下命令来查看 Pod 的状态：

1. 查看Pod的配置是否正确
```
kubectl get pod <pod-name> -o yaml
```
2. 查看 Pod 的详细信息 
```
kubectl describe pod <pod-name>
```
3. 查看Pod里对应容器的日志
```
kubectl logs <pod-name> [-c <container-name>]
```

### 部分节点无法启动Pod：ContainerCreating状态
诊断方法

1. 查看pod
```
kubectl get pods -o wide | grep tomcat
NAME READY STATUS RESTARTS AGE IP NODE
tomcat 0/1 ContainerCreating 0 106s 10.244.1.4 k8s-node1
```
2. 查看pod 详情信息
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
总结：pod 异常需要通过 describe 查看具体报错信息，根据具体报错信息再进行处理，以上是遇到同类问题的一种排查思路，问题各种各样，排查问题的思路确是可以有迹可循的

### Pod状态异常排查问题集-pending状态排查思路
Pending是挂起状态：表示创建的Pod找不到可以运行它的物理节点，不能调度到相应的节点上运行，那么这种情况如何去排查呢？可以从两个层面分析问题：

**1. 物理节点层面分析**
- 查看节点资源使用情况：如free -m查看内存、top查看CPU使用率、df –h查看磁盘使用情况，这样就可以快
速定位节点资源情况，判断是不是节点有问题
- 查看节点污点：kubectl describe nodes master1（控制节点名字），如果节点定义了污点，那么Pod不能容
忍污点，就会导致调度失败，如下：
- Taints: node-role.kubernetes.io/master:NoSchedule
- 这个看到的是控制节点的污点，表示不允许Pod调度（如果Pod定义容忍度，则可以实现调度）

**1. Pod 本身分析 **
1. 在定义pod时，如果指定了nodeName是不存在的，那也会调度失败
2. 在定义Pod时，如果定义的资源请求比较大，导致物理节点资源不够也是不会调度的

### Pod状态异常排查问题集- ImagePullBackOff状态
问题分析
- 拉取镜像时间长导致超时
- 配置的镜像有错误：如镜像名字不存在等
- 配置的镜像无法访问：如网络限制无法访问hub上的镜像
- 配置的私有镜像仓库参数错误：如imagePullSecret没有配置或者配置错误
- dockerfile打包的镜像不可用

### Pod状态异常排查问题集- CrashLoopBackOff状态
K8s中的pod正常运行，但是里面的容器可能退出或者多次重启或者准备删除，导致出现这个状态，这个状态具有偶发性，可能上一秒还是running状态，但是突然就变成CrashLoopBackOff状态了。

### Pod状态异常排查问题集- Error状态
这种状态主要是Pod启动过程会出现：

原因分析：  
1）依赖的ConfigMap、Secret、PV、StorageClass等不存在；
2）请求的资源超过了管理员设置的限制，比如超过了limits等；
3）无法操作集群内的资源，比如开启RBAC后，需要为ServiceAccount配置权限。

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
### Pod状态异常排查问题集- Terminating或Unknown状态
原因：Terminated是因为容器里的应用挂了。

从k8s v1.5开始，Kubernetes不会因为Node失联而删除其上正在运行的Pod，而是将其标记为Terminating 或 Unknown 状态。想要删除这些状态的Pod有 3 种方法：
1. 手动删除node:
手动删除kubectl delete nodes node节点名字
2. 恢复正常可选择性删除: 
若node恢复正常，kubelet会重新跟kube-apiserver通信
确认这些Pod的期待状态，进而再决定删除或者继续运行这些Pod，如果删除可以通过kubectl delete pods pod-name --grace-period=0 –force强制删除
3. 使用参数重建容器  
Pod行为异常，Pod没有按预期的行为执行，比如没有运行podSpec里面设置的命令行参数，这一般是podSpec yaml文件内容有误，可以尝试使用 validate 参数重建容器，
比如（kubectl delete pod mypod 和 kubectl create -- validate -f mypod.yaml）；也可以查看创建后的 podSpec 是否是对的，比如（kubectl get pod mypod -o yaml）

Unknown是pod状态不能通过kubelet汇报给apiserver，这很有可能是主从节点（Master和Kubelet）间的通信出现了问题。

解决思路:
1. 查看kubectl 状态
2. 查看节点状态
3. 查看apiserver是否异常
4. 查看 kubelet 日志
### Pod健康检查：存活性探测&就绪性探测
**为什么需要探针？**

如果没有探针，k8s无法知道应用是否还活着，只要pod还在运行，k8s则认为容器是健康的。但实际上，pod虽然运行了，但里面容器中的应用可能还没提供服务，那就需要做健康检查，健康检查就需要探针，常见的探针有如下几种：
1. httGet
对容器的ip地址(指定的端口和路径)执行http get请求
如果探测器收到响应，并且响应码是2xx,则认为探测成功。如果服务器没有响应或者返回错误响
应码则说明探测失败，容器将重启。
2. tcpSock  
探针与容器指定端口建立tcp连接，如果连接建立则探测成功，否则探测失败容器重启。
3. exec  
在容器内执行任意命令，并检查命令退出状态码，如果状态码为0，则探测成功，否则探测失败
容器重启

LivenessProbe和ReadinessProbe区别：  
- LivenessProbe决定是否重启容器
- ReadinessProbe主要来确定容器是否已经就绪：
只有当Pod里的容器部署的应用都处于就绪状态，也就是pod的Ready为true（1/1）时，才会将请求转发给容器。

### coredns或者kube-dns经常重启和报错
问题描述：  
在k8s中部署服务，服务之间无法通过dns解析，coredns一直处于CrashLoopBackOff状态
查看日志，报错如下：
```
plugin/loop: Loop (127.0.0.1:44222 -> :53) detected for zone
```
解决思路：  
coredns主要和resolv.conf打交道，会读取宿主机的`/etc/resolv.conf`中的nameserver内容，查询
resolv.conf文件，如果里面存在本地回环如127.0.0.1或者127.0.0.53那么就容易造成死循环

解决步骤：  
修改resolv.conf文件：
将nameserver临时修改为114.114.114.114之后：`kubectl edit deployment coredns -n kube-system`
将replicates改为0，从而停止已经启动的coredns pod 再将replicates改为2， 这样会触发coredns重新读取系统配置，此时服务的状态为Running

### kubectl执行异常：无权限问题定位
描述
```
$ kubectl get node
> The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
这是因为安装k8s之后，默认是没有权限操作k8s资源的，可用如下方法解决
```
mkdir -p $HOME/.kube
sudo cp -ifr /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```













## K8S 之统信异常——【网络故障】
### 网络故障原因
Kubernetes中的网络是非常重要的，整个k8s集群的通信调度都是通过网络插件实现的，kubernetes之上的网路通信主要有如下几种：

1. 容器间的通信: 同一个pod内的多个容器间的通信，lo
2. pod 通信： 从一个pod Ip到另一个pod Ip
3. Pod与service通信： pod ip 到 cluster ip
4. 外部通信： Service与集群外部客户端的通信

通信需要依赖网络插件，如calico、flannel、canel等，通信过程最常见的故障如下

1. Pod 自身网络故障
2. Pod 间通信故障
3. Pod 和物理节点通信故障




## K8S 之内部异常——【节点故障】
### 节点异常1: 集群节点NotReady
一般是该节点上的kubelet 挂掉了，查看kubelet 状态
```
systemctl status kubelet
```
如果挂掉需要重启一下kubelet

扩展：为什么要重启kubelet？
- 监视分配给该Node节点的pods
- 挂载pod所需要的volumes
- 下载pod的secret
- 通过docker/rkt来运行pod中的容器
- 周期的执行pod中为容器定义的liveness探针
- 上报pod的状态给系统的其他组件
- 上报Node的状态

### 节点异常2: pod调度失败故障排查
在k8s集群中，pod是最小单元，需要调度到具体的物理节点上运行，那么为了更好排查问题，了解pod的调度流程是重中之重，接下来介绍下pod工作和调度流程：

创建pod过程：
1、用户创建pod的信息通过API Server存储到etcd中，etcd记录pod的元信息并将结果返回API Server
2、API Server告知调度器请求资源调度分配，调度器通过计算，将优先级高的node与pod绑定并告知API Server
3、API Server将此信息写入etcd，得到etcd回复后调用kubelet创建pod
4、kubelet使用docker run创建pod内的容器，得到反馈信息后将容器信息告知API Server
5、API Server将收到的信息写入etcd并得到回馈
6、此时使用kubectl get pod就可以查看到信息了

调度方式：
1、当我们发送请求创建Pod，调度器就会把pod调度到合适的物理节点，大致分为以下过程：
Scheduler根据预选策略和优选函数，选额合适的noed节点调度
2、可以通过定义nodeName和nodeSelector进行调度
3、可以通过节点亲和性进行调度

Pod在调度的时候会找到一个合适的节点，所以如果节点资源不足；pod定义了指定的节点，但是指定的节点出现故障，或者pod定义了亲和性，但是节点没有满足的条件，都会导致调度失败。

排查思路
1. describe pod 查看是否节点加了污点，pod 无法容忍，当然还要根据报错信息进行定位
2. 查看目标节点 kubelet 服务的状态


pod调度失败内置污点

当某些条件为真时，节点控制器会自动为节点添加污点：

- node.kubernetes.io/ NoSchedule：如果一个pod没有声明容忍这个Taint，则系统不会把该Pod调度到有这个Taint的node上
- node.kubernetes.io/NoExecute：定义pod的驱逐行为，以应对节点故障。
- node.kubernetes.io/not-ready：节点尚未准备好。这对应于NodeConditionReady为False。
- node.kubernetes.io/unreachable：无法从节点控制器访问节点。这对应于NodeConditionReady为Unknown。
- node.kubernetes.io/out-of-disk：节点磁盘不足。
- node.kubernetes.io/memory-pressure：节点有内存压力。
- node.kubernetes.io/disk-pressure：节点有磁盘压力。
- node.kubernetes.io/network-unavailable：节点的网络不可用。
- node.kubernetes.io/unschedulable：节点不可调度。

### 节点资源不足：Pod超过节点资源限制
Pod超过节点资源限制分两种情况：
情况一：Pod数量太多超过物理节点限制，较少见
情况二：Pod资源超过物理节点资源限制，相对频繁一些，requests 资源申请过多

### K8S之内部异常 【节点故障】 排查思路总结
节点状态查询：kubectl get nodes  
常见异常现象：1、节点状态NotReady 2、调度到该节点的pod显示Unkonwn、Pending等  
常见故障：kubelet进程异常、未安装cni网络插件、docker异常、磁盘空间不足、内存不足、cpu不足  

排查排查：
- `kubectl describe nodes 查看node节点信息`
- `kubectl describe pods <pod 名字> 查看pod节点信息`
- `systemctl status 查看kubelet状态`
- `Journalctl查看系统日志``
- `free查看内存`
- `top查看cpu`


## K8S 之应用故障——【应用故障】

### 故障排查：Service访问异常
Service是什么？为什么容易出现问题？

在kubernetes中，Pod是有生命周期的，如果Pod重启IP很有可能会发生变化。如果我们的服务都是将Pod的IP地址写死，Pod的挂掉或者重启，和刚才重启的pod相关联的其他服务将会找不到它所关联的Pod，为了解决这个问题，在kubernetes中定义了service资源对象，Service 定义了一个服务访问的入口，客户端通过这个入口即可访问服务背后的应用集群实例，Service是一组Pod的逻辑集合，这一组 Pod 能够被 Service 访问到，通常是通过 Label Selector实现的。

![service](/images/kubernetes/service-labelselecotr.png "service label selector")

情况一： service 和 pod labels 没有匹配
情况二：service 和 pod 正确关联，kube-proxy 异常

#### kube-proxy 导致Service 访问异常
kube-proxy 是什么？

service 为一组相同的 pod 提供了统一的入口，起到负载均衡的效果。service 具有这样的功能，正是 kube-proxy 的功劳，kube-proxy是一个代理，安装在每一个k8s节点，当我们暴露一个 service 的时候，kube-proxy 会在iptables中追加一些规则，为我们实现路由与负载均衡的功能。

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
查看iptables 是否正常创建规则
```
iptables -t nat -L|grep 10.98.1.137
KUBE-MARK-MASQ tcp -- !10.244.0.0/16 10.98.1.137 /* default/myapp-v1-service cluster IP */ tcp dpt:http
KUBE-SVC-J2PKJ4NHOU6XHVIG tcp -- anywhere 10.98.1.137 /* default/myapp-v1-service cluster IP */ tcp
dpt:http
```
通过上面可以看到之前创建的service，会通过kube-proxy在iptables中生成一个规则，来实现流量路由。

通过上面可以看到，有一系列目标为 KUBE-SVC-xxx 链的规则，每条规则都会匹配某个目标 ip 与端口。也就是说访问某个
ip:port 的请求会由 KUBE-SVC-xxx 链来处理。这个目标 IP 其实就是service ip。
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
通过上面分析可以知道，kube-proxy在service请求中起到流量路由转发的作用：
kube-proxy 会为我们暴露的service追加iptables规则，具体流程如下：

1. 所有进出请求都会经过 KUBE-SERVICES 链
2. KUBE-SERVICES 链有我们发布的每一个服务，每个服务的链会对应到 DNAT 到 service 的 endpoint
3. KUBE-SERVICES 链最后的是 KUBE-NODEPORTS 链，会匹配请求本主机的一些端口，这就是我们通过 NODEPORT 类型发布服务能在集群外部访问的原因。

经过上面的分析，service和pod正确关联，pod也处于运行状态，那请求service还是访问不到pod，90%就是kube-proxy引起的。查看kube-proxy是否在机器上正常运行。

kube-proxy如果在节点上运行，下一步，确认它有没有出现其他异常，比如连接主节点失败。要做到这一点，必须查看日志：如 /var/log/messages或者/var/log/kube-proxy.log(二进制安装的时候可以指定的)，也使用 journalctl 访问日志、kube-proxy如果是在k8s中以pod形式运行的，可以通过kubectl logs <pod name> 查看日志。 根据日志报错信息再进行进一步的排查


### 故障排查：Pod删除失败
故障描述：在k8s中，可能会产生很多垃圾pod，也就是有些pod虽然是running状态，可是其所调度到的k8s node节点已经从k8s集群删除了，但是pod还是在这个node上，没有被驱逐，针对这种情况就需要把pod删除。

删除pod的几种方法：
1. 删除pod所在的namespace
2. 如果pod是通过控制器管理的，可以删除控制器资源
3. 如果pod是自主式管理，直接删除pod资源就可以了

但是我们常常遇见这样一种情况，我们在删除pod资源的时候，pod会一直terminate状态，很长时间无法删除

解决方案：
可以通过如下方法强制删除：
加参数 `--force --grace-period=0，grace-period`表示过渡存活期，默认
30s，在删除POD之前允许POD慢慢终止其上的容器进程，从而优雅退出，0表示立即终止POD
`kubectl delete pod <pod名字> -n --force --grace-period=0`
