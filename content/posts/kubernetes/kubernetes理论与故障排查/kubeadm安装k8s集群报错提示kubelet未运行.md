---
title: "Kubeadm 安装k8s集群报错提示 kubelet 未运行"
subtitle: ""
date: 2021-05-28T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes排障"]
slug: "kubeadm-kubelet-cgroup-driver"
summary: "记录 kubeadm 安装 Kubernetes v1.20 集群时报错 kubelet 未运行的排障过程：根因是 Docker 与 kubelet 的 cgroup 驱动不一致，通过将 Docker 驱动改为 systemd 解决。"
---

使用 kubeadm init 安装 Kubernetes v1.20 集群时报错提示 kubelet 未运行或健康检查失败。本文记录排查过程：根因是 Docker 默认 cgroupfs 驱动与 kubelet 的 systemd 驱动不一致，修改 /etc/docker/daemon.json 统一为 systemd 后解决。

<!--more-->

## 一、背景
通过 kubeadm 安装 K8s v1.20 集群报错
操作系统环境信息
```Bash
$ cat /etc/os-release 
NAME="Ubuntu"
VERSION="18.04.5 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.5 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
```

kubeadm init 安装报错信息
```
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused.

        Unfortunately, an error has occurred:
                timed out waiting for the condition

        This error is likely caused by:
                - The kubelet is not running
                - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

        If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
                - 'systemctl status kubelet'
                - 'journalctl -xeu kubelet'

        Additionally, a control plane component may have crashed or exited when started by the container runtime.
        To troubleshoot, list all containers using your preferred container runtimes CLI.

        Here is one example how you may list all Kubernetes containers running in docker:
                - 'docker ps -a | grep kube | grep -v pause'
                Once you have found the failing container, you can inspect its logs with:
                - 'docker logs CONTAINERID'
```
## 二、排查思路
查看官网介绍为 Docker 和 kubelet  服务中的 cgroup 驱动不一致，有两种方法  
方式一：驱动向 Docker 看齐  
方式二：驱动向 kubelet 看齐  
如果 Docker 不方便重启则统一向 kubelet 看齐，并重启对应的服务即可

## 三、解决方式
### Docker 配置文件
这里采取的是方式二，Docker 默认驱动为 cgroupfs，只需要添加
```
 "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
```
修改后配置文件
```Bash
$ cat /etc/docker/daemon.json 
{
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "bip":"172.12.0.1/24",
  "registry-mirrors": [
    "http://docker-registry-mirror.kodekloud.com"
  ]
}
```
重启 Docker
`systemctl restart docker`
### kubelet 配置文件
grep 截取一下，可以看得出来 kubelet 默认 cgroup 驱动为 systemd
```Bash
$ cat /var/lib/kubelet/config.yaml |grep group
cgroupDriver: systemd
```
重启 kubelet（optional）

`systemctl restart kubelet`

## 四、参考

- [配置 cgroup 驱动](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#configure-cgroup-driver)  
- [Docker 中的 Cgroup Driver:Cgroupfs 与 Systemd](https://www.cnblogs.com/hongdada/p/9771857.html)  
- [为什么要修改 Docker 的 cgroup driver](http://www.manongjc.com/detail/20-nytgptlajikhhpx.html)