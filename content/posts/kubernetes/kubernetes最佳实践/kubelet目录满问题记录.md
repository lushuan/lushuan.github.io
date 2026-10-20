---
title: "kubelet目录满问题记录"
subtitle: ""
date: 2023-04-04T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
slug: "kubelet-dir-disk-full"
summary: "记录一次 /var/lib/kubelet 目录空间占满的排查过程，通过 du 逐层定位到 Prometheus 容器使用 emptyDir 挂载 /data 导致磁盘占用过大。"
---

某台主机 /var/lib/kubelet 目录使用率超过 80%，占用 40 多 G 存储。本文记录通过 du 命令逐层定位，最终发现是 Prometheus 容器使用 emptyDir 挂载内部 /data 目录导致的磁盘占用问题。

<!--more-->

## 一、现象
K8s 某台主机的/var/lib/kubelet目录存储使用率超过80% 使用了 40 多 G 存储

## 二、分析
1. 申请 root 权限  
2. cd /var/lib/kubelet
执行 du -h -d 1
发现 pod 目录占据大量存储

3. cd pod  
继续执行 du -h -d 1  
定位到 pod  
2753dcf9-4113-4e65-a944-278bcbd8d6d8  

4. 按上面方法继续执行  
发现最终路径为  
/var/lib/kubelet/pods/2753dcf9-4113-4e65-a944-278bcbd8d6d8/volumes/kubernetes.io~empty-dir/storage-volume

5. Docker ps|grep 2753dcf9-4113-4e65-a944-278bcbd8d6d8  
定位容器为 Prometheus  

6. docker inspect 122ec2fbbd1e  
![kubelet](/images/kubernetes/problem_and_solve/kubelet.png "kubelet")
该容器使用了 empty-dir 挂载内部的/data
