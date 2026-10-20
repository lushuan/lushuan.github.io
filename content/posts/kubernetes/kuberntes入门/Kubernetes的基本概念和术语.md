---
title: "Kubernetes的基本概念和术语"
subtitle: ""
date: 2021-06-13T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
slug: "k8s-basic-concepts"
featuredImage: "/images/kubernetes/cover.jpg"
summary: "系统梳理 Kubernetes 的基本概念和术语，围绕资源对象讲解 Master/Node 集群架构、Service 与 Pod、Label 与标签选择器、Deployment 等核心概念，适合入门者建立整体知识框架。"
---

Kubernetes 的基本概念和术语大多围绕资源对象展开。本文系统梳理集群类（Master、Node）、应用类（Service、Pod、Label、Deployment）等核心概念，帮助入门者快速建立对 Kubernetes 整体架构和关键对象的认知。

<!--more-->

#  资源对象概述
Kubernetes 中的基本概念和术语大多是围绕资源对象（Resource Object）来说的，而资源对象在总体上可分为以下两类。

- 某种资源的对象，例如节点（Node）、Pod、服务（Service）、存储卷（Volume）
- 与资源对象相关的事物与动作，例如标签（Label）、注解（Annotation）、命名空间（Namespace）、部署（Deployment）、HPA、PVC。

#   集群类
集群（Cluster）表示一个由 Master 和 Node 组成的 Kubernetes 集群。

## 一、Master  
Master 指的是集群的控制节点。在每个 Kubernetes 集群中都需要有一个或一组被称为 Master 的节点，来负责整个集群的管理和控制。Master 通常占据一个独立的服务器（在高可用部署中建议至少使用 3 台服务器），是整个集群的"大脑"，如果它发生宕机或者不可用，那么对集群内容器应用的管理都将无法实施。

在 Master 上运行着以下关键进程。
- Kubernetes API Server（kube-apiserver）：提供 HTTP RESTfulAPI 接口的主要服务，是 Kubernetes 里对所有资源进行增、删、改、查等操作的唯一入口，也是集群控制的入口进程。
- Kubernetes Controller Manager（kube-controller-manager）：Kubernetes 里所有资源对象的自动化控制中心，可以将其理解为资源对象的"大总管"。
- Kubernetes Scheduler（kube-scheduler）：负责资源调度（Pod 调度）的进程，相当于公交公司的调度室。

另外，在 Master 上通常还需要部署 etcd 服务

## 二、Node

Kubernetes 集群中除 Master 外的其他服务器被称为 Node，Node 在较早的版本中也被称为 Minion。与 Master 一样，Node 可以是一台物理主机，也可以是一台虚拟机。Node 是 Kubernetes 集群中的工作负载节点，每个 Node 都会被 Master 分配一些工作负载（Docker 容器），当某个 Node 宕机时，其上的工作负载会被 Master 自动转移到其他 Node 上。

在每个 Node 上都运行着以下关键进程。  
- kubelet：负责 Pod 对应容器的创建、启停等任务，同时与 Master 密切协作，实现集群管理的基本功能。
- kube-proxy：实现 Kubernetes Service 的通信与负载均衡机制的服务。
- 容器运行时（如 Docker）：负责本机的容器创建和管理。Node 可以在运行期间动态增加到 Kubernetes 集群中，前提是在这个 Node 上已正确安装、配置和启动了上述关键进程。在默认情况下，kubelet 会向 Master 注册自己，这也是 Kubernetes 推荐的 Node 管理方式。一旦 Node 被纳入集群管理范畴，kubelet 进程就会定时向 Master 汇报自身的情报，例如操作系统、主机 CPU 和内存使用情况，以及当前有哪些 Pod 在运行等，这样 Master 就可以获知每个 Node 的资源使用情况，并实现高效均衡的资源调度策略。而某个 Node 在超过指定时间不上报信息时，会被 Master 判定为"失联"，该 Node 的状态就被标记为不可用（NotReady），Master 随后会触发"工作负载大转移"的自动流程。

#  应用类
Kubernetes 中属于应用类的概念和相应的资源对象类型最多，所以应用类也是需要重点学习的一类。

## 三、 Service 与 Pod  
应用类相关的资源对象主要是围绕 Service（服务）和 Pod 这两个核心对象展开的。

一般说来，Service 指的是无状态服务，通常由多个程序副本提供服务，在特殊情况下也可以是有状态的单实例服务，比如 MySQL 这种数据存储类的服务。与我们常规理解的服务不同，Kubernetes 里的 Service 具有一个全局唯一的虚拟 ClusterIP 地址，Service 一旦被创建，Kubernetes 就会自动为它分配一个可用的 ClusterIP 地址，而且在 Service 的整个生命周期中，它的 ClusterIP 地址都不会改变，客户端可以通过这个虚拟 IP 地址+服务的端口直接访问该服务，再通过部署 Kubernetes 集群的 DNS 服务，就可以实现 Service Name（域名）到 ClusterIP 地址的 DNS 映射功能，我们只要使用服务的名称（DNS 名称）即可完成到目标服务的访问请求。"服务发现"这个传统架构中的棘手问题在这里首次得以完美解决，同时，凭借 ClusterIP 地址的独特设计，Kubernetes 进一步实现了 Service 的透明负载均衡和故障自动恢复的高级特性。

通过分析、识别并建模系统中的所有服务为微服务——Kubernetes Service，我们的系统最终由多个提供不同业务能力而又彼此独立的微服务单元组成，服务之间通过 TCP/IP 进行通信，从而形成强大又灵活的弹性网格，拥有强大的分布式能力、弹性扩展能力、容错能力，程序架构也变得简单和直观许多

## 四、Pod
为什么 Kubernetes 会设计出一个全新的 Pod 概念并且 Pod 有这样特殊的组成结构？原因如下。

- 为多进程之间的协作提供一个抽象模型，使用 Pod 作为基本的调度、复制等管理工作的最小单位，让多个应用进程能一起有效地调度和伸缩。
- Pod 里的多个业务容器共享 Pause 容器的 IP，共享 Pause 容器挂接的 Volume，这样既简化了密切关联的业务容器之间的通信问题，也很好地解决了它们之间的文件共享问题。

Kubernetes 为每个 Pod 都分配了唯一的 IP 地址，称之为 Pod IP，一个 Pod 里的多个容器共享 Pod IP 地址。Kubernetes 要求底层网络支持集群内任意两个 Pod 之间的 TCP/IP 直接通信，这通常采用虚拟二层网络技术实现，例如 Flannel、OpenvSwitch 等，因此我们需要牢记一点：在 Kubernetes 里，一个 Pod 里的容器与另外主机上的 Pod 容器能够直接通信。

Pod 其实有两种类型：普通的 Pod 及静态 Pod（Static Pod）。后者比较特殊，它并没被存放在 Kubernetes 的 etcd 中，而是被存放在某个具体的 Node 上的一个具体文件中，并且只能在此 Node 上启动、运行。而普通的 Pod 一旦被创建，就会被放入 etcd 中存储，随后被 Kubernetes Master 调度到某个具体的 Node 上并绑定（Binding），该 Pod 被对应的 Node 上的 kubelet 进程实例化成一组相关的 Docker 容器并启动。在默认情况下，当 Pod 里的某个容器停止时，Kubernetes 会自动检测到这个问题并且重新启动这个 Pod（重启 Pod 里的所有容器），如果 Pod 所在的 Node 宕机，就会将这个 Node 上的所有 Pod 都重新调度到其他节点上。

Pod、容器与 Node 的关系
![Pod、容器与Node的关系](/images/kubernetes/basic/pod-node.png "Pod、容器与Node的关系")

## 五、 Label 与标签选择器
Label（标签）是 Kubernetes 系统中的另一个核心概念，相当于我们熟悉的"标签"。一个 Label 是一个 key=value 的键值对，其中的 key 与 value 由用户自己指定。Label 可以被附加到各种资源对象上，例如 Node、Pod、Service、Deployment 等，一个资源对象可以定义任意数量的 Label，同一个 Label 也可以被添加到任意数量的资源对象上。Label 通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。我们可以通过给指定的资源对象捆绑一个或多个不同的 Label 来实现多维度的资源分组管理功能，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作，例如，部署不同版本的应用到不同的环境中，以及监控、分析应用（日志记录、监控、告警）等。一些常用的 Label 示例如下。

- 版本标签：release：stable 和 release：canary。
- 环境标签：environment：dev、environment：qa 和 environment：production。
- 架构标签：tier：frontend、tier：backend 和 tier：middleware。
- 分区标签：partition：customerA 和 partition：customerB。
- 质量管控标签：track：daily 和 track：weekly

Label 也是 Pod 的重要属性之一，其重要性仅次于 Pod 的端口，我们几乎见不到没有 Label 的 Pod

Service 很重要的一个属性就是标签选择器，如果我们不小心把标签选择器写错了，就会出现指鹿为马的闹剧。如果恰好匹
配到了另一种 Pod 实例，而且对应的容器端口恰好正确，服务可以正常连接，则很难排查问题，特别是在有众多 Service 的复杂系统中。

## 六、Pod 与 Deployment
前面提到，大部分 Service 都是无状态的服务，可以由多个 Pod 副本实例提供服务。通常情况下，每个 Service 对应的 Pod 服务实例数量都是固定的，如果一个一个地手工创建 Pod 实例，就太麻烦了，最好是用模板的思路，即提供一个 Pod 模板（Template），然后由程序根据我们指定的模板自动创建指定数量的 Pod 实例。这就是 Deployment 这个资源对象所要完成的事情了。

## 七、Service 的 ClusterIP 地址
既然每个 Pod 都会被分配一个单独的 IP 地址，而且每个 Pod 都提供了一个独立的 Endpoint（Pod IP+containerPort）以被客户端访问，那么现在多个 Pod 副本组成了一个集群来提供服务，客户端如何访问它们呢？传统的做法是部署一个负载均衡器（软件或硬件），为这组 Pod 开启一个对外的服务端口如 8000 端口，并且将这些 Pod 的 Endpoint 列表加入 8000 端口的转发列表中，客户端就可以通过负载均衡器的对外 IP 地址+8000 端口来访问此服务了。Kubernetes 也是类似的做法，Kubernetes 内部在每个 Node 上都运行了一套全局的虚拟负载均衡器，自动注入并自动实时更新集群中所有 Service 的路由表，通过 iptables 或者 IPVS 机制，把对 Service 的请求转发到其后端对应的某个 Pod 实例上，并在内部实现服务的负载均衡与会话保持机制。不仅如此，Kubernetes 还采用了一种很巧妙又影响深远的设计——ClusterIP 地址。我们知道，Pod 的 Endpoint 地址会随着 Pod 的销毁和重新创建而发生改变，因为新 Pod 的 IP 地址与之前旧 Pod 的不同。Service 一旦被创建，Kubernetes 就会自动为它分配一个全局唯一的虚拟 IP 地址——ClusterIP 地址，而且在 Service 的整个生命周期内，其 ClusterIP 地址不会发生改变，这样一来，每个服务就变成了具备唯一 IP 地址的通信节点，远程服务之间的通信问题就变成了基础的 TCP 网络通信问题。

之所以说 ClusterIP 地址是一种虚拟 IP 地址，原因有以下几点。    
- ClusterIP 地址仅仅作用于 Kubernetes Service 这个对象，并由 Kubernetes 管理和分配 IP 地址（来源于 ClusterIP 地址池），与 Node 和 Master 所在的物理网络完全无关。
- 因为没有一个"实体网络对象"来响应，所以 ClusterIP 地址无法被 Ping 通。ClusterIP 地址只能与 Service Port 组成一个具体的服务访问端点，单独的 ClusterIP 不具备 TCP/IP 通信的基础。
- ClusterIP 属于 Kubernetes 集群这个封闭的空间，集群外的节点要访问这个通信端口，则需要做一些额外的工作

## 八、Service 的外网访问问题
前面提到，服务的 ClusterIP 地址在 Kubernetes 集群内才能被访问，那么如何让集群外的应用访问我们的服务呢？这也是一个相对复杂的问题。要弄明白这个问题的解决思路和解决方法，我们需要先弄明白

Kubernetes 的三种 IP，这三种 IP 分别如下。
- Node IP：Node 的 IP 地址。
- Pod IP：Pod 的 IP 地址。
- Service IP：Service 的 IP 地址。

首先，Node IP 是 Kubernetes 集群中每个节点的物理网卡的 IP 地址，是一个真实存在的物理网络，所有属于这个网络的服务器都能通过这个网络直接通信，不管其中是否有部分节点不属于这个 Kubernetes 集群。这也表明 Kubernetes 集群之外的节点访问 Kubernetes 集群内的某个节点或者 TCP/IP 服务时，都必须通过 Node IP 通信。

其次，Pod IP 是每个 Pod 的 IP 地址，在使用 Docker 作为容器支持引擎的情况下，它是 Docker Engine 根据 docker0 网桥的 IP 地址段进行分配的，通常是一个虚拟二层网络。前面说过，Kubernetes 要求位于不同 Node 上的 Pod 都能够彼此直接通信，所以 Kubernetes 中一个 Pod 里的容器访问另外一个 Pod 里的容器时，就是通过 Pod IP 所在的虚拟二层网络进行通信的，而真实的 TCP/IP 流量是通过 Node IP 所在的物理网卡流出的。

在 Kubernetes 集群内，Service 的 ClusterIP 地址属于集群内的地址，无法在集群外直接使用这个地址。为了解决这个问题，Kubernetes 首先引入了 NodePort 这个概念，NodePort 也是解决集群外的应用访问集群内服务的直接、有效的常见做法

NodePort 的实现方式是，在 Kubernetes 集群的每个 Node 上都为需要外部访问的 Service 开启一个对应的 TCP 监听端口，外部系统只要用任意一个 Node 的 IP 地址+NodePort 端口号即可访问此服务，在任意 Node 上运行 netstat 命令，就可以看到有 NodePort 端口被监听。

## 九、NodePort 存在的一个问题引出了 Ingress 对象  

NodePort 的确功能强大且通用性强，但也存在一个问题，即每个 Service 都需要在 Node 上独占一个端口，而端口又是有限的物理资源，那能不能让多个 Service 共用一个对外端口呢？这就是后来增加的 Ingress 资源对象所要解决的问题。在一定程度上，我们可以把 Ingress 的实现机制理解为基于 Nginx 的支持虚拟主机的 HTTP 代理。

## 十、有状态的应用集群
我们知道，Deployment 对象是用来实现无状态服务的多副本自动控制功能的，那么有状态的服务，比如 ZooKeeper 集群、MySQL 高可用集群（3 节点集群）、Kafka 集群等是怎么实现自动部署和管理的呢？这个问题就复杂多了，这些一开始是依赖 StatefulSet 解决的，但后来发现对于一些复杂的有状态的集群应用来说，StatefulSet 还是不够通用和强大，所以后面又出现了 Kubernetes Operator。

我们先说说 StatefulSet。StatefulSet 之前曾用过 PetSet 这个名称，很多人都知道，在 IT 世界里，有状态的应用被类比为宠物（Pet），无状态的应用则被类比为牛羊，每个宠物在主人那里都是"唯一的存在"，宠物生病了，我们是要花很多钱去治疗的，需要我们用心照料，而无差别的牛羊则没有这个待遇。总结下来，在有状态集群中一般有如下特殊共性。

- 每个节点都有固定的身份 ID，通过这个 ID，集群中的成员可以相互发现并通信。
- 集群的规模是比较固定的，集群规模不能随意变动。
- 集群中的每个节点都是有状态的，通常会持久化数据到永久存储中，每个节点在重启后都需要使用原有的持久化数据。
- 集群中成员节点的启动顺序（以及关闭顺序）通常也是确定的。
- 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损

StatefulSet 从本质上来说，可被看作 Deployment/RC 的一个特殊变种，它有如下特性。

- StatefulSet 里的每个 Pod 都有稳定、唯一的网络标识，可以用来发现集群内的其他成员。假设 StatefulSet 的名称为 Kafka，那么第 1 个 Pod 叫 kafka-0，第 2 个叫 kafka-1，以此类推。
- StatefulSet 控制的 Pod 副本的启停顺序是受控的，操作第 n 个 Pod 时，前 n-1 个 Pod 已经是运行且准备好的状态。
- StatefulSet 里的 Pod 采用稳定的持久化存储卷，通过 PV 或 PVC 来实现，删除 Pod 时默认不会删除与 StatefulSet 相关的存储卷（为了保证数据安全）。
StatefulSet 除了要与 PV 卷捆绑使用，以存储 Pod 的状态数据，还要与 Headless Service 配合使用，即在每个 StatefulSet 定义中都要声明它属于哪个 Headless Service。StatefulSet 在 Headless Service 的基础上又为 StatefulSet 控制的每个 Pod 实例都创建了一个 DNS 域名，这个域名的格式
如下：
```shell
${podname}.${headless service name}
```

StatefulSet 的建模能力有限，面对复杂的有状态集群时显得力不从心，所以就有了后来的 Kubernetes Operator 框架和众多的 Operator 实现了。需要注意的是，Kubernetes Operator 框架并不是面向普通用户的，而是面向 Kubernetes 平台开发者的。

## 十一、批处理应用
除了无状态服务、有状态集群、常见的第三种应用，还有批处理应用。批处理应用的特点是一个或多个进程处理一组数据（图像、文件、视频等），在这组数据都处理完成后，批处理任务自动结束。为了支持这类应用，Kubernetes 引入了新的资源对象——Job。

Jobs 控制器提供了两个控制并发数的参数：completions 和 parallelism，completions 表示需要运行任务数的总数，parallelism 表示并发运行的个数，例如设置 parallelism 为 1，则会依次运行任务，在前面的任务运行后再运行后面的任务。Job 所控制的 Pod 副本是短暂运行的，可以将其视为一组容器，其中的每个容器都仅运行一次。当 Job 控制的所有 Pod 副本都运行结束时，对应的 Job 也就结束了。Job 在实现方式上与 Deployment 等副本控制器不同，Job 生成的 Pod 副本是不能自动重启的，对应 Pod 副本的 restartPolicy 都被设置为 Never，因此，当对应的 Pod 副本都执行完成时，相应的 Job 也就完成了控制使命。后来，Kubernetes 增加了 CronJob，可以周期性地执行某个任务。

## 十二、应用的配置问题

通过前面的学习，我们初步理解了三种应用建模的资源对象，总结如下。
- 无状态服务的建模：Deployment。
- 有状态集群的建模：StatefulSet。
- 批处理应用的建模：Job。
在进行应用建模时，应该如何解决应用需要在不同的环境中修改配置的问题呢？这就涉及 ConfigMap 和 Secret 两个对象。

ConfigMap 顾名思义，就是保存配置项（key=value）的一个 Map，如果你只是把它理解为编程语言中的一个 Map，那就大错特错了。ConfigMap 是分布式系统中"配置中心"的独特实现之一。我们知道，几乎所有应用都需要一个静态的配置文件来提供启动参数，当这个应用是一个分布式应用，有多个副本部署在不同的机器上时，配置文件的分发就成为一个让人头疼的问题，所以很多分布式系统都有一个配置中心组件，来解决这个问题。但配置中心通常会引入新的 API，从而导致应用的耦合和侵入。

- 用户将配置文件的内容保存到 ConfigMap 中，文件名可作为 key，value 就是整个文件的内容，多个配置文件都可被放入同一个 ConfigMap。
- 在建模用户应用时，在 Pod 里将 ConfigMap 定义为特殊的 Volume 进行挂载。在 Pod 被调度到某个具体 Node 上时，ConfigMap 里的配置文件会被自动还原到本地目录下，然后映射到 Pod 里指定的配置目录下，这样用户的程序就可以无感知地读取配置了。
- 在 ConfigMap 的内容发生修改后，Kubernetes 会自动重新获取 ConfigMap 的内容，并在目标节点上更新对应的文件。

接下来说说 Secret。Secret 也用于解决应用配置的问题，不过它解决的是对敏感信息的配置问题，比如数据库的用户名和密码、应用的数字证书、Token、SSH 密钥及其他需要保密的敏感配置。对于这类敏感信息，我们可以创建一个 Secret 对象，然后被 Pod 引用。Secret 中的数据要求以 BASE64 编码格式存放。注意，BASE64 编码并不是加密的，在 Kubernetes 1.7 版本以后，Secret 中的数据才可以以加密的形式进行保存，更加安全。

## 十三、应用的运维问题
最后说说与应用的自动运维相关的几个重要对象。

首先就是 HPA（Horizontal Pod Autoscaler），如果我们用 Deployment 来控制 Pod 的副本数量，则可以通过手工运行 kubectl scale 命令来实现 Pod 扩容或缩容。如果仅仅到此为止，则显然不符合谷歌对 Kubernetes 的定位目标——自动化、智能化。在谷歌看来，分布式系统要能够根据当前负载的变化自动触发水平扩容或缩容，因为这一过程可能是频繁发生、不可预料的，所以采用手动控制的方式是不现实的，因此就有了后来的 HPA 这个高级功能。我们可以将 HPA 理解为 Pod 横向自动扩容，即自动控制 Pod 数量的增加或减少。通过追踪分析指定 Deployment 控制的所有目标 Pod 的负载变化情况，来确定是否需要有针对性地调整目标 Pod 的副本数量，这是 HPA 的实现原理。Kubernetes 内置了基于 Pod 的 CPU 利用率进行自动扩缩容的机制，应用开发者也可以自定义度量指标如每秒请求数，来实现自定义的 HPA 功能。

#  存储类
存储类的资源对象主要包括 Volume、Persistent Volume、PVC 和 StorageClass。

首先看看基础的存储类资源对象——Volume（存储卷）

Volume 是 Pod 中能够被多个容器访问的共享目录。Kubernetes 中的 Volume 概念、用途和目的与 Docker 中的 Volume 比较类似，但二者不能等价。首先，Kubernetes 中的 Volume 被定义在 Pod 上，被一个 Pod 里的多个容器挂载到具体的文件目录下；其次，Kubernetes 中的 Volume 与 Pod 的生命周期相同，但与容器的生命周期不相关，当容器终止或者重启时，Volume 中的数据也不会丢失；最后，Kubernetes 支持多种类型的 Volume，例如 GlusterFS、Ceph 等分布式文件系统。Volume 的使用也比较简单，在大多数情况下，我们先在 Pod 上声明一个 Volume，然后在容器里引用该 Volume 并将其挂载（Mount）到容器里的某个目录下。举例来说，若我们要给之前的 Tomcat Pod 增加一个名为 datavol 的 Volume，并将其挂载到容器的某个路径/mydata-data 目录下，则只对 Pod 的定义文件做下修正即可。

## 十四、emptyDir
一个 emptyDir 是在 Pod 分配到 Node 时创建的。从它的名称就可以看出，它的初始内容为空，并且无须指定宿主机上对应的目录文件，因为这是 Kubernetes 自动分配的一个目录，当 Pod 从 Node 上移除时，emptyDir 中的数据也被永久移除。emptyDir 的一些用途如下。
- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留。
- 长时间任务执行过程中使用的临时目录。
- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）。

在默认情况下，emptyDir 使用的是节点的存储介质，例如磁盘或者网络存储。还可以使用 emptyDir.medium 属性，把这个属性设置为"Memory"，就可以使用更快的基于内存的后端存储了。需要注意的是，这种情况下的 emptyDir 使用的内存会被计入容器的内存消耗，将受到资源限制和配额机制的管理。

## 十五、hostPath
hostPath 为在 Pod 上挂载宿主机上的文件或目录，通常可以用于以下几方面。  
- 在容器应用程序生成的日志文件需要永久保存时，可以使用宿主机的高速文件系统对其进行存储。
- 需要访问宿主机上 Docker 引擎内部数据结构的容器应用时，可以通过定义hostPath为宿主机/var/lib/docker目录，使容器内部的应用可以直接访问 Docker 的文件系统。
在使用这种类型的 Volume 时，需要注意以下几点。
- 在不同的 Node 上具有相同配置的 Pod，可能会因为宿主机上的目录和文件不同，而导致对 Volume 上目录和文件的访问结果不一致。
- 如果使用了资源配额管理，则 Kubernetes 无法将 hostPath 在宿主机上使用的资源纳入管理。

## 十六、公有云 Volume
公有云提供的 Volume 类型包括谷歌公有云提供的 GCEPersistentDisk、亚马逊公有云提供的 AWS Elastic Block Store（EBSVolume）等。当我们的 Kubernetes 集群运行在公有云上或者使用公有云厂家提供的 Kubernetes 集群时，就可以使用这类 Volume。

## 十七、其他类型的 Volume
- iscsi：将 iSCSI 存储设备上的目录挂载到 Pod 中。
- NFS：将 NFS Server 上的目录挂载到 Pod 中。
- glusterfs：将开源 GlusterFS 网络文件系统的目录挂载到 Pod 中。
- rbd：将 Ceph 块设备共享存储（Rados Block Device）挂载到 Pod 中。
- gitRepo：通过挂载一个空目录，并从 Git 库克隆（clone）一个 git repository 以供 Pod 使用。
- configmap：将配置数据挂载为容器内的文件。
- secret：将 Secret 数据挂载为容器内的文件。

#  安全类

安全始终是 Kubernetes 发展过程中的一个关键领域。

从本质上来说，Kubernetes 可被看作一个多用户共享资源的资源管理系统，这里的资源主要是各种 Kubernetes 里的各类资源对象，比如 Pod、Service、Deployment 等。只有通过认证的用户才能通过 Kubernetes 的 API Server 查询、创建及维护相应的资源对象，理解这一点很关键。

Kubernetes 里的用户有两类：我们开发的运行在 Pod 里的应用；普通用户，如典型的 kubectl 命令行工具，基本上由指定的运维人员（集群管理员）使用。在更多的情况下，我们开发的 Pod 应用需要通过 API Server 查询、创建及管理其他相关资源对象，所以这类用户才是 Kubernetes 的关键用户。为此，Kubernetes 设计了 Service Account 这个特殊的资源对象，代表 Pod 应用的账号，为 Pod 提供必要的身份认证。在此基础上，Kubernetes 进一步实现和完善了基于角色的访问控制权限系统——RBAC（Role-Based Access Control）。

在默认情况下，Kubernetes 在每个命名空间中都会创建一个默认的名称为 default 的 Service Account，因此 Service Account 是不能全局使用的，只能被它所在命名空间中的 Pod 使用。通过以下命令可以查看集群中的所有 Service Account：
```shell
sudo kubectl get sa -A
```

Service Account 是通过 Secret 来保存对应的用户（应用）身份凭证的，这些凭证信息有 CA 根证书数据（ca.crt）和签名后的 Token 信息（Token）。在 Token 信息中就包括了对应的 Service Account 的名称，因此 API Server 通过接收到的 Token 信息就能确定 Service Account 的身份。在默认情况下，用户创建一个 Pod 时，Pod 会绑定对应命名空间中的 default 这个 Service Account 作为其"公民身份证"。当 Pod 里的容器被创建时，Kubernetes 会把对应的 Secret 对象中的身份信息（ca.crt、Token 等）持久化保存到容器里固定位置的本地文件中，因此当容器里的用户进程通过 Kubernetes 提供的客户端 API 去访问 API Server 时，这些 API 会自动读取这些身份信息文件，并将其附加到 HTTPS 请求中传递给 API Server 以完成身份认证逻辑。在身份认证通过以后，就涉及"访问授权"的问题，这就是 RBAC 要解决的问题了。

首先我们要学习的是 Role 这个资源对象，包括 Role 与 ClusterRole 两种类型的角色。角色定义了一组特定权限的规则，比如可以操作某类资源对象。局限于某个命名空间的角色由 Role 对象定义，作用于整个 Kubernetes 集群范围内的角色则通过 ClusterRole 对象定义。

在 RoleBinding 中使用 subjects（目标主体）来表示要授权的对象，这是因为我们可以授权三类目标账号：Group（用户组）、User（某个具体用户）和 Service Account（Pod 应用所使用的账号）。

在安全领域，除了以上针对 API Server 访问安全相关的资源对象，还有一种特殊的资源对象——NetworkPolicy（网络策略），它是网络安全相关的资源对象，用于解决用户应用之间的网络隔离和授权问题。NetworkPolicy 是一种关于 Pod 间相互通信，以及 Pod 与其他网络端点间相互通信的安全规则设定。

NetworkPolicy 资源使用标签选择 Pod，并定义选定 Pod 所允许的通信规则。在默认情况下，Pod 间及 Pod 与其他网络端点间的访问是没有限制的，这假设了 Kubernetes 集群被一个厂商（公司/租户）独占，其中部署的应用都是相互可信的，无须相互防范。但是，如果存在多个厂商共同使用一个 Kubernetes 集群的情况，则特别是在公有云环境中，不同厂商的应用要相互隔离以增加安全性，这就可以通过 NetworkPolicy 来实现了。
