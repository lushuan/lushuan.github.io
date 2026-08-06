---
title: "CRI-Dockerd 完全指南：Docker 与 Kubernetes 的“分手”与“和解”"
subtitle: ""
date: 2024-04-19T16:25:58+08:00
lastmod: 2024-06-26T16:25:58+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Docker"]
tags: ["Docker","CRI-Dockerd"]
---

### 前言

在 Kubernetes v1.24 正式移除 dockershim 之后，整个容器生态经历了一次重大的架构调整。许多运维工程师在面对“K8s 不再原生支持 Docker”这一事实时，产生了大量的困惑与焦虑。CRI-Dockerd 正是在这一历史转折点诞生的关键组件。

本文将从背景、原理、部署决策及生产实践四个维度，彻底讲透 CRI-Dockerd 的前世今生。

---

### 一、 历史背景：为什么需要 CRI-Dockerd？

要理解 CRI-Dockerd，必须先理清 Kubernetes 容器运行时的演进史：

1.  **早期（v1.0 - v1.5）**：K8s 硬编码绑定 Docker。Docker 是唯一的容器运行时。
2.  **CRI 时代（v1.5+）**：引入容器运行时接口（Container Runtime Interface），K8s 开始解耦运行时。但为了兼容存量集群，K8s 内置了一个叫 **dockershim** 的适配层，将 CRI 请求翻译成 Docker API。
3.  **弃用声明（v1.20）**：官方宣布 dockershim 将在 v1.24 移除。社区震动，“Docker 被 K8s 抛弃”的标题党文章满天飞。
4.  **正式移除（v1.24）**：kubelet 中不再有 dockershim 代码。**此时 kubelet 无法直接与 Docker Engine 通信**。
5.  **Mirantis 接手**：原 dockershim 的核心开发者加入 Mirantis，将 dockershim 从 kubelet 中剥离，重构为一个独立的二进制服务——**CRI-Dockerd**。

> 💡 **核心认知**：Docker 从未被 K8s “封杀”，只是 K8s 不再愿意在自己的代码库里维护一个仅服务于 Docker 的适配器了。CRI-Dockerd 就是这个适配器的“独立发行版”。

---

### 二、 CRI-Dockerd 是什么？它算“垫片”吗？

**是的，它本质上就是一个协议转换垫片（Shim / Adapter）。**

#### 工作原理

```
┌─────────────┐     gRPC (CRI)      ┌──────────────────┐    REST API     ┌──────────────┐
│   kubelet   │ ◄──────────────────► │  cri-dockerd     │ ◄────────────► │ Docker Engine│
│             │                      │  (独立进程/服务)   │                │  (dockerd)   │
└─────────────┘                      └──────────────────┘                └──────────────┘
                                            │
                                     Unix Socket
                                  /var/run/cri-dockerd.sock
```

-   **对 kubelet 而言**：CRI-Dockerd 是一个标准的 CRI 实现，与 containerd、CRI-O 没有区别。
-   **对 Docker Engine 而言**：CRI-Dockerd 只是一个普通的 Docker API 客户端，和 `docker` CLI 一样调用 dockerd。
-   **对 Pod 而言**：完全无感知，容器依然由 Docker 创建和管理。

#### 它的三个核心作用

| 作用 | 说明 |
| :--- | :--- |
| **协议翻译** | 将 kubelet 发出的 CRI gRPC 请求转换为 Docker Engine 的 RESTful API 调用 |
| **生命周期管理** | 作为 systemd 服务运行，负责维持与 dockerd 的连接、健康检查、日志记录 |
| **网络集成** | 处理 CNI 插件与 Docker 网络驱动之间的对接，确保 Pod 网络正常创建 |

---

### 三、 灵魂拷问：现在部署 K8s，还需要 CRI-Dockerd 吗？

这是最关键的问题。答案取决于你的**具体场景**：

#### ✅ 必须使用 CRI-Dockerd 的场景

| 场景 | 原因 |
| :--- | :--- |
| 老集群从 v1.23 升级到 v1.24+ | 已有大量基于 Docker 的运维脚本、监控探针、日志采集配置，迁移成本极高 |
| 团队 Docker 技能栈深厚，短期内无法转型 | 降低学习曲线，保持运维一致性 |
| 依赖 Docker 特有功能 | 如 Docker BuildKit、特定 storage driver、Docker plugin 等 |
| 混合环境统一管理 | 开发机用 Docker，生产 K8s 也用 Docker，减少心智负担 |

#### ❌ 不需要（也不推荐）使用的场景

| 场景 | 推荐方案 | 原因 |
| :--- | :--- | :--- |
| **全新集群部署** | containerd | 少一层翻译，性能更好，资源占用更低 |
| 追求极致轻量级 | CRI-O | 专为 K8s 设计，无多余功能 |
| 边缘计算 / IoT | containerd + k3s | 资源受限环境下多一层 shim 就是浪费 |
| 已通过 CNCF 合规认证要求 | containerd / CRI-O | CRI-Dockerd 不是 CNCF 毕业项目 |

#### ⚠️ 如果当前环境还想用 Docker，必须装 CRI-Dockerd 吗？

**是的，必须。** 在 K8s ≥ v1.24 的环境中，kubelet 与 Docker Engine 之间**没有任何直连通道**。不装 CRI-Dockerd，节点将无法创建任何 Pod。

> 📌 **重要区分**：“用 Docker 构建镜像”和“用 Docker 作为 K8s 运行时”是两件事。你完全可以**用 Docker 构建镜像 → 推送到 Harbor → K8s 用 containerd 拉取运行**，这种组合下不需要 CRI-Dockerd。只有当你坚持让 kubelet 通过 Docker 来运行容器时，才需要它。

---

### 四、 版本兼容矩阵与使用范围

#### 版本对应关系

| CRI-Dockerd 版本 | 支持的 K8s 版本 | 支持的 Docker 版本 | 备注 |
| :--- | :--- | :--- | :--- |
| v0.3.x | v1.24 - v1.27 | 20.10.x / 23.x / 24.x | 首个稳定系列 |
| v0.3.8+ | v1.28 | 24.x / 25.x | 修复多个网络相关 bug |
| v0.3.10+ | v1.29 - v1.30 | 25.x / 26.x | 支持 cgroup v2 完善 |
| v0.3.14+ | v1.31+ | 27.x | 持续跟进上游 |

> ⚠️ **版本匹配原则**：CRI-Dockerd 的版本必须与 K8s minor 版本对齐。不要在高版本 K8s 上使用低版本 CRI-Dockerd，反之亦然。每次升级 K8s 前，务必查阅 [CRI-Dockerd Release Notes](https://github.com/Mirantis/cri-dockerd/releases)。

#### 适用操作系统

-   Linux（主流发行版均支持）
-   Windows Server 2019/2022（实验性支持）
-   **国产 OS**：Kylin V10、UOS、Anolis 等均可正常使用（本质是标准 Linux binary）

---

### 五、 扩展知识点：生产环境避坑指南

#### 1. CRI-Dockerd vs Containerd 性能对比

| 指标 | containerd | CRI-Dockerd | 差异原因 |
| :--- | :--- | :--- | :--- |
| Pod 启动延迟 | ~1.2s | ~1.8s | 多一层 REST API 序列化/反序列化 |
| 内存占用 | ~30MB | ~80MB | dockerd 本身 + shim 进程开销 |
| CPU 开销 | 基准 | +5~10% | 协议转换开销 |
| 镜像拉取速度 | 基准 | 相当 | 底层都用 containerd 的 pull 机制 |

> 对于大多数业务场景，这个性能差距可以忽略。但在大规模集群（500+ 节点）或高频调度场景下，containerd 的优势会累积放大。

#### 2. 常见故障排查路径

```bash
# 1. 确认 cri-dockerd 服务状态
systemctl status cri-dockerd

# 2. 确认 socket 文件存在且权限正确
ls -la /var/run/cri-dockerd.sock

# 3. 确认 kubelet 指向正确的 CRI endpoint
crictl --runtime-endpoint unix:///var/run/cri-dockerd.sock pods

# 4. 查看 cri-dockerd 日志（排障第一手资料）
journalctl -u cri-dockerd -f --no-pager

# 5. 验证 Docker Engine 本身是否正常
docker info && docker ps
```

#### 3. 从 Docker 迁移到 containerd 的渐进策略

如果你计划最终摆脱 CRI-Dockerd，建议分三步走：

1.  **Phase 1**：新节点直接用 containerd，老节点保持 Docker + CRI-Dockerd，混合运行验证
2.  **Phase 2**：逐批驱逐老节点上的 Pod → 重装为 containerd → 重新加入集群
3.  **Phase 3**：全部迁移完成后，卸载 Docker Engine 和 CRI-Dockerd，清理残留配置

#### 4. 安全加固注意事项

-   CRI-Dockerd 的 socket 文件默认权限为 `0660`，属组为 `docker`，**不要改为 0777**
-   建议使用 systemd 的 `ProtectSystem=strict` 限制其文件系统访问
-   定期更新 CRI-Dockerd，Mirantis 会持续修复安全漏洞（CVE 响应速度较快）

---

### 六、 总结

| 问题 | 结论 |
| :--- | :--- |
| CRI-Dockerd 是什么？ | Docker 与 K8s 之间的 CRI 协议转换垫片 |
| 新集群要不要用？ | **不要**，直接用 containerd |
| 老集群升级要不要用？ | **要**，否则 Pod 无法创建 |
| 想用 Docker 就必须装它吗？ | K8s ≥ 1.24 时，**是的** |
| 它是临时过渡方案吗？ | 短期看是过渡，长期看 Mirantis 仍在积极维护，可作为合法选项 |
| 未来趋势？ | containerd 是事实标准，CRI-Dockerd 服务于存量兼容 |

> 🎯 **一句话总结**：CRI-Dockerd 是 Docker 时代的体面退场机制，而非未来的入场券。新项目的正确选择永远是 containerd，但对于那些承载着历史包袱的生产集群，CRI-Dockerd 提供了最平滑、最安全的延续之路。