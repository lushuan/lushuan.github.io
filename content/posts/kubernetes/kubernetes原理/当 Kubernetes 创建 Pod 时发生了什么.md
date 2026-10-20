---
title: "当 Kubernetes 创建 Pod 时发生了什么"
subtitle: ""
date: 2023-03-26T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
slug: "what-happens-when-k8s-creates-pod"
summary: "本文通过一张全景图和相关资料，梳理 Kubernetes 创建 Pod 时各组件之间的完整交互流程。"
---

当 Kubernetes 创建一个 Pod 时，API Server、调度器、kubelet 等多个组件会协同工作完成整个流程。本文借助一张全景图和开源资料，带你直观了解 Pod 创建背后的完整链路。

<!--more-->

- [当 Kubernetes 创建 Pod 时发生了什么-全景图](https://hugo-picture.oss-cn-beijing.aliyuncs.com/what-happens-when-k8s.svg)
- https://github.com/jamiehannaford/what-happens-when-k8s/tree/master/zh-cn

![what-happens-when-k8s](/images/kubernetes/what-happens-when-k8s.svg "what-happens-when-k8s")
