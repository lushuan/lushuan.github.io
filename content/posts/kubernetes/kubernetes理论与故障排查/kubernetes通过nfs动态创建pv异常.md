---
title: "Kubernetes 通过 nfs 动态创建 pv 异常"
subtitle: ""
date: 2022-05-18T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes排障"]
slug: "nfs-dynamic-pv-selflink"
summary: "记录 StatefulSet 通过 NFS 动态创建 PV 失败的排障过程，梳理 PVC 无法绑定 PV 的常见原因，并说明 v1.20 后 selfLink 字段默认移除对 NFS 动态供应的影响及开启方法。"
---

StatefulSet 通过 NFS 动态创建 PV 时发现 PVC 一直无法绑定。本文梳理 NFS 动态供应失败的常见原因，并重点说明 Kubernetes v1.20 之后 selfLink 字段默认被移除导致的问题，以及通过 --feature-gates=RemoveSelfLink=false 显式开启的方法。

<!--more-->

## 一、背景
statefulset 通过 NFS 动态创建 pv，发现无法创建成功

## 二、报错日志
```
 Warning  FailedMount  2m50s (x6 over 6m4s)  kubelet  Unable to attach or mount volumes: unmounted volumes=[data], unattached volumes=[kube-api-access-8p8wf data]: error processing PVC kube-logging/data-es-cluster-0: PVC is not bound
```

通过报错日志可以看得出来 PVC 无法绑定 pv，通过命令查看 PVC 创建成功，pv 则没有进行创建

StatefulSet 挂载 NFS 存储卷时出现了问题，导致 PVC 没有被成功绑定到 PV

常见原因有：

1. PVC 和 PV 不匹配

PVC 的 storage class、访问模式、大小资源请求等需要和 PV 定义一致，否则不会被匹配到合适的 PV。

2. NFS 服务器配置问题 

NFS 服务器需要正常启动、导出共享目录，并且 Kubernetes 节点能够访问到 NFS 服务器。

3. RBAC 鉴权问题

Kubernetes 节点上的 kubelet 需要有获取、挂载 PV 的权限。

4. NFS Client 配置问题

Kubernetes 节点上需要安装 NFS Client，并且配置可以访问 NFS 服务器。

5. StatefulSet 配置错误

StatefulSet 的 volumeClaimTemplates 需要与 PVC 的定义相匹配。

6. 存储卷读写权限问题

存储卷需要有读写权限，避免因权限问题无法挂载。

## 三、NFS 在 Kubernetes 中动态创建 pv 和版本的关系

在早期的 Kubernetes 版本中（如 v1.11 之前）,NFS  provisioner 并不包含在默认部署中，这会导致通过 NFS 存储类无法动态创建 PV。

从 Kubernetes v1.11 开始，NFS 动态供应功能成为了默认部署的一部分，但也需要进行额外的配置，主要步骤包括：

1. 安装 NFS 客户端组件
2. 部署 NFS 动态供应器 external-provisioner
3. 创建 StorageClass，指定 NFS 作为 provisioner
4. 创建 PersistentVolumeClaim，引用该 StorageClass

此时，NFS 供应器就可以根据 PVC 的请求动态创建 PV 来绑定 PVC。

所以简单来说，Kubernetes 低版本中需要手动安装和配置 NFS 动态供应器；
高版本中已内置但也需要显式配置才能启用该功能。正确配置后就可以通过 NFS StorageClass 来动态创建 PV。

这里的高版本指的是 v1.20 之后，在 kube-apiserver 的启动参数中不移除 API 对象中的 selfLink 字段。
selfLink 是 Kubernetes 中每个 API 对象的一个字段，用于表示对象自身的 URL 地址

- 从 Kubernetes 1.20 版本开始，该字段默认被移除了。但可以通过设置 RemoveSelfLink=false 来保留该字段
- 原因是 selfLink 字段已很少被用到，但删去后可能会影响部分老版本的客户端。所以提供了这个特性开关来向后兼容
- 一般来说，除非确实需要兼容老版本客户端，否则不建议保留 selfLink 字段

显式开启方式：  

```shell
$ vi /etc/kubernetes/manifests/kube-apiserver.yaml
....
spec:
  containers:
  - command:
    - kube-apiserver
    - --feature-gates=RemoveSelfLink=false
```