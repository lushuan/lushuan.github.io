# kubernetes常用命令整理

使用 kubernetes 经常会用到命令行，这里对命令行进行分类整理

- 集群管理
  - kubectl cluster-info
  - kubectl config
  - kubectl version
  - kubectl api-versions 查看当前集群支持的api版本
  - kubectl get node
  - kubectl get pod
  - kubectl get service
  - kubectl cordon 命令：用于标记某个节点不可调度
  - kubectl uncordon 命令：用于标签节点可以调度
  - kubectl drain 命令： 用于在维护期间排除节点。
  - kubectl taint 命令：用于给某个Node节点设置污点 
- Pod 管理
  - kubectl create pod
  - kubectl get pod
  - kubectl get pod pod_name --show-labels
  - kubectl describe pod
  - kubectl logs
  - kubectl exec
  - kubectl delete pod
- 资源监测
  - kubectl top node
  - kubectl top pods
  - kubectl get quota
  - kubectl describe
- Service管理
  - kubectl create service
  - kubectl get service
  - kubectl expose
  - kubectl describe service
  - kubectl delete service
  - kubectl port-forwad
- 配置和加密
  - kubectl create configmap
  - kubectl get configmap
  - kubectl create secret
  - kubectl get secret
  - kubectl describe configmap
  - kubectl describe secret
- Deployment管理
  - kubectl create deployment
  - kubectl get deployment
  - kubectl scale deployment
    - sudo kubectl scale deployment deploy-name --replicas=0 不删除应用暂停服务
  - kubectl autoscale deployment
  - kubectl rollout status
  - kubectl rollout history
  - kubectl delete deployment
- 名字空间管理
  - kubectl create namespace
  - kubectl get namespace
  - kubectl describe namespace
  - kubectl delete namespace
  - kubectl apply -n <namespace>
  - kubectl switch -n <namespace>
- kubeadm管理
  - kubeadm init 启动引导一个 Kubernetes 主节点
  - kubeadm join 启动引导一个 Kubernetes 工作节点并且将其加入到集群
  - kubeadm upgrade 更新 Kubernetes 集群到新版本
  - kubeadm config 如果你使用 kubeadm v1.7.x 或者更低版本，你需要对你的集群做一些配置以便使用 kubeadm upgrade 命令
  - kubeadm token 使用 kubeadm join 来管理令牌
  - kubeadm reset 还原之前使用 kubeadm init 或者 kubeadm join 对节点所作改变
  - kubeadm version 打印出 kubeadm 版本
  - kubeadm alpha 预览一组可用的新功能以便从社区搜集反馈
  - kubeadm token create --print-join-command 创建node加入口令


## 命令说明
### 1. kubectl expose 和 kubectl port-forwad 的区别
kubectl expose 和 kubectl port-forward 是两个不同的命令，它们在 Kubernetes 中有不同的作用和用法。

kubectl expose 命令用于创建一个新的 Service 对象来暴露 Deployment、ReplicationController 或 ReplicaSet 等其他资源。它会为指定的资源创建一个新的 ClusterIP 类型的 Service，在集群内部通过 Service IP 和端口号来访问暴露的资源。这个 Service 对象可以是临时的，也可以是永久的，它根据指定的参数设置来控制其行为。

kubectl port-forward 命令用于在本地机器和 Kubernetes 集群之间建立一个临时的协议转发，将本地的端口与 Kubernetes Pod 或 Service 的端口进行绑定。这样可以直接通过本地机器上的端口来访问 Kubernetes 集群中的服务或应用。该命令通常用于开发和调试，比如在本地机器上直接访问运行在集群中的应用程序或服务的日志。

总结来说，kubectl expose 用于创建一个新的 Service 对象来暴露 Kubernetes 资源，而 kubectl port-forward 则用于建立本地与集群之间的临时端口转发，方便本地机器访问集群中的服务或应用程序。

### 2. kubectl apply 和 kubectl switch 的区别
kubectl apply 用于在 Kubernetes 集群上部署或更新资源对象，可以通过文件或目录进行声明。它会读取文件中的资源定义，并在集群中创建或更新相应的资源对象。kubectl switch 是一个非官方的插件，用于切换当前上下文的集群。Kubernetes 集群允许存在多个上下文，每个上下文对应一个集群。

kubectl switch 可以方便地在多个集群之间切换，目的是为了方便用户在不同的集群之间进行操作。因此，kubectl apply 用于部署和更新资源对象，而 kubectl switch 用于切换当前操作的集群。它们是两个不同的命令，用途和功能不同。

### 3. kubectl scale 和 kubectl autoscale的区别
kubectl scale和kubectl autoscale是用于扩容或缩容Kubernetes Deployment或ReplicaSet的命令，但它们有一些区别。

kubectl scale命令允许通过手动指定副本数量来更新Deployment或ReplicaSet的副本数量。例如，可以使用以下命令将副本数量设置为3：

```shell
kubectl scale deployment/my-deployment --replicas=3
```

kubectl autoscale命令则是根据指定的CPU使用率来自动扩容或缩容Deployment。它会创建一个HorizontalPodAutoscaler(HPA)对象，该对象会根据CPU的使用情况动态地增加或减少Pod的副本数量以满足指定的CPU目标使用率。例如，可以使用以下命令将Pod的副本数量自动调整为至少2个，当CPU使用率超过50%时：

```shell
kubectl autoscale deployment/my-deployment --min=2 --max=10 --cpu-percent=50
```

总结起来，kubectl scale用于手动指定副本数量来更新Deployment或ReplicaSet的副本数量，而kubectl autoscale用于根据CPU使用率自动扩容或缩容Pod的副本数量。

### 4. kubectl exec和kubectl attach 的区别
kubectl exec和kubectl attach是用于与正在运行的Pod进行交互的命令，但它们在使用方式和功能上有一些区别。

kubectl exec命令用于在正在运行的Pod中执行命令。它可以连接到正在运行的容器，并在容器中执行指定的命令。例如，可以使用以下命令在名为my-pod的Pod中执行echo命令：

```shell
kubectl exec -it my-pod -- echo "Hello, World!"
```

kubectl attach命令用于将本地终端连接到正在运行的Pod中的某个容器的标准输入、输出和错误流。它可以与容器建立交互式会话。例如，可以使用以下命令将本地终端连接到名为my-pod的Pod中的第一个容器：

```shell
kubectl attach -it my-pod
```

总结起来，kubectl attach命令类似于kubectl exec，但它附加到容器中运行的主进程，而不是运行额外的进程。

### 5. kubectl patch和kubectl replace的区别
kubectl patch和kubectl replace是用于部分更新和替换更新Kubernetes资源的命令，但它们在更新方式和行为上有一些区别。

kubectl patch命令用于部分更新Kubernetes资源的特定字段。它可以通过提供一个JSON或YAML文件、一个JSON或YAML字符串，或者使用特定的操作类型（如add、remove、replace等）来更新资源的字段。例如，可以使用以下命令将名为my-deployment的Deployment资源的labels字段添加一个新的标签：

```shell
kubectl patch deployment/my-deployment -p '{"spec":{"template":{"metadata":{"labels":{"new-label":"value"}}}}}'
```

kubectl replace命令用于替换更新整个Kubernetes资源。它会基于提供的配置文件完全替换现有资源的内容。例如，可以使用以下命令将名为my-deployment的Deployment资源替换为一个新的配置文件：

```shell
kubectl replace -f new-deployment.yaml
```

总结起来，kubectl patch用于部分更新Kubernetes资源的特定字段，而kubectl replace用于完全替换更新整个Kubernetes资源。
