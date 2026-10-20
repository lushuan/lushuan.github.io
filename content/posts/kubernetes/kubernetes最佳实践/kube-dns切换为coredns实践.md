---
title: "kube-dns切换为coredns实践"
subtitle: ""
date: 2023-03-29T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
slug: "kube-dns-to-coredns"
featuredImage: "/images/kubernetes/cover.jpg"
summary: "本文介绍将 Kubernetes 集群 DNS 从 kube-dns 切换为 CoreDNS 的实践步骤，包括版本适配、镜像准备、部署操作，并对比 CoreDNS 与 kube-dns 的差异，附完整 CoreDNS 部署 YAML。"
---

CoreDNS 相比 kube-dns 更轻量灵活，插件化架构支持更丰富的 DNS 功能。本文记录将集群 DNS 从 kube-dns 平滑切换为 CoreDNS 的完整操作步骤，并附上不同版本适配表和部署 YAML。

<!--more-->

## 一、kube-dns 切换为 CoreDNS
K8s 版本及其 DNS 适配版本

| K8s 版本| CoreDNS 版本|镜像|镜像 YAML|
|---|---|---|---|
|1.17|1.6.7|coredns-1.6.7.tar.gz|coredns-1.6.7.yaml|
|1.22|1.8.4|coredns-1.8.4.tar.gz|coredns-1.8.4.yaml|

coredns/coredns:1.6.7/1.8.4

## 二、操作
根据版本选择对应的 DNS 镜像以及 YAML 文件上传到其中一台 master 主机，以 1.8.4 版本为例

如果是内网则将 coredns-1.6.7/1.8.4.tar.gz 压缩包进行解压并上传至现场的 Harbor 仓库，如果可以连接外网则直接修改 coredns-1.8.4.yaml 文件，将占位符{{ image_address }} 进行修改

## 三、将原有的 kube-dns 停止
```shell
$ kubectl scale deploy kube-dns  --replicas=0 -n kube-system
$ kubectl scale deploy kube-dns-autoscaler  --replicas=0 -n kube-system
```
## 四、部署 CoreDNS
```shell
$ kubectl apply -f coredns.yaml
```

## 五、补充： kube-dns vs CoreDNS
CoreDNS 和 kube-dns 都是用于 Kubernetes 集群中的 DNS 解析服务，它们在不同方面有一些区别，适用于不同的场景。
另外还有一个方面是早期版本的 kube-dns 在 Kubernetes 中主要支持 IPv4。IPv6 的支持不如 CoreDNS 完善，并且在实际应用中可能会遇到限制。

CoreDNS：
- CoreDNS 是一个开源的、轻量级的、灵活的 DNS 服务器，它是一个通用的 DNS 服务器，可以用于各种不同的环境和场景，不仅限于 Kubernetes 集群。
- CoreDNS 是一个插件驱动的 DNS 服务器，可以通过插件的方式支持各种功能，例如反向代理、负载均衡、缓存、DNSSEC 等。
- CoreDNS 在 Kubernetes 集群中可以作为集群的默认 DNS 插件，用于提供内部服务发现和外部域名解析。它可以动态地解析 Kubernetes 集群内部的服务和 Pod，并提供 DNS 记录。

- kube-dns：
- kube-dns 是 Kubernetes 集群中的默认 DNS 解析服务，是一种基于 SkyDNS 的插件，与 Kubernetes 紧密集成。
- kube-dns 使用了一个集群内部的 DNS 服务器和一个外部的 DNS 服务器配合工作。集群内部的 DNS 服务器负责解析集群内的服务和 Pod，并提供 DNS 记录。外部的 DNS 服务器负责解析集群外的域名，并提供 DNS 记录。
- kube-dns 的优点是它是 Kubernetes 的默认解析器，易于配置和使用，可以满足大多数基本的服务发现和域名解析需求。

综上所述，CoreDNS 适用于更通用的环境和场景，可以满足更多的功能需求，而 kube-dns 是 Kubernetes 集群中的默认解析器，适用于基本的服务发现和域名解析需求。具体使用哪个取决于特定的需求和场景。

## 六、参考
[【Kubernetes】部署-CoreDNS-服务](https://blog.csdn.net/qq_36073886/article/details/131117066)

## 七、coredns-1.6.7.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          fallthrough in-addr.arpa ip6.arpa
        }
        template ANY AAAA {
          rcode NXDOMAIN
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. Default is 1.
  # 2. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
        - key: node.kubernetes.io/unschedulable
          effect: NoSchedule
      nodeSelector:
        kubernetes.io/role: master
      affinity:
         podAntiAffinity:
           preferredDuringSchedulingIgnoredDuringExecution:
           - weight: 100
             podAffinityTerm:
               labelSelector:
                 matchExpressions:
                   - key: k8s-app
                     operator: In
                     values: ["kube-dns"]
               topologyKey: kubernetes.io/hostname
      containers:
      - name: coredns
        image: {{ image_address }}/coredns/coredns:1.6.7 
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.254.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP


```