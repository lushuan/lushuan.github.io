# k8s容器运行时 containerd 版本升级

## 背景
k8s 容器运行时是docker和containerd结合使用，且生产环境是离线环境将需要的containerd和docker版本相关的rpm包通过yum 进行升级，升级过程涉及到docker的重启，docker容器将会导致所有的容器进行重启，故升级前需要将k8s该节点先设置成维护状态，待升级结束再解除，本文对离线运行时升级进行一个记录，至于为什么需要将Docker和containerd进行结合使用参考下面补充介绍

## 升级前提和影响
docker 版本低于 19.03.xx

containerd 版本低于 1.3.9

升级后的版本
```shell
$ sudo docker version
Client: Docker Engine - Community
 Version:           19.03.11
 API version:       1.40
 Go version:        go1.13.10
 Git commit:        42e35e61f3
 Built:             Mon Jun  1 09:13:48 2020
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.11
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.13.10
  Git commit:       42e35e61f3
  Built:            Mon Jun  1 09:12:26 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.3.9
  GitCommit:        ea765aba0d05254012b0b9e595e995c09186427f
 runc:
  Version:          1.0.0-rc10
  GitCommit:        dc9208a3303feef5b3839f4323d9beb36df0a9dd
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683

$ containerd --version
containerd containerd.io 1.3.9 ea765aba0d05254012b0b9e595e995c09186427f
```

## 升级操作
### 获取升级前集群所有节点状态
```shell
$ kubectl get nodes > before-update-node-status.txt 
```
### 升级过程
升级原则，升级 1 个，检查 OK 后，再升级另外 1 个节点

### 升级前操作
将准备的升级介质上传至yum源
```shell
# rpm 包指定文件夹 /repos/zcm-custom/centos/7/x86_64/
mv docker-ce-19.03.11.tar.gz /repos/zcm-custom/centos/7/x86_64/
cd /repos/zcm-custom/centos/7/x86_64/
tar zxvf docker-ce-19.03.11.tar.gz
cd /repos/zcm-custom/centos/7/
createrepo --update  x86_64/  # 更新yum源仓库
yum makecache  # #重新生成缓存
yum  list  docker-ce
```

检查
```shell
$ sudo yum list docker-ce
Loaded plugins: product-id, search-disabled-repos
Repodata is over 2 weeks old. Install yum-cron? Or run: yum makecache fast
Installed Packages
docker-ce.x86_64                    3:19.03.11-3.el7                                                                             @zcm-custom
```
能看到 19.03.11 说明 yum 源更新成功

### 升级步骤
1. 将被升级节点置为非调度状态；对被升级节点执行驱逐操作，将该节点上的业务容器飘到其它节点

```shell
$ /usr/local/bin/kubectl cordon 10.45.80.44
$ /usr/local/bin/kubectl drain 10.45.80.44 --ignore-daemonsets
```

2. 停止 kubelet 服务、停止 kube-proxy 服务
```shell
$ sudo systemctl stop kube-proxy
$ sudo systemctl stop kubelet
```

3. 停止 docker/containerd 服务
```shell
$ sudo systemctl stop docker
$ sudo systemctl stop containerd
$ sudo ps aux | grep docker   # 检查该机器上所有与 docker 相关的进程
# 如果存在，则 kill 掉
$ sudo ps -ef|grep docker|grep -v dockerd|awk '{print $2}'|xargs kill -9
```

4. containerd 升级
```shell
$ yum install -y containerd
```

5. 启动升级后的 docker/containerd 服务
```shell
$ sudo systemctl start containerd
$ sudo systemctl start docker
$ sudo systemctl enable docker
$ sudo systemctl enable containerd
```
6. 启动 kubelet 服务、启动 kube-proxy 服务
```shell
$ sudo systemctl start kube-proxy
$ sudo systemctl start kubelet
```
7. 登录 master 节点将升级节点状态设置为允许调度状态
```shell
$ sudo kubectl get nodes
$ sudo kubectl uncordon 10.45.80.44
```
8. 被升级节点的状态检查

检查被升级节点的状态

- 在 k8s master 节点上，执行 kubectl get nodes |grep 被升级节点的 IP 看看是否处于 Ready 状态。
- 检查被升级节点的容器网络，在升级完后的节点上 ping 一下其它机器上的容器 IP，ping 通的话，容器网络正常
- 该节点升级完成
- 依次升级其它节点，按前面的升级步骤位次升级后续的节点，在升级后续节点的过程中，业务容器会在前面升级完成后的节点上调度


## 升级后的节点状态检查
```shell
$ kubectl get nodes 
```

## 补充-为什么要将Docker和containerd进行结合使用？

1. 更轻量级：containerd是一个更轻量级且专注于容器操作的运行时工具，相比于Docker具有更小的内存占用和更快的启动时间。

2. 更稳定和可靠：containerd是由Docker开源社区开发和维护的，因此可以受益于Docker社区的经验和反馈，从而获得更稳定和可靠的容器技术。

3. 更灵活的集成：containerd被设计为模块化的容器运行时，可以与其他容器生态系统中的工具和服务集成，例如Kubernetes、CRI-O等。

4. 更高性能：由于containerd比Docker更轻量级，因此可以获得更高的性能。它可以快速创建和销毁容器，并提供更低的延迟和更高的吞吐量。

5. 兼容性：通过使用Docker作为容器镜像格式，containerd能够与Docker生态系统中的工具和服务无缝集成，而无需进行任何修改。

综上所述，使用Docker和containerd结合可以提供更轻量级、更稳定可靠、更灵活集成、更高性能和更好的兼容性。这使得它成为一个强大的容器运行时组合，适用于各种不同的容器化场景。

## 参考
[Docker 与 Containerd 并用配置](https://www.cnblogs.com/hahaha111122222/p/16451663.html)  
[手把手教你搭建Linux离线YUM源环境](https://blog.csdn.net/weixin_43770382/article/details/118334644#)
