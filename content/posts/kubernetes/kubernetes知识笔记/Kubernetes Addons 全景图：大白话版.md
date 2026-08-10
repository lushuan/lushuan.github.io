---
title: "一文搞懂 Kubernetes Addons 生态：分类解读、选型建议与避坑指南"
subtitle: ""
date: 2025-03-23T12:06:37+08:00
lastmod: 2025-03-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
---

## 前言

刚接触 Kubernetes 的同学往往会遇到一个困惑：**集群装好了，然后呢？**

`kubeadm init` 执行成功只是万里长征第一步。此时的集群就像一个"毛坯房"——没有网络（Pod 之间不能通信）、没有 DNS（不能用服务名互相访问）、没有监控（不知道集群是否健康）、没有日志收集（出了问题无从查起）。这些能力都需要通过 **Addons（插件）** 来补齐。

Kubernetes 官方在 [Installing Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/) 页面罗列了各类插件项目，但只是简单列了个名字和链接，没有告诉你**该选哪个、哪个已经死了、哪个适合你的场景**。

本文基于官方 Addons 页面，结合实际生产经验，将所有主流 Addon 按功能分类，用**通俗解读**的方式说明每个项目"解决什么痛点"，并给出热度、上手难度、是否适合上生产、是否还在维护等关键信息，帮助你快速建立对 K8s 插件生态的全局认知。

> 📌 **阅读建议**：不需要全部看完。先看完"总览分层图"，再根据你的实际需求跳转到对应章节即可。

---

## 总览：按"必选程度"分层

```
┌─────────────────────────────────────────────────────────────┐
│  🔴 必装（不装集群跑不起来）                                    │
│     CNI 网络插件 + CoreDNS                                   │
├─────────────────────────────────────────────────────────────┤
│  🟠 强烈建议（不装影响日常运维）                                │
│     Metrics Server + Ingress Controller + cert-manager       │
├─────────────────────────────────────────────────────────────┤
│  🟡 生产标配（上生产前必须配齐）                                │
│     监控(Prometheus+Grafana) + 日志(Loki/EFK)               │
│     + 安全策略(Gatekeeper/Kyverno) + GitOps(ArgoCD)         │
├─────────────────────────────────────────────────────────────┤
│  🟢 按需选配（特定场景才需要）                                  │
│     服务网格(Istio/Linkerd) + 分布式存储(Rook/Longhorn)       │
│     + MetalLB + KEDA + Falco                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 一、🌐 网络插件（CNI）—— 必选，集群的"水电煤"

> **通俗解读**：K8s 装完默认 Pod 之间**不能通信**。CNI 插件就是给每个 Pod "拉网线、分 IP、配路由"的基础设施。不装 CNI，节点永远是 `NotReady`，集群就是个空壳。这是**唯一必选**的 Addon 类别。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **Calico** | ⭐⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃 | 最主流的 CNI。像给每个 Pod 配了"IP + 路由器"，支持 NetworkPolicy 网络隔离。BGP 模式性能好，概念多、学习曲线陡 |
| **Cilium** | ⭐⭐⭐⭐⭐ | 高 | ✅ | ✅ 活跃（CNCF 毕业） | 基于 eBPF 的"瑞士军刀"，不光做网络，还自带可观测性、负载均衡、安全策略。功能最强但最复杂，CNCF 贡献量第二大项目 |
| **Flannel** | ⭐⭐⭐⭐ | 低 | ✅ | ✅ 维护中 | 最简单粗暴的 CNI，给每个节点划一段 IP，用 VXLAN 隧道打通。"能用就行"的首选，但不支持 NetworkPolicy |
| **Antrea** | ⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF 沙箱） | VMware 出品，基于 OVS 虚拟交换机，安全功能比 Flannel 强，适合已有 VMware 生态的企业 |
| **kube-ovn** | ⭐⭐⭐ | 中 | ✅ | ✅ 活跃 | 国产（灵雀云），把 SDN 概念搬进 K8s，支持固定 IP、VLAN、子网划分。传统网络工程师上手最快 |
| **Multus** | ⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF 沙箱） | 不是 CNI 本身，而是"多网卡插件"。让一个 Pod 同时挂多张网卡，适合 NFV/电信场景 |
| **ACI Containers** | ⭐⭐ | 高 | ✅ | ✅ | Cisco 专属，必须搭配 Cisco ACI 硬件。把容器网络接入 Cisco SDN 体系，大型企业专用 |

**企业选型建议**：
- 小规模/求稳：**Flannel**（最简单）或 **Calico**（功能全）
- 大规模/要安全/要可观测：**Cilium**（趋势所向）
- 传统网络团队：**kube-ovn**
- 电信/NFV 多网卡：**Multus + Calico/Cilium**

**项目链接**：

| 项目 | GitHub | 官网/文档 |
|:---|:---|:---|
| Calico | [github.com/projectcalico/calico](https://github.com/projectcalico/calico) | [docs.tigera.io/calico](https://docs.tigera.io/calico/) |
| Cilium | [github.com/cilium/cilium](https://github.com/cilium/cilium) | [cilium.io](https://cilium.io/) |
| Flannel | [github.com/flannel-io/flannel](https://github.com/flannel-io/flannel) | — |
| Antrea | [github.com/antrea-io/antrea](https://github.com/antrea-io/antrea) | [antrea.io](https://antrea.io/) |
| kube-ovn | [github.com/kubeovn/kube-ovn](https://github.com/kubeovn/kube-ovn) | [kubeovn.github.io](https://kubeovn.github.io/) |
| Multus | [github.com/k8snetworkplumbingwg/multus-cni](https://github.com/k8snetworkplumbingwg/multus-cni) | — |
| ACI | [github.com/noironetworks/aci-containers](https://github.com/noironetworks/aci-containers) | — |

---

## 二、📡 DNS 服务发现 —— 必选

> **通俗解读**：让 Pod 能通过"名字"找到 Service，而不是记一串 IP。就像你打电话不用记号码，直接说"帮我转客服部"一样。没有它，集群内所有基于名称的服务调用全部瘫痪。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **CoreDNS** | ⭐⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃（CNCF 毕业） | K8s 唯一官方 DNS，kubeadm 默认装它。Pod 里访问 `my-svc.default.svc.cluster.local` 就是它在解析。插件化架构，扩展性强 |

> ⚠️ 以前的 **kube-dns** 已被 CoreDNS 完全取代，新项目不要再考虑。

**项目链接**：

| 项目 | GitHub | 官网 |
|:---|:---|:---|
| CoreDNS | [github.com/coredns/coredns](https://github.com/coredns/coredns) | [coredns.io](https://coredns.io/) |

---

## 三、📊 容器资源监控 —— 强烈建议

> **通俗解读**：告诉你每个 Pod/节点用了多少 CPU 和内存。没有它，`kubectl top` 命令直接报错，HPA 自动扩缩容无法工作。相当于给集群装了个"仪表盘"。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **Metrics Server** | ⭐⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃（K8s SIG） | 轻量级指标采集器，只保留最近几分钟数据。是 `kubectl top` 和 HPA 的唯一数据源。**几乎每个集群都装**，5 分钟部署完 |
| **cAdvisor** | ⭐⭐⭐⭐ | — | ✅ | ✅ 内置 | 已集成进 kubelet，不需要单独装。采集每个容器的 CPU/内存/IO/网络原始数据 |

**项目链接**：

| 项目 | GitHub |
|:---|:---|
| Metrics Server | [github.com/kubernetes-sigs/metrics-server](https://github.com/kubernetes-sigs/metrics-server) |

---

## 四、📋 集群级日志收集 —— 生产必备

> **通俗解读**：容器挂了、重启了，日志就没了。日志收集系统把日志"搬出来"集中存储，方便事后查问题、做审计。相当于给集群装了"行车记录仪"。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **Fluent Bit** | ⭐⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃（CNCF 毕业） | 超轻量日志采集 Agent（C 语言写的，内存占用几 MB），装在节点上把容器日志转发走。**小集群首选** |
| **Fluentd** | ⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF 毕业） | Ruby 写的日志处理引擎，插件生态极丰富，适合复杂日志管道。比 Fluent Bit 重 |
| **Loki + Promtail** | ⭐⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃 | Grafana 家的日志方案，"日志界的 Prometheus"。不建全文索引，只存标签，省资源。搭配 Grafana 查看体验极好 |
| **EFK (Elasticsearch + Fluentd + Kibana)** | ⭐⭐⭐⭐ | 高 | ✅ | ✅ | 传统方案：ES 存储搜索 + Kibana 可视化 + Fluentd 采集。功能强大但 ES 吃内存（至少 4G），运维成本高 |
| **Vector** | ⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃 | Datadog 开源，Rust 写的，性能极好，正在快速替代 Fluentd。支持日志+指标统一采集 |

**企业选型建议**：
- 小集群/资源紧张：**Fluent Bit + Loki**
- 大集群/需要全文搜索：**EFK/ELK**
- 已有 Grafana 体系：**Loki**
- 新项目推荐：**Vector**（趋势）

**项目链接**：

| 项目 | GitHub | 官网 |
|:---|:---|:---|
| Fluent Bit | [github.com/fluent/fluent-bit](https://github.com/fluent/fluent-bit) | [fluentbit.io](https://fluentbit.io/) |
| Fluentd | [github.com/fluent/fluentd](https://github.com/fluent/fluentd) | [fluentd.org](https://www.fluentd.org/) |
| Loki | [github.com/grafana/loki](https://github.com/grafana/loki) | [grafana.com/oss/loki](https://grafana.com/oss/loki/) |
| Vector | [github.com/vectordotdev/vector](https://github.com/vectordotdev/vector) | [vector.dev](https://vector.dev/) |

---

## 五、🖥️ 可视化 & 控制面板 —— 可选但方便

> **通俗解读**：给不习惯敲命令行的人提供一个 Web 界面看集群状态。就像给服务器装了个"图形化遥控器"。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **Kubernetes Dashboard** | ⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃（K8s SIG） | K8s 官方 Web UI，能看 Pod、Service、日志，能简单操作。功能有限但够用 |
| **Rancher** | ⭐⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃（SUSE） | 多集群管理平台，相当于"K8s 的图形化管理台"。能管多个集群、权限、应用商店 |
| **KubeSphere** | ⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃（国产） | 青云出品，国产 K8s 管理平台，界面友好，集成 DevOps/微服务/监控，中文文档好 |
| **Headlamp** | ⭐⭐⭐ | 低 | ✅ | ✅ 活跃（CNCF 沙箱） | 开源免费的 Dashboard 替代品，UI 比官方好看，支持插件扩展 |
| **Lens** | ⭐⭐⭐⭐ | 低 | ✅ | ⚠️ 部分商业化 | 桌面客户端（装电脑上），连接集群直接看/操作，开发调试神器 |

**项目链接**：

| 项目 | GitHub | 官网 |
|:---|:---|:---|
| Kubernetes Dashboard | [github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard) | — |
| Rancher | [github.com/rancher/rancher](https://github.com/rancher/rancher) | [rancher.com](https://www.rancher.com/) |
| KubeSphere | [github.com/kubesphere/kubesphere](https://github.com/kubesphere/kubesphere) | [kubesphere.io](https://kubesphere.io/) |
| Headlamp | [github.com/headlamp-k8s/headlamp](https://github.com/headlamp-k8s/headlamp) | [headlamp.dev](https://headlamp.dev/) |
| Lens | [github.com/lensapp/lens](https://github.com/lensapp/lens) | [k8slens.dev](https://k8slens.dev/) |

---

## 六、🚪 Ingress / API 网关 —— 生产必备

> **通俗解读**：外部流量怎么进入集群？Ingress 就是"集群大门口的保安+指路人"，根据 URL 路径把请求分发给不同的 Service。比如 `/api` 转给后端服务，`/web` 转给前端服务。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **Nginx Ingress Controller** | ⭐⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃（K8s SIG） | 最经典的 Ingress 实现，基于 Nginx 反向代理。配置简单，文档多，90% 的企业在用 |
| **Traefik** | ⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃 | 自动发现 Service、自动配置路由，"零配置"体验好。自带 Dashboard |
| **HAProxy Ingress** | ⭐⭐⭐ | 中 | ✅ | ✅ 活跃 | 性能极强（HAProxy 底子），适合高并发场景，配置比 Nginx 复杂 |
| **Kong Ingress** | ⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃 | API 网关出身，除了路由还能做限流、认证、插件扩展 |
| **APISIX Ingress** | ⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF） | Apache 项目，国产主导，性能极强，插件丰富，社区活跃 |
| **Envoy Gateway** | ⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF） | 基于 Envoy + Gateway API 新标准，代表未来方向，生态还在成长 |

> ⚠️ 注意：Ingress Controller ≠ kube-proxy。kube-proxy 处理集群**内部** Service 流量，Ingress 处理**外部进入集群**的 HTTP/HTTPS 流量。

**项目链接**：

| 项目 | GitHub | 官网 |
|:---|:---|:---|
| Nginx Ingress | [github.com/kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx) | — |
| Traefik | [github.com/traefik/traefik](https://github.com/traefik/traefik) | [traefik.io](https://traefik.io/) |
| HAProxy Ingress | [github.com/jcmoraisjr/haproxy-ingress](https://github.com/jcmoraisjr/haproxy-ingress) | — |
| Kong | [github.com/Kong/kubernetes-ingress-controller](https://github.com/Kong/kubernetes-ingress-controller) | [konghq.com](https://konghq.com/) |
| APISIX | [github.com/apache/apisix](https://github.com/apache/apisix) | [apisix.apache.org](https://apisix.apache.org/) |
| Envoy Gateway | [github.com/envoyproxy/gateway](https://github.com/envoyproxy/gateway) | [gateway.envoyproxy.io](https://gateway.envoyproxy.io/) |

---

## 七、🔒 服务网格（Service Mesh）—— 高阶可选

> **通俗解读**：比 Ingress 更细粒度的"微服务间通信管控"。给每个 Pod 旁边塞个"代理保镖"（Sidecar），所有进出流量都经过它，从而实现熔断、重试、加密、链路追踪。适合**微服务数量多、调用关系复杂**的场景。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **Istio** | ⭐⭐⭐⭐⭐ | 🔴 高 | ✅ | ✅ 活跃（CNCF 毕业） | 服务网格的"老大"，功能最全但最复杂。适合大型微服务架构（几百个服务） |
| **Linkerd** | ⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF 毕业） | "轻量版 Istio"，Rust 写的代理，资源占用小，上手快。适合中小规模 |
| **Cilium Service Mesh** | ⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃 | Cilium 自带的网格能力（eBPF 实现），不用注入 Sidecar，性能损耗小 |

> 💡 **企业建议**：微服务少于 20 个，**不建议**上服务网格，复杂度收益比不划算。

**项目链接**：

| 项目 | GitHub | 官网 |
|:---|:---|:---|
| Istio | [github.com/istio/istio](https://github.com/istio/istio) | [istio.io](https://istio.io/) |
| Linkerd | [github.com/linkerd/linkerd2](https://github.com/linkerd/linkerd2) | [linkerd.io](https://linkerd.io/) |
| Cilium Service Mesh | [github.com/cilium/cilium](https://github.com/cilium/cilium) | [cilium.io](https://cilium.io/) |

---

## 八、💾 存储 —— 按需

> **通俗解读**：Pod 重启数据就没了。存储 Addon 让 Pod 能挂载持久化磁盘/分布式存储，数据不会因容器重启而丢失。相当于给容器配了块"外接硬盘"。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **Rook (Ceph)** | ⭐⭐⭐⭐ | 🔴 高 | ✅ | ✅ 活跃（CNCF 毕业） | 用 K8s 管理 Ceph 分布式存储。块/文件/对象存储都支持，但运维极复杂，至少 3 节点 |
| **Longhorn** | ⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃（CNCF 沙箱） | Rancher 出品的轻量分布式块存储，自带 Web UI，部署简单。**小集群首选** |
| **OpenEBS** | ⭐⭐⭐ | 中 | ✅ | ✅ 活跃 | 基于容器本身的存储引擎，适合开发测试环境 |
| **NFS Client Provisioner** | ⭐⭐⭐⭐ | 低 | ✅ | ✅ | 不是存储本身，而是让 K8s 自动从 NFS 服务器创建 PV。配置极简，适合已有 NAS 的环境 |

**项目链接**：

| 项目 | GitHub | 官网 |
|:---|:---|:---|
| Rook | [github.com/rook/rook](https://github.com/rook/rook) | [rook.io](https://rook.io/) |
| Longhorn | [github.com/longhorn/longhorn](https://github.com/longhorn/longhorn) | [longhorn.io](https://longhorn.io/) |
| OpenEBS | [github.com/openebs/openebs](https://github.com/openebs/openebs) | [openebs.io](https://openebs.io/) |
| NFS Provisioner | [github.com/kubernetes-sigs/nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) | — |

---

## 九、🔐 安全 & 策略 —— 生产加固

> **通俗解读**：K8s 默认安全策略很宽松，这些工具帮你"上锁"。防止有人部署违规镜像、运行特权容器、或者容器被入侵后横向扩散。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **OPA / Gatekeeper** | ⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF 毕业） | 策略引擎。比如"禁止使用 latest 标签""禁止特权容器"，不合规的 Pod 直接拒绝创建 |
| **Kyverno** | ⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃（CNCF 毕业） | 用 YAML 写安全策略（不用学 Rego 语言），比 OPA 上手快，功能对标 |
| **Falco** | ⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF 毕业） | 运行时安全检测。比如"容器里突然执行了 bash""某个 Pod 在偷偷挖矿"，Falco 会报警 |
| **cert-manager** | ⭐⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF） | 自动申请、续期 TLS 证书。配 Ingress HTTPS 必备，不用手动管证书过期 |
| **Trivy** | ⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃 | 镜像漏洞扫描。部署前扫描镜像有没有 CVE 漏洞，CI/CD 流水线集成 |

**项目链接**：

| 项目 | GitHub | 官网 |
|:---|:---|:---|
| Gatekeeper | [github.com/open-policy-agent/gatekeeper](https://github.com/open-policy-agent/gatekeeper) | [open-policy-agent.github.io/gatekeeper](https://open-policy-agent.github.io/gatekeeper/) |
| Kyverno | [github.com/kyverno/kyverno](https://github.com/kyverno/kyverno) | [kyverno.io](https://kyverno.io/) |
| Falco | [github.com/falcosecurity/falco](https://github.com/falcosecurity/falco) | [falco.org](https://falco.org/) |
| cert-manager | [github.com/cert-manager/cert-manager](https://github.com/cert-manager/cert-manager) | [cert-manager.io](https://cert-manager.io/) |
| Trivy | [github.com/aquasecurity/trivy](https://github.com/aquasecurity/trivy) | [trivy.dev](https://trivy.dev/) |

---

## 十、📈 监控告警（完整方案）—— 生产必备

> **通俗解读**：Metrics Server 只给"快照"（最近几分钟的 CPU/内存），真正的监控需要时序数据库 + 告警 + 可视化。相当于从"看一眼仪表盘"升级为"24 小时监控室 + 自动报警"。

| 项目 | 热度 | 难度 | 生产可用 | 维护状态 | 通俗解读 |
|:---|:---:|:---:|:---:|:---:|:---|
| **Prometheus** | ⭐⭐⭐⭐⭐ | 中 | ✅ | ✅ 活跃（CNCF 毕业） | 监控界的"事实标准"。抓取所有指标存时序数据库，PromQL 查询。几乎所有公司都在用 |
| **Grafana** | ⭐⭐⭐⭐⭐ | 低 | ✅ | ✅ 活跃 | Prometheus 的"画板"，把指标画成仪表盘。配合 Loki 还能看日志 |
| **Alertmanager** | ⭐⭐⭐⭐⭐ | 低 | ✅ | ✅ | Prometheus 的"报警器"，CPU 超 90% 就发钉钉/邮件/短信/电话 |
| **kube-prometheus-stack** | ⭐⭐⭐⭐⭐ | 中 | ✅ | ✅ | 上面三件套 + 预置监控面板 + 告警规则，Helm 一键装完，**生产标配** |

**项目链接**：

| 项目 | GitHub | 官网 |
|:---|:---|:---|
| Prometheus | [github.com/prometheus/prometheus](https://github.com/prometheus/prometheus) | [prometheus.io](https://prometheus.io/) |
| Grafana | [github.com/grafana/grafana](https://github.com/grafana/grafana) | [grafana.com](https://grafana.com/) |
| Alertmanager | [github.com/prometheus/alertmanager](https://github.com/prometheus/alertmanager) | — |
| kube-prometheus-stack | [github.com/prometheus-community/helm-charts](https://github.com/prometheus-community/helm-charts) | — |

---

## 十一、🔄 其他实用 Addons

| 项目 | 类别 | 热度 | 通俗解读 | GitHub | 官网 |
|:---|:---|:---:|:---|:---|:---|
| **MetalLB** | 负载均衡 | ⭐⭐⭐⭐ | 裸金属环境给 LoadBalancer Service 分配真实 IP。云上不需要（云有 ELB）。单节点场景收益有限，多节点裸金属环境刚需 | [github.com/metallb/metallb](https://github.com/metallb/metallb) | [metallb.io](https://metallb.io/) |
| **Helm** | 包管理 | ⭐⭐⭐⭐⭐ | K8s 的"apt/yum"，一条命令装复杂应用。不是集群内 Addon，但是部署 Addon 的核心工具 | [github.com/helm/helm](https://github.com/helm/helm) | [helm.sh](https://helm.sh/) |
| **ArgoCD** | GitOps | ⭐⭐⭐⭐⭐ | 把 Git 仓库当"真相来源"，代码提交自动同步到集群。生产环境 CI/CD 标配 | [github.com/argoproj/argo-cd](https://github.com/argoproj/argo-cd) | [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/) |
| **Flux** | GitOps | ⭐⭐⭐⭐ | 和 ArgoCD 同赛道，CNCF 毕业项目，更偏声明式，轻量 | [github.com/fluxcd/flux2](https://github.com/fluxcd/flux2) | [fluxcd.io](https://fluxcd.io/) |
| **KEDA** | 自动扩缩 | ⭐⭐⭐⭐ | 基于事件驱动的 HPA 增强版，支持按 Kafka 消息数、队列深度扩缩 Pod | [github.com/kedacore/keda](https://github.com/kedacore/keda) | [keda.sh](https://keda.sh/) |
| **Descheduler** | 调度优化 | ⭐⭐⭐ | 定期把分布不均的 Pod 重新调度，让资源利用更均衡 | [github.com/kubernetes-sigs/descheduler](https://github.com/kubernetes-sigs/descheduler) | — |

---

## 🪦 已废弃 / 停更项目黑名单（不要碰）

以下项目已停止维护或归档，**新项目请勿选用**。如果你在旧系统中遇到它们，建议规划迁移。

| 项目 | 原属类别 | 废弃原因 |
|:---|:---|:---|
| **kube-dns** | DNS | 已被 CoreDNS 完全替代，K8s 官方不再推荐 |
| **Heapster** | 监控 | 已被 Metrics Server 替代，项目归档 |
| **Contiv** | CNI 网络 | Cisco 已停止维护，代码无人更新 |
| **Romana** | CNI 网络 | 项目已归档，社区消亡 |
| **Weave Net** | CNI 网络 | Weaveworks 公司已倒闭，项目无人维护 |
| **kube-router** | CNI 网络 | 已移入 CNCF Archive，不再发展 |
| **CNI-Genie** | CNI 网络 | 华为出品，更新极低，社区几乎无活动 |
| **Tiller (Helm v2)** | 包管理 | Helm v3 已移除 Tiller 组件，架构完全不同 |

> ⚠️ 如果你在网上看到教程推荐上述项目，大概率是**过时内容**。请以本文列出的活跃项目为准。

---

## 📝 写在最后

Kubernetes 的 Addon 生态非常庞大，但**不要贪多**。装得越多，运维负担越重，出问题排查面越广。

**最小生产集群推荐清单**（6 件套）：

```
CNI（Calico/Cilium/Flannel）
+ CoreDNS
+ Metrics Server
+ Ingress Controller（Nginx）
+ Prometheus + Grafana（kube-prometheus-stack）
+ 日志方案（Loki 或 Fluent Bit）
```

其余 Addon 根据业务需要逐步添加。先跑起来，再慢慢完善。

---

*本文基于 [Kubernetes 官方 Addons 页面](https://kubernetes.io/docs/concepts/cluster-administration/addons/) 整理，结合社区活跃度和生产实践编写。如有遗漏或信息过时，欢迎评论区指正。*
