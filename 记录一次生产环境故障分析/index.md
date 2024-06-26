# Kubernetes 记录一次项目生产环境故障分析

## 问题现象
生产环境无法访问，这里主要是梳理遇到问题应该有的一个排查思路

![portal-exception](/images/kubernetes/problem_and_solve/portal.png)

## 问题排查
> 1. 登录节点查看namespace 下各pod状态
```shell
kubectl get pod -o wide -n prod
```
发现portal cmdb application等均处于异常状态

> 2. 由于portal启动会依赖cmdb

优先查看cmdb的日志，发现报错连接redis异常

> 3. 查看redis状态 处于正常状态
```shell
kubectl get pod -o wide -n prod|grep redis
```
> 4. 进入cmdb 容器进行redis 连接测试
```shell
telnet redis 6379
```
发现解析域名 redis 有问题

> 5. 因此怀疑coredns存在问题

> 6. 查看 coredns 状态
```shell
kubectl get pod -o wide -n kube-system|grep coredns
```
发现pod处于terminating以及pending

> 7. 由于coredns配置有节点选择器，只会调度到k8s master节点
此外对master做了taint ,----防止其它的各种系统组件向Master调度，导致master资源受压缩。（此污点对已经调度在该节点的pod不会产生驱逐，但是新建pod的将无法调度）
```shell
$ kubectl describe node 10.10.xxx
Name:               10.10.xxx
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=10.10.xxx
                    kubernetes.io/os=linux
                    kubernetes.io/role=master
                    zcm.role=k8s
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sat, 25 Mar 2023 10:08:15 +0800
Taints:             scheduler=custom:NoSchedule
Unschedulable:      false
```
导致coredns pending
> 8. 临时处理,将节点选择器移除
coredns调度成功，启动完成

> 9. portal cmdb等依赖redis的服务自行恢复

> 10. 生产环境恢复访问

## 问题分析
问题发生前，集成对10.10.xxx等k8s master机器进行了迁移操作，主机发生了重启。

因此coredns发生重新调度，此时由于节点选择器以及taint的缘故，coredns无法成功调度启动，

进而影响了容器内对redis的解析 ，导致依赖redis的容器不断重启，生产环境无法访问。

后续对各个k8s集群的coredns配置了对污点的容忍，避免类似问题的再次发生。

