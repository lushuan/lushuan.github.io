# Kubernetes Pod 排障指南

## Pod 一直处于 ContainerCreating 或 Waiting 状态
### Pod 配置错误
- 检查是否打包了正确的镜像
- 检查配置了正确的容器参数
### 磁盘爆满
启动 Pod 会调 CRI 接口创建容器，容器运行时创建容器时通常会在数据目录下为新建的容器创建一些目录和文件，如果数据目录所在的磁盘空间满了就会创建失败并报错:
```
Events:
  Type     Reason                  Age                  From                   Message
  ----     ------                  ----                 ----                   -------
  Warning  FailedCreatePodSandBox  2m (x4307 over 16h)  kubelet, 10.179.80.31  (combined from similar events): Failed create pod sandbox: rpc error: code = Unknown desc = failed to create a sandbox for pod "apigateway-6dc48bf8b6-l8xrw": Error response from daemon: mkdir /var/lib/docker/aufs/mnt/1f09d6c1c9f24e8daaea5bf33a4230de7dbc758e3b22785e8ee21e3e3d921214-init: no space left on device
```
### limit 设置太小或者单位不对
如果 limit 设置过小以至于不足以成功运行 Sandbox 也会造成这种状态，常见的是因为 memory limit 单位设置不对造成的 limit 过小，比如误将 memory 的 limit 单位像 request 一样设置为小 m，这个单位在 memory 不适用，会被 k8s 识别成 byte， 应该用 Mi 或 M。，

举个例子: 如果 memory limit 设为 1024m 表示限制 1.024 Byte，这么小的内存， pause 容器一起来就会被 cgroup-oom kill 掉，导致 pod 状态一直处于 ContainerCreating。

这种情况通常会报下面的 event:
```
Pod sandbox changed, it will be killed and re-created。
```
kubelet报错
```
to start sandbox container for pod ... Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "process_linux.go:301: running exec setns process for init caused \"signal: killed\"": unknown
```

### 拉取镜像失败
镜像拉取失败也分很多情况，这里列举下:

- 配置了错误的镜像
- Kubelet 无法访问镜像仓库（比如默认 pause 镜像在 gcr.io 上，国内环境访问需要特殊处理）
- 拉取私有镜像的 imagePullSecret 没有配置或配置有误
- 镜像太大，拉取超时（可以适当调整 kubelet 的 —image-pull-progress-deadline 和 —runtime-request-timeout 选项）

### controller-manager 异常
查看 master 上 kube-controller-manager 状态，异常的话尝试重启。

## Pod 一直处于 Error 状态
通常处于 Error 状态说明 Pod 启动过程中发生了错误。常见的原因包括：

- 依赖的 ConfigMap、Secret 或者 PV 等不存在
- 请求的资源超过了管理员设置的限制，比如超过了 LimitRange 等
- 违反集群的安全策略
- 容器无权操作集群内的资源，比如开启 RBAC 后，需要为 ServiceAccount 配置角色绑定


## Pod 一直处于 ImagePullBackOff 状态
### http 类型 registry，地址未加入到 insecure-registry
dockerd 默认从 https 类型的 registry 拉取镜像，如果使用 https 类型的 registry，则必须将它添加到 insecure-registry 参数中，然后重启或 reload dockerd 生效。

### https 自签发类型 resitry，没有给节点添加 ca 证书
如果 registry 是 https 类型，但证书是自签发的，dockerd 会校验 registry 的证书，校验成功才能正常使用镜像仓库，
要想校验成功就需要将 registry 的 ca 证书放置到 /etc/docker/certs.d/\<registry:port>/ca.crt 位置。

### 私有镜像仓库认证失败
如果 registry 需要认证，但是 Pod 没有配置 imagePullSecret，配置的 Secret 不存在或者有误都会认证失败。

### 镜像文件损坏
如果 push 的镜像文件损坏了，下载下来也用不了，需要重新 push 镜像文件

### 镜像拉取超时
如果节点上新起的 Pod 太多就会有许多可能会造成容器镜像下载排队，如果前面有许多大镜像需要下载很长时间，后面排队的 Pod 就会报拉取超时。

### 镜像不存在
kubelet 日志：
```
PullImage "imroc/test:v0.2" from image service failed: rpc error: code = Unknown desc = Error response from daemon: manifest for imroc/test:v0.2 not found
```

## Pod 一直处于 Pending 状态
Pending 状态说明 Pod 还没有被调度到某个节点上，需要看下 Pod 事件进一步判断原因，比如:
```shell
$ kubectl describe pod tikv-0
...
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  3m (x106 over 33m)  default-scheduler  0/4 nodes are available: 1 node(s) had no available volume zone, 2 Insufficient cpu, 3 Insufficient memory.
```
下面列举下可能原因和解决方法。

### 节点资源不够
节点资源不够有以下几种情况:

- CPU 负载过高
- 剩余可以被分配的内存不够

如果判断某个 Node 资源是否足够？ 通过 kubectl describe node <node-name> 查看 node 资源情况，关注以下信息：

- Allocatable: 表示此节点能够申请的资源总和
- Allocated resources: 表示此节点已分配的资源 (Allocatable 减去节点上所有 Pod 总的 Request)
```shell
$ sudo kubectl describe node 10.10.192.220|grep Allo -A 6
Allocatable:
  cpu:                8
  ephemeral-storage:  33806352329
  hugepages-2Mi:      0
  memory:             24543516Ki
  pods:               110
System Info:
--
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                3650m (45%)   50150m (626%)
  memory             5092Mi (21%)  81290Mi (339%)
  ephemeral-storage  0 (0%)        0 (0%)
```
可以看到能够申请的资源总和，当前节点可以创建110个pods，cpu 8核，cpu requests占比45%

前者与后者相减，可得出剩余可申请的资源。如果这个值小于 Pod 的 request，就不满足 Pod 的资源要求，Scheduler 在 Predicates (预选) 阶段就会剔除掉这个 Node，也就不会调度上去

### 不满足 nodeSelector 与 affinity
如果 Pod 包含 nodeSelector 指定了节点需要包含的 label，调度器将只会考虑将 Pod 调度到包含这些 label 的 Node 上，如果没有 Node 有这些 label 或者有这些 label 的 Node 其它条件不满足也将会无法调度。参考官方文档：

https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/

如果 Pod 包含 affinity（亲和性）的配置，调度器根据调度算法也可能算出没有满足条件的 Node，从而无法调度。affinity 有以下几类:

- nodeAffinity: 节点亲和性，可以看成是增强版的 nodeSelector，用于限制 Pod 只允许被调度到某一部分 Node。
- podAffinity: Pod 亲和性，用于将一些有关联的 Pod 调度到同一个地方，同一个地方可以是指同一个节点或同一个可用区的节点等。
- podAntiAffinity: Pod 反亲和性，用于避免将某一类 Pod 调度到同一个地方避免单点故障，比如将集群 DNS 服务的 Pod 副本都调度到不同节点，避免一个节点挂了造成整个集群 DNS 解析失败，使得业务中断。

### Node 存在 Pod 没有容忍的污点
如果节点上存在污点 (Taints)，而 Pod 没有响应的容忍 (Tolerations)，Pod 也将不会调度上去。通过 describe node 可以看下 Node 有哪些 Taints:
```shell
$ kubectl describe nodes host1
...
Taints:             special=true:NoSchedule
...
```
污点既可以是手动添加也可以是被自动添加

> 手动添加污点
```shell
$ kubectl taint node host1 special=true:NoSchedule
node "host1" tainted
```
另外，有些场景下希望新加的节点默认不调度 Pod，直到调整完节点上某些配置才允许调度，就给新加的节点都加上 node.kubernetes.io/unschedulable 这个污点

> 自动添加污点
如果节点运行状态不正常，污点也可以被自动添加，从 v1.12 开始，TaintNodesByCondition 特性进入 Beta 默认开启，controller manager 会检查 Node 的 Condition，如果命中条件就自动为 Node 加上相应的污点，这些 Condition 与 Taints 的对应关系如下:
```
Conditon               Value       Taints
--------               -----       ------
OutOfDisk              True        node.kubernetes.io/out-of-disk
Ready                  False       node.kubernetes.io/not-ready
Ready                  Unknown     node.kubernetes.io/unreachable
MemoryPressure         True        node.kubernetes.io/memory-pressure
PIDPressure            True        node.kubernetes.io/pid-pressure
DiskPressure           True        node.kubernetes.io/disk-pressure
NetworkUnavailable     True        node.kubernetes.io/network-unavailable
```
解释下上面各种条件的意思:
- OutOfDisk 为 True 表示节点磁盘空间不够了
- Ready 为 False 表示节点不健康
- Ready 为 Unknown 表示节点失联，在 node-monitor-grace-period 这么长的时间内没有上报状态 controller-manager 就会将 Node 状态置为 Unknown (默认 40s)
- MemoryPressure 为 True 表示节点内存压力大，实际可用内存很少
- PIDPressure 为 True 表示节点上运行了太多进程，PID 数量不够用了
- DiskPressure 为 True 表示节点上的磁盘可用空间太少了
- NetworkUnavailable 为 True 表示节点上的网络没有正确配置，无法跟其它 Pod 正常通信

### kube-scheduler 没有正常运行
检查 maser 上的 kube-scheduler 是否运行正常，异常的话可以尝试重启临时恢复。
```shell
$ sudo kubectl -n kube-system get pod|grep kube-scheduler
kube-scheduler-10.10.192.220              1/1     Running   183 (40d ago)   424d
```

## Pod 一直处于 Terminating 状态
### 磁盘爆满
如果 docker 的数据目录所在磁盘被写满，docker 无法正常运行，无法进行删除和创建操作，所以 kubelet 调用 docker 删除容器没反应，看 event 类似这样：
```
Normal  Killing  39s (x735 over 15h)  kubelet, 10.179.80.31  Killing container with id docker://apigateway:Need to kill Pod
```

### 存在 "i" 文件属性
如果容器的镜像本身或者容器启动后写入的文件存在 "i" 文件属性，此文件就无法被修改删除，而删除 Pod 时会清理容器目录，但里面包含有不可删除的文件，就一直删不了，Pod 状态也将一直保持 Terminating，kubelet 报错:
```
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.922965   14109 remote_runtime.go:250] RemoveContainer "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257" from runtime service failed: rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.923027   14109 kuberuntime_gc.go:126] Failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
```
通过 man chattr 查看 "i" 文件属性描述:
```
       A file with the 'i' attribute cannot be modified: it cannot be deleted or renamed, no
link can be created to this file and no data can be written to the file.  Only the superuser
or a process possessing the CAP_LINUX_IMMUTABLE capability can set or clear this attribute.
```
彻底解决当然是不要在容器镜像中或启动后的容器设置 "i" 文件属性，临时恢复方法： 复制 kubelet 日志报错提示的文件路径，然后执行 chattr -i <file>:
```
chattr -i /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash
```
执行完后等待 kubelet 自动重试，Pod 就可以被自动删除了。

### docker 17 的 bug
docker hang 住，没有任何响应，看 event:
```
Warning FailedSync 3m (x408 over 1h) kubelet, 10.179.80.31 error determining status: rpc error: code = DeadlineExceeded desc = context deadline exceeded
```
怀疑是17版本dockerd的BUG。可通过 kubectl -n cn-staging delete pod apigateway-6dc48bf8b6-clcwk --force --grace-period=0 强制删除pod，但 docker ps 仍看得到这个容器

处置建议：
- 升级到docker 18. 该版本使用了新的 containerd，针对很多bug进行了修复。
- 如果出现terminating状态的话，可以提供让容器专家进行排查，不建议直接强行删除，会可能导致一些业务上问题。

### 存在 Finalizers
k8s 资源的 metadata 里如果存在 finalizers，那么该资源一般是由某程序创建的，并且在其创建的资源的 metadata 里的 finalizers 加了一个它的标识，这意味着这个资源被删除时需要由创建资源的程序来做删除前的清理，清理完了它需要将标识从该资源的 finalizers 中移除，然后才会最终彻底删除资源。比如 Rancher 创建的一些资源就会写入 finalizers 标识。

处理建议：kubectl edit 手动编辑资源定义，删掉 finalizers，这时再看下资源，就会发现已经删掉了。通过prometheus-operator 创建prometheus时，最后无法删除namespace 也是此原因。

## Pod 一直处于 Unknown 状态
通常是节点失联，没有上报状态给 apiserver，到达阀值后 controller-manager 认为节点失联并将其状态置为 Unknown。

可能原因:

- 节点高负载导致无法上报
- 节点宕机
- 节点被关机
- 网络不通

## Pod 健康检查失败
- Kubernetes 健康检查包含就绪检查(readinessProbe)和存活检查(livenessProbe)
- pod 如果就绪检查失败会将此 pod ip 从 service 中摘除，通过 service 访问，流量将不会被转发给就绪检查失败的 pod
- pod 如果存活检查失败，kubelet 将会杀死容器并尝试重启

健康检查失败的可能原因有多种，除了业务程序BUG导致不能响应健康检查导致 unhealthy，还能有有其它原因，下面我们来逐个排查。

### 健康检查配置不合理
initialDelaySeconds 太短，容器启动慢，导致容器还没完全启动就开始探测，如果 successThreshold 是默认值 1，检查失败一次就会被 kill，然后 pod 一直这样被 kill 重启。

### 节点负载高
cpu 占用高（比如跑满）会导致进程无法正常发包收包，通常会 timeout，导致 kubelet 认为 pod 不健康。

### 容器内进程端口监听挂掉
使用 netstat -tunlp 检查端口监听是否还在，如果不在了，抓包可以看到会直接 reset 掉健康检查探测的连接:
```
20:15:17.890996 IP 172.16.2.1.38074 > 172.16.2.23.8888: Flags [S], seq 96880261, win 14600, options [mss 1424,nop,nop,sackOK,nop,wscale 7], length 0
20:15:17.891021 IP 172.16.2.23.8888 > 172.16.2.1.38074: Flags [R.], seq 0, ack 96880262, win 0, length 0
20:15:17.906744 IP 10.0.0.16.54132 > 172.16.2.23.8888: Flags [S], seq 1207014342, win 14600, options [mss 1424,nop,nop,sackOK,nop,wscale 7], length 0
20:15:17.906766 IP 172.16.2.23.8888 > 10.0.0.16.54132: Flags [R.], seq 0, ack 1207014343, win 0, length 0
```
连接异常，从而健康检查失败。发生这种情况的原因可能在一个节点上启动了多个使用 hostNetwork 监听相同宿主机端口的 Pod，只会有一个 Pod 监听成功，但监听失败的 Pod 的业务逻辑允许了监听失败，并没有退出，Pod 又配了健康检查，kubelet 就会给 Pod 发送健康检查探测报文，但 Pod 由于没有监听所以就会健康检查失败。

## Pod 处于 CrashLoopBackOff 状态
Pod 如果处于 CrashLoopBackOff 状态说明之前是启动了，只是又异常退出了，应用实例状态为CrashLoopBackOff， 表现为实例不断重启，由ready状态变为Complete再变成CrashLoopBackOff。
一般由于应用实例容器异常退出导致的实例异常，需要排查应用日志确定异常退出原因。
### 容器进程主动退出
如果是容器进程主动退出，退出状态码一般在 0-128 之间，除了可能是业务程序 BUG，还有其它许多可能原因

### 系统OOM
如果发生系统 OOM，可以看到 Pod 中容器退出状态码是 137，表示被 SIGKILL 信号杀死，同时内核会报错: Out of memory: Kill process ...。大概率是节点上部署了其它非 K8S 管理的进程消耗了比较多的内存，或者 kubelet 的 --kube-reserved 和 --system-reserved 配的比较小，没有预留足够的空间给其它非容器进程，节点上所有 Pod 的实际内存占用总量不会超过 /sys/fs/cgroup/memory/kubepods 这里 cgroup 的限制，这个限制等于 capacity - "kube-reserved" - "system-reserved"，如果预留空间设置合理，节点上其它非容器进程（kubelet, dockerd, kube-proxy, sshd 等) 内存占用没有超过 kubelet 配置的预留空间是不会发生系统 OOM 的，可以根据实际需求做合理的调整。

### cgroup OOM
如果是 cgroup OOM 杀掉的进程，从 Pod 事件的下 Reason 可以看到是 OOMKilled，说明容器实际占用的内存超过 limit 了，同时内核日志会报: ``。 可以根据需求调整下 limit


