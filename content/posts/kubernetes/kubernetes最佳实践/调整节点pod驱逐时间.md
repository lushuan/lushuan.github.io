---
title: "调整节点pod驱逐时间"
subtitle: ""
date: 2023-09-09T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
slug: "tune-pod-eviction-timeout"
summary: "介绍 Kubernetes 节点故障时 Pod 驱逐时间的调整方法，梳理 kubelet 状态更新的相关参数（如 --pod-eviction-timeout），并给出修改 kube-controller-manager 配置以加快 Pod 迁移的操作步骤。"
---

当 Node 节点故障、kubelet 无法工作时，Pod 会自动迁移到其他节点，迁移时长直接影响业务恢复速度与集群压力。本文梳理 kubelet 状态更新流程中的关键参数，并演示如何通过修改 kube-controller-manager 的 --pod-eviction-timeout 调整 Pod 驱逐时间。

<!--more-->

## 一、问题描述
{{< admonition question >}}
在高可用的 K8s 集群中，当 Node 节点挂掉，kubelet 无法提供工作的时候，pod 将会自动调度到其他的节点上去，
而调度到节点上的时间需要我们慎重考量，因为它决定了生产的稳定性、可靠性，更快的迁移可以减少对我们业务的影响，
但是有可能会对集群造成一定的压力，从而造成集群崩溃。
{{< /admonition >}}

## 二、Kubelet 状态更新的基本流程

1. kubelet 自身会定期更新状态到 apiserver，通过参数--node-status-update-frequency 指定上报频率，默认是 10s 上报一次。
2. kube-controller-manager 会每隔--node-monitor-period 时间去检查 kubelet 的状态，默认是 5s。
3. 当 node 失联一段时间后，Kubernetes 判定 node 为 notready 状态，这段时长通过--node-monitor-grace-period 参数配置，默认 40s。
4. 当 node 失联一段时间后，Kubernetes 判定 node 为 unhealthy 状态，这段时长通过--node-startup-grace-period 参数配置，默认 1m0s。
5. 当 node 失联一段时间后，Kubernetes 开始删除原 node 上的 pod，这段时长是通过--pod-eviction-timeout 参数配置，默认 5m0s。

kube-controller-manager 和 kubelet 是异步工作的，这意味着延迟可能包括任何的网络延迟、apiserver 的延迟、etcd 延迟，一个节点上的负载引起的延迟等等。因此，如果--node-status-update-frequency 设置为 5s，那么实际上 etcd 中的数据变化会需要 6-7s，甚至更长时间。

如果业务生产认为默认的驱逐时间不能满足业务的及时性，可以手动修改驱逐时间。

在 master 节点操作：

修改/etc/systemd/system/kube-controller-manager.service文件

增加一行--pod-eviction-timeout=1m（这里是调整为 1 分钟，具体数值可以根据业务的重要性和实际情况修改）