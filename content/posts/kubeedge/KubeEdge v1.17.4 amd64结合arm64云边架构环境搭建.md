---
title: "KubeEdge v1.17.4 amd64结合arm64云边架构环境搭建"
subtitle: ""
date: 2025-09-10T12:06:37+08:00
lastmod: 2025-09-14T12:06:37+08:00
draft: false
toc:
enable: true
weight: false
categories: ["KubeEdge "]
tags: ["KubeEdge "]
---
## 前言
最近在做一个网络内生安全方面的一个项目，需要在物联网IOT边端设备上部署软蜜罐和蜜网，为了方便管理边端设备服务，选型华为的[KubeEdge开源方案](https://kubeedge.io/zh/),
当然使用k8s 定制版的 k3s 也可以，这里由于设备资源有限最终还是选择了KubeEdge,但是在网上找了很多资料以及官网都没有找到基于云端amd64+边端arm64架构组合的环境搭建，
基本上云端amd64+边端amd64组合的方案，更多的是一种演示。本文是基于真实采购的边缘网关搭建的一整套流程，在此做一下记录，方便后面回溯复用。

## KubeEdge 介绍
KubeEdge是一个开源系统，用于将容器化应用程序编排功能扩展到Edge的主机。它基于Kubernetes构建，并为网络应用程序提供基础架构支持、云和边缘之间的部署和元数据同步。

目标是创建一个开放平台，使能边缘计算，将容器化应用编排功能扩展到基于Kubernetes构建的边缘节点和设备，并为云和边缘之间的网络、应用部署和元数据同步提供基础架构支持。

简单的讲就是将云端和终端硬件设备在同一局域网进行了打通，并方便管理边端设备，边缘设备可以理解为RTU、边缘网关，而这些边缘网关又串联了支持485、232等协议的设备，例如电扇、空调、温湿度变送器都可以接入到边缘网关进行管理，而云端又管理了边端设备，间接的管理了各种终端设备。

优势
1. k8s 原生支持，也就是说需要提前搭建一套k8s 集群，无需高可用，生产建议高可用
2. 边端设备管理：通过 Kubernetes 的原生API，并由CRD来管理边缘设备
3. 极致轻量的边缘代理：在资源有限的边缘端上运行的非常轻量级的边缘代理(EdgeCore)，边端服务使用的内存占用降至约 70MB
4. 异构性：支持x86、ARMv7、ARMv8，这个个人认为是最重要的，因为物联网边端设备更多使用的是arm架构，并且还会对操作系统进行相应的裁剪，来保障安全和低耗能，尤其是一些物联网RTOS设备

- **一句话介绍**：把 Kubernetes 的能力“下沉”到边缘，让边缘节点（IoT 设备、工控机、边缘服务器等）也能像云端节点一样被统一调度、管理。

### 核心组件
| 云端                                         | 边端                                                     |
| ------------------------------------------ | ------------------------------------------------------ |
| **CloudCore**：负责与 kube-apiserver 对接、下发期望状态 | **EdgeCore**：运行在边缘节点，接收/缓存任务、控制容器、同步状态                 |
| deviceController、edgeController、cloudhub   | edged、edgeHub、metaManager、eventBus、serviceBus、edgeMesh |
| 主要跑在数据中心/公有云                               | 主要跑在工厂、门店、机房、摄像头等设备上                                   |

### KubeEdge解决的核心痛点

| 场景痛点             | KubeEdge 提供的能力                            |
| ---------------- | ----------------------------------------- |
| **海量边缘节点管理难**    | 通过 Kubernetes CRD，把边缘节点纳入统一调度视图           |
| **网络不稳定**（断网、弱网） | 本地缓存 + 自主调度：断网也能继续运行，重连后同步状态              |
| **应用下沉到边缘**      | 支持在边缘节点直接运行 Pod、Deployment、DaemonSet      |
| **IoT 设备接入繁琐**   | 内置 Device CRD & eventBus，设备属性、状态上云        |
| **云边通信安全**       | 双向 TLS、MQTT over WebSocket 等安全通道          |
| **流量本地化/低延迟**    | edgeMesh 提供边-边、边-云的 service mesh，本地流量不必回云 |

### 典型使用场景

1. **智慧园区/楼宇**：门禁、监控、告警在本地实时处理，云端只做大数据分析
2. **工业 IoT**：产线传感器、PLC 数据采集与控制，边缘计算降低时延
3. **零售连锁**：门店收银、广告推送、客流分析本地化，降低云成本
4. **视频边缘分析**：摄像头实时推理、智能告警，减少视频上云带宽
5. **智能交通**：路口红绿灯、车流监控在路侧计算，中心调度策略


### 优势/价值

* **兼容原生 Kubernetes API**：开发/运维门槛低
* **云边解耦**：网络断开时边缘工作负载不受影响
* **硬件友好**：支持 **amd64 / arm32 / arm64 / risc-v** 等多种 CPU 架构
* **IoT 能力**：原生支持设备管理、MQTT 消息总线
* **社区活跃**：CNCF 沙箱 → 孵化项目，有较多工业界落地案例

### 总结

* **一句话价值**：KubeEdge = “Kubernetes + IoT + 边缘自治”。
* **适用行业**：制造、零售、安防、能源、交通、智能家居等所有需要**就近计算、降低云依赖、保障实时性**的场景。
* **混合架构**：同时支持 amd64 与 arm 混部，方便企业“中心机房 + 边缘节点”一体化运维。

## KubeEdge 部署架构
### 1. 典型部署架构

```
┌───────────────────────────────────────────┐
│                Cloud (数据中心)          │
│                                           │
│  ┌───────────────┐       ┌─────────────┐ │
│  │  Kubernetes   │       │  CloudCore  │ │
│  │  API Server   │<----->│  (KubeEdge) │ │
│  └───────────────┘       └─────────────┘ │
│                                           │
└───────────────────────────────────────────┘
                ||  WebSocket/QUIC/MQTT
                ||  双向同步/指令下发
┌───────────────────────────────────────────┐
│                Edge (工厂/分支)           │
│                                           │
│  ┌───────────────┐       ┌─────────────┐ │
│  │  EdgeCore     │<----->│  Mosquitto  │ │(可选)
│  │  (KubeEdge)   │       │  DeviceTwin │ │
│  └───────────────┘       └─────────────┘ │
│     │  Pod Sandbox / CRI                   │
│     │  应用容器（Pod）                      │
└───────────────────────────────────────────┘
```

* **CloudCore**

    * 部署在云端集群（一般与 kube-apiserver、controller-manager、scheduler 同机或同网段）。
    * 负责：

        * 同步 Kubernetes 对象到边缘
        * 接收边缘上报状态、事件
        * 下发指令、配置
        * 提供设备管理 API（Device CRD）

* **EdgeCore**

    * 部署在边缘节点（工厂、店面、站点）。
    * 组件：

        * **EdgeHub**：负责与 CloudCore 建立长连接（WebSocket/QUIC）。
        * **Edged**：轻量 kubelet，负责 Pod 生命周期。
        * **EventBus**：通过 MQTT/内存总线收集、分发消息。
        * **DeviceTwin**：维护设备状态、影子。
        * **MetaManager**：对象缓存、断网自治。

* **通信机制**

    * Cloud ↔ Edge 长连接（默认 WebSocket，可选 QUIC/MQTT）
    * 支持对象同步（Pod/ConfigMap/Secrets 等）与事件上报。

---

### 2. 部署特征

| 角色        | 运行位置             | 核心进程      | 依赖                         |
| --------- | ---------------- | --------- | -------------------------- |
| CloudCore | 云端 Kubernetes 集群 | CloudCore | 依赖 kube-apiserver          |
| EdgeCore  | 边缘节点（工厂/门店）      | EdgeCore  | containerd/Docker、MQTT（可选） |

---

### 3. 架构特点

* **K8s 原生**：云端用原生 API Server，边缘直接感知 Pod、ConfigMap。
* **边缘自治**：网络中断时，EdgeCore 继续管理本地 Pod、设备。
* **轻量运行**：EdgeCore 占用资源远低于完整 kubelet。
* **多协议接入**：MQTT、Modbus、OPC-UA 等通过 DeviceTwin。

---

### 4. 多架构混部支持

KubeEdge 本质是：

* CloudCore 是普通 Kubernetes Deployment/Pod
* EdgeCore 是一个独立进程

因此它 **天然支持多 CPU 架构混合**，前提是：

1. CloudCore 容器镜像有 **amd64/arm64** 多架构版本（KubeEdge 官方镜像一般是 multi-arch）。
2. EdgeCore 安装在边缘节点时选择对应平台的二进制或镜像。
3. 云端 Kubernetes 集群本身也支持 **arm/amd 混部**（Kubernetes >=1.20 多架构调度已经成熟）。

实践上：

* 云端大多是 x86\_64（amd64）
* 边缘（树莓派、工控机）常见 arm64/armv7
* 通过 `nodeSelector` / `affinity` 把工作负载调度到匹配架构的节点即可

```yaml
# Pod 只跑在 arm64
apiVersion: v1
kind: Pod
metadata:
  name: arm-workload
spec:
  nodeSelector:
    kubernetes.io/arch: arm64
  containers:
  - name: app
    image: your-app:arm64
```

云端与边缘混部时，CloudCore 与 EdgeCore **通过协议解耦，不依赖相同 CPU 架构**，所以完全可行。

### 5. 总结

* **架构**：云端 CloudCore + 边缘 EdgeCore，长连接同步 Kubernetes 对象，断网自治。
* **多架构混部**：支持。CloudCore 可跑 amd64，EdgeCore 可跑 arm64/amd64/armv7，只需确保镜像/二进制匹配。
* **调度策略**：利用 `nodeSelector`、`affinity` 保证应用调度到正确架构。

## KubeEdge 部署方案

* **Kubernetes 集群 v1.28**（已有 master/control-plane）
* **容器运行时：containerd v1.7.24**
* 目标：在集群里部署 **KubeEdge**，使用 **keadm** 工具进行安装部署

### 集群环境规划

| IP           | 主机名    | OS                                        | ARCH    | 角色 | 负载                   |
| :----------- | ------ | :---------------------------------------- | :------ | -- | -------------------- |
| 192.168.2.118 | master | Kylin Linux Advanced Server V10 (Halberd) | x86_64 | 云端 | K8s，docker，cloudcore |
| 192.168.4.50 | node1  | Ubuntu 20.04                              | aarch64 | 边端 | docker，edgecore      |


部署KubeEdge组件版本,根据 Kubernetes 版本进行选定
master 节点
1. kubernetes v1.28.0
2. keadm v1.17.4
3. metrics-server v0.7.2 可选
4. 操作系统和架构：Kylin v10 amd64

node节点
1. keadm v1.17.4
2. docker v26.1.3
3. containerd v1.7.24
4. cri-dockerd v0.3.4
5. cni v1.8.0
6. 操作系统和架构：Ubuntu 20.04 arm64

提示
- [KubeEdge 和 Kubernetes 部署对应版本关系](https://github.com/kubeedge/kubeedge?tab=readme-ov-file#kubernetes-compatibility)
- root 或者 具有 sudo 权限的用户

## KubeEdge 环境准备
边端node节点
```
sudo apt-get update 
sudo apt-get upgrade
sudo apt-get install net-tools make vim openssh-server docker.io

# 网络参数配置(可选)
cat >> /etc/hosts << EOF
# GitHub Start
52.74.223.119 github.com
192.30.253.119 gist.github.com
54.169.195.247 api.github.com
185.199.111.153 assets-cdn.github.com
151.101.76.133 raw.githubusercontent.com
151.101.108.133 user-images.githubusercontent.com
151.101.76.133 gist.githubusercontent.com
151.101.76.133 cloud.githubusercontent.com
151.101.76.133 camo.githubusercontent.com
151.101.76.133 avatars0.githubusercontent.com
151.101.76.133 avatars1.githubusercontent.com
151.101.76.133 avatars2.githubusercontent.com
151.101.76.133 avatars3.githubusercontent.com
151.101.76.133 avatars4.githubusercontent.com
151.101.76.133 avatars5.githubusercontent.com
151.101.76.133 avatars6.githubusercontent.com
151.101.76.133 avatars7.githubusercontent.com
151.101.76.133 avatars8.githubusercontent.com
# GitHub End
EOF

# node节点设置主机名
hostnamectl set-hostname node1
```

- 部署分为两部分云端也就是k8s master 节点，另一部是边端，部署顺序为先云端后边端

## Cloud 节点部署
- 运行在 amd64 master节点
### 部署前准备
云端 master 节点需要对外暴露一个广播地址,边缘节点在加入集群时会用到，这里是通过metallb进行创建


创建`IPAddressPool`和`L2Advertisement`
```
kubectl -n metallb-system get IPAddressPool

cat > kubeedge-IPAddressPool.yaml << EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ippools-kubeedge
  namespace: metallb-system
spec:
  addresses:
  - 192.168.2.220-192.168.2.222 # 此处需要自定义
  autoAssign: false       # 关闭，后面手动指定
  avoidBuggyIPs: false
EOF

kubectl apply -f kubeedge-IPAddressPool.yaml

kubectl -n metallb-system get L2Advertisement

cat > kubeedge-L2Advertisement.yaml << EOF
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-kubeedge
  namespace: metallb-system
spec:
  ipAddressPools:
  - ippools-kubeedge  # 关联IPAddressPool
EOF

kubectl apply -f  kubeedge-L2Advertisement.yaml

# 查看部署状态
kubectl -n metallb-system get IPAddressPool 
kubectl -n metallb-system get L2Advertisement
```

### 部署cloudcore
保存为 `deploy-cloud.sh`：

```bash
#!/bin/bash
set -e

# === 配置部分 ===
KE_VERSION=v1.17.4
KUBE_CONFIG=/root/.kube/config
# 公网ip 或者 对外暴露的ip
ADVERTISE_ADDR=192.168.2.220   # 取 metallb 自定义IP

echo "[Cloud] 使用 IP: $ADVERTISE_ADDR"
echo "[Cloud] 安装 KubeEdge $KE_VERSION"

# === 下载 keadm ===
ARCH=$(uname -m)
if [ "$ARCH" == "x86_64" ]; then
  ARCH_NAME=amd64
elif [ "$ARCH" == "aarch64" ]; then
  ARCH_NAME=arm64
else
  echo "不支持的架构: $ARCH"
  exit 1
fi

# 1. 下载
curl -L https://github.com/kubeedge/kubeedge/releases/download/$KE_VERSION/keadm-$KE_VERSION-linux-$ARCH_NAME.tar.gz 
tar -zxvf keadm-$KE_VERSION-linux-$ARCH_NAME.tar.gz
mv keadm*/keadm/keadm /usr/local/bin/
#rm -rf keadm*

# 查看版本
keadm version

# 2. 安装
# === 初始化 CloudCore ===
# init执行后会自动生成cloudcore，是通过helm 部署的，创建kubeedge namespace

keadm init \
  --kube-config=${KUBE_CONFIG} \
  --advertise-address=${ADVERTISE_ADDR} \
  --profile version=${KE_VERSION}  \
  --set iptablesManager.mode="external" \

更多参数
#--set cloudCore.hostNetwork=false
#--set cloudCore.service.enable=true

#重新安装
# keadm reset
# 先正常删除，有问题再强制删除
# kubectl delete namespace kubeedge --grace-period=0 --force
# kubectl get ns kubeedge

# === 获取加入 token ===
TOKEN=$(keadm gettoken --kube-config=${KUBE_CONFIG})
echo "[Cloud] 节点加入 Token: $TOKEN"
echo "[Cloud] 请在 Edge 节点执行 join 时使用: $TOKEN"

# === 检查 CloudCore 状态 ===
kubectl get pod -n kubeedge -o wide

# 3. 阻止kube-proxy 等Daemonset服务调度边缘节点
# 通常做法是给边缘节点打一个标签
kubectl label node node1 node-role.kubernetes.io/edge=
# 通用地阻止 kube-system 下的多个 DaemonSet（比如 coredns、cilium、kube-proxy）

# 这里是禁止将 kube-system下的 ds 调度到边端节点，其它namespace 下的 ds 也同理，例如采集日志组件等
kubectl -n kube-system get daemonset \
  | grep -v NAME \
  | awk '{print $1}' \
  | xargs -n1 -I {} kubectl -n kube-system patch daemonset {} \
  -p '{
    "spec": {
      "template": {
        "spec": {
          "affinity": {
            "nodeAffinity": {
              "requiredDuringSchedulingIgnoredDuringExecution": {
                "nodeSelectorTerms": [
                  {
                    "matchExpressions": [
                      {
                        "key": "node-role.kubernetes.io/edge",
                        "operator": "DoesNotExist"
                      }
                    ]
                  }
                ]
              }
            }
          }
        }
      }
    }
  }'


# for 循环处理(参考)
for ns in default kube-system logging-system metallb-system; do
  kubectl -n ${ns} get daemonset \
    | grep -v NAME \
    | awk '{print $1}' \
    | xargs -n1 -I {} kubectl -n ${ns} patch daemonset {} \
    -p '{
      "spec": {
        "template": {
          "spec": {
            "affinity": {
              "nodeAffinity": {
                "requiredDuringSchedulingIgnoredDuringExecution": {
                  "nodeSelectorTerms": [
                    {
                      "matchExpressions": [
                        {
                          "key": "node-role.kubernetes.io/edge",
                          "operator": "DoesNotExist"
                        }
                      ]
                    }
                  ]
                }
              }
            }
          }
        }
      }
    }'
done


# 4. 启用metrics-server,用来查看边缘上pod的日志(可选且推荐),如果已经部署则无序重复部署
kubectl -n kube-system get pod|grep metrics-server

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.7.2/components.yaml

# 忽略tls 校验
kubectl -n kube-system patch deployment metrics-server \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# 增加一条iptables 不然会初始化报错
iptables -t nat -A OUTPUT -p tcp --dport 10350 -j DNAT --to $ADVERTISE_ADDR:10003

# 查看部署状态
kubectl -n kubeedge get pod
# 获取token，24h 失效
keadm gettoken
```

### 指定 metallb 自定义的广播地址

```
$ kubectl -n kubeedge edit svc cloudcore
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: cloudcore
    meta.helm.sh/release-namespace: kubeedge
    
    ........
  
  - name: tunnelport
    port: 10004
    protocol: TCP
    targetPort: 10004
  selector:
    k8s-app: kubeedge
    kubeedge: cloudcore
  sessionAffinity: None
  type: ClusterIP   # 变更
status:
  loadBalancer: {}    
# 调整
  ...
  sessionAffinity: None
  type: LoadBalancer             # 修改
  loadBalancerIP: 192.168.2.220   # 新增
```
测试端口是否是通的
```
telnet 192.168.2.220 10000
```
修改cloudcore config
```
$ kubectl -n kubeedge get cm cloudcore -o yaml|grep dynamicController -A 2
      dynamicController:
        enable: true   # 调整为true
      edgeController:

$ kubectl -n kubeedge get cm cloudcore -o yaml|grep cloudStream  -A 2                 
      cloudStream:
        enable: true  # 调整为true
        streamPort: 10003
```
## Edge 节点部署
- 运行在 arm 工控机
### 容器运行时

### deploy-edge

保存为 `deploy-edge.sh`：

```bash
#!/bin/bash
set -e

# === 配置部分 ===
KE_VERSION=v1.17.4
CLOUD_IP=192.168.2.220   # 👈 请手动替换为 master 内网IP
TOKEN=<Token_来自_Cloud>  # 👈 请替换为 Cloud 节点生成的 token
EDGE_NAME=$(hostname)     # 用主机名作为 Edge 节点名
CGROUP_DRIVER=systemd

echo "1. [Edge] 节点名: $EDGE_NAME" "2.[Edge] 安装 KubeEdge $KE_VERSION" "3. CGROUP_DRIVER is $CGROUP_DRIVER"


# === 下载 keadm ===
ARCH=$(uname -m)
if [ "$ARCH" == "x86_64" ]; then
  ARCH_NAME=amd64
elif [ "$ARCH" == "aarch64" ]; then
  ARCH_NAME=arm64
else
  echo "不支持的架构: $ARCH"
  exit 1
fi

# 1. 安装容器运行时，docker、contaierd 自行选择
# 2. 安装keadm
curl -L https://github.com/kubeedge/kubeedge/releases/download/$KE_VERSION/keadm-$KE_VERSION-linux-$ARCH_NAME.tar.gz -o keadm.tar.gz
tar -zxvf keadm.tar.gz
mv keadm*/keadm/keadm /usr/local/bin/
keadm version
#rm -rf keadm*

# 3. 安装EdgeCore
# === Edge 节点加入集群 ===
# 可以提前将kubeedge/installation-package:v1.17.4 镜像导入至边缘节点的containerd中
# ctr -n k8s.io images import installation-package-v1.17.4.tar

keadm join \
  --cloudcore-ipport=${CLOUD_IP}:10000 \
  --edgenode-name=${EDGE_NAME} \
  --token=${TOKEN} \
  --cgroupdriver=${CGROUP_DRIVER} \
  --tarballpath=/root/kubeedge/images/kubeedge_installation-package_v1.17.4_arm64-export.tar \
  --remote-runtime-endpoint=unix:///var/run/cri-dockerd.sock \  # 更多参数
  --with-mqtt

# 安装失败清理配置重新部署
# keadm reset 会尝试同时清理 cloudcore 和 edgecore 的安装
keadm reset
rm -rf /etc/kubeedge/*

# 4. 安装成功后，部署cni
# 上传至/opt/cni/bin 目录下解压
tar -zxvf cni-plugins-linux-arm64-v1.8.0.tgz
rm cni-plugins-linux-arm64-v1.8.0.tgz

# === 确认 EdgeCore 服务状态 ===
systemctl status edgecore --no-pager
# or
service edgecore status
# 查看edgecore 日志
journalctl -u edgecore.service -f
journalctl -u edgecore.service -xe

# 6. 修改配置
# 备份
cp  /etc/kubeedge/config/edgecore.yaml  /etc/kubeedge/config/edgecore.yaml.bak
# 6.1 开启EdgeStream(可选),搭配metrics-server 查看pod 日志,当然也可以直接通过docker 进行查看日志
# vi /etc/kubedge/config/edgecore.yaml
# 找到 edgeStram 下的 enable 置为true后重启edgecore服务

# 6.2 修改Endpoint 为 cri-dockerd
# cat /etc/kubeedge/config/edgecore.yaml |grep Endpoint
    remoteImageEndpoint: unix:///var/run/cri-dockerd.sock
    remoteRuntimeEndpoint: unix:///var/run/cri-dockerd.sock
      containerRuntimeEndpoint: unix:///var/run/cri-dockerd.sock
      imageServiceEndpoint: unix:///var/run/cri-dockerd.sock
      
# edgecore restart

# 检查生成文件
ls /etc/systemd/system/edgecore.service 
# 配置文件
ls /etc/kubeedge/config/edgecore.yaml
```

安装CNI 网络插件-需要科学上网,KubeEdge 官方推荐安装方式(可选)
```
# 查看是否已安装
$ ls -lh /opt/cni/bin/
# 如果没有则需要科学上网进行安装
$ git clone https://github.com/kubeedge/kubeedge.git
$ cd kubeedge/hack/lib
$ source install.sh 

# install_cni_plugins 会先检查 /opt/cni/bin/ 下是否已经有插件，如果有，就直接跳过安装
$ install_cni_plugins
The installation of the cni plugin will overwrite the cni config file. Use export CNI_CONF_OVERWRITE=false to disable it.
CNI plugins already installed and no need to install

# 强制覆盖
systemctl stop edgecore
export CNI_CONF_OVERWRITE=true


# 重新安装cni
mv /opt/cni/bin /opt/cni/bin.bak
mkdir -p /opt/cni/bin
$ install_cni_plugins
...
root@node1:/opt/cni# install_cni_plugins     
The installation of the cni plugin will overwrite the cni config file. Use export CNI_CONF_OVERWRITE=false to disable it.
start installing CNI plugins...
--2025-09-09 16:35:04--  https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
正在连接 192.168.3.73:7890... 已连接。
已发出 Proxy 请求，正在等待回应... 302 Found
位置：https://release-assets.githubusercontent.com/github-production-release-asset/84575398/34412816-cbca-47a1-a428-9e738f2451d8?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-09-09T09%3A14%3A20Z&rscd=attachment%3B+filename%3Dcni-plugins-linux-amd64-v1.1.1.tgz&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-09-09T08%3A13%3A30Z&ske=2025-09-09T09%3A14%3A20Z&sks=b&skv=2018-11-09&sig=DLHC23bDHN591glN%2BbbNh%2BmQArl5fnmctIRVIM7zhvo%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc1NzQwNzIwNSwibmJmIjoxNzU3NDA2OTA1LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.eVXcWS_SAbMSnboMUyVldhLt6NDO69Jt8OUS4Qw4txE&response-content-disposition=attachment%3B%20filename%3Dcni-plugins-linux-amd64-v1.1.1.tgz&response-content-type=application%2Foctet-stream [跟随至新的 URL]
--2025-09-09 16:35:05--  https://release-assets.githubusercontent.com/github-production-release-asset/84575398/34412816-cbca-47a1-a428-9e738f2451d8?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-09-09T09%3A14%3A20Z&rscd=attachment%3B+filename%3Dcni-plugins-linux-amd64-v1.1.1.tgz&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-09-09T08%3A13%3A30Z&ske=2025-09-09T09%3A14%3A20Z&sks=b&skv=2018-11-09&sig=DLHC23bDHN591glN%2BbbNh%2BmQArl5fnmctIRVIM7zhvo%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc1NzQwNzIwNSwibmJmIjoxNzU3NDA2OTA1LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.eVXcWS_SAbMSnboMUyVldhLt6NDO69Jt8OUS4Qw4txE&response-content-disposition=attachment%3B%20filename%3Dcni-plugins-linux-amd64-v1.1.1.tgz&response-content-type=application%2Foctet-stream
正在连接 192.168.3.73:7890... 已连接。
已发出 Proxy 请求，正在等待回应... 200 OK
长度： 36336160 (35M) [application/octet-stream]
正在保存至: “cni-plugins-linux-amd64-v1.1.1.tgz”

cni-plugins-linux-amd64-v1.1.1.tgz             100%[===================================================================================================>]  34.65M   632KB/s    用时 84s   

2025-09-09 16:36:30 (421 KB/s) - 已保存 “cni-plugins-linux-amd64-v1.1.1.tgz” [36336160/36336160])

./
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth

```

- cni 离线安装,[下载指定版本](https://github.com/containernetworking/plugins/releases),高版本会向后兼容

### 注意

1.  edge边缘节点不要部署kubelet
2.  `/etc/kubeedge/config/edgecore.yaml` 修改mqqt 连接地址

## KubeEdge 验证部署

在 master 节点上：

```bash
kubectl get node -o wide
NAME     STATUS   ROLES           AGE   VERSION                     INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                                    KERNEL-VERSION                    CONTAINER-RUNTIME
master   Ready    control-plane   62d   v1.28.0                     192.168.2.118   <none>        Kylin Linux Advanced Server V10 (Halberd)   4.19.90-89.11.v2401.ky10.x86_64   containerd://1.7.15
node1    Ready    agent,edge      42m   v1.28.11-kubeedge-v1.17.4   192.168.4.50   <none>        Ubuntu 20.04.6 LTS                          5.10.209-rt89                     containerd://1.7.24

# 给node1 打taint
kubectl taint nodes node1 node-role.kubernetes.io/edge=:NoSchedule
# 取消taint
kubectl taint nodes node1 node-role.kubernetes.io/edge:NoSchedule-
kubectl taint nodes node1 node.kubernetes.io/disk-pressure:NoSchedule-



# 查看节点状态
[root@master ~]# kubectl get node --show-labels
NAME     STATUS   ROLES           AGE   VERSION                     LABELS
master   Ready    control-plane   62d   v1.28.0                     beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
node1    Ready    agent,edge      31m   v1.28.11-kubeedge-v1.17.4   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,kubernetes.io/arch=arm64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,node-role.kubernetes.io/agent=,node-role.kubernetes.io/edge=
```

应该能看到 Edge 节点（ARM 工控机）处于 `Ready` 状态。

测试运行 Pod，在云端将 pod 调度至边端：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-edge
spec:
  nodeSelector:
    kubernetes.io/hostname: <Edge节点的hostname>
  containers:
  - name: nginx
    image: nginx:alpine
```

`kubectl apply -f nginx-edge.yaml`

节点容忍
```
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-edge
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/edge"
        operator: "Exists"
        effect: "NoSchedule"
      nodeSelector:
        kubernetes.io/hostname: node1
      containers:
      - name: nginx
        image: nginx:alpine
        imagePullPolicy: IfNotPresent
```
`kubectl create -f soft-honeypot-edge.yaml`
```
    apiVersion: v1
    kind: Pod
    metadata:
      name: soft-honeypot-edge
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/edge"
        operator: "Exists"
        effect: "NoSchedule"
      nodeSelector:
        kubernetes.io/hostname: node1
      containers:
      - name: soft-honeypot
        image: docker.io/library/soft-honeypot:v1
        imagePullPolicy: IfNotPresent
```

## CloudCore 配置文件参数释义(回顾)
```
[root@master soft-honeypot]# kubectl -n kubeedge get cm cloudcore -o yaml
apiVersion: v1
data:
  cloudcore.yaml: |
    apiVersion: cloudcore.config.kubeedge.io/v1alpha2
    commonConfig:
      monitorServer:
        bindAddress: 127.0.0.1:9091
      tunnelPort: 10351
    featureGates:
      requireAuthorization: false
    kind: CloudCore
    kubeAPIConfig:
      burst: 10000
      contentType: application/vnd.kubernetes.protobuf
      kubeConfig: ""
      master: ""
      qps: 5000
    modules:
      cloudHub:
        advertiseAddress:
        - 192.168.2.220
        dnsNames:
        - ""
        edgeCertSigningDuration: 365
        enable: true
        https:
          address: 0.0.0.0
          enable: true
          port: 10002
        keepaliveInterval: 30
        nodeLimit: 1000
        quic:
          address: 0.0.0.0
          enable: false
          maxIncomingStreams: 10000
          port: 10001
        tlsCAFile: /etc/kubeedge/ca/rootCA.crt
        tlsCAKeyFile: /etc/kubeedge/ca/rootCA.key
        tlsCertFile: /etc/kubeedge/certs/edge.crt
        tlsPrivateKeyFile: /etc/kubeedge/certs/edge.key
        tokenRefreshDuration: 12
        unixsocket:
          address: unix:///var/lib/kubeedge/kubeedge.sock
          enable: true
        websocket:
          address: 0.0.0.0
          enable: true
          port: 10000
        writeTimeout: 30
      cloudStream:
        enable: true
        streamPort: 10003
        tlsStreamCAFile: /etc/kubeedge/ca/streamCA.crt
        tlsStreamCertFile: /etc/kubeedge/certs/stream.crt
        tlsStreamPrivateKeyFile: /etc/kubeedge/certs/stream.key
        tlsTunnelCAFile: /etc/kubeedge/ca/rootCA.crt
        tlsTunnelCertFile: /etc/kubeedge/certs/server.crt
        tlsTunnelPrivateKeyFile: /etc/kubeedge/certs/server.key
        tunnelPort: 10004
      deviceController:
        buffer:
          deviceEvent: 1
          deviceModelEvent: 1
          updateDeviceStatus: 1024
        enable: true
        load:
          updateDeviceStatusWorkers: 1
      dynamicController:
        enable: true
      edgeController:
        buffer:
          certificateSigningRequest: 1024
          configMapEvent: 1
          createLease: 2024
          createNode: 1024
          createPod: 1024
          deletePod: 1024
          patchNode: 1524
          patchPod: 1024
          podEvent: 1
          queryConfigMap: 1024
          queryLease: 1024
          queryNode: 2024
          queryPersistentVolume: 1024
          queryPersistentVolumeClaim: 1024
          querySecret: 1024
          queryVolumeAttachment: 1024
          ruleEndpointsEvent: 1
          rulesEvent: 1
          secretEvent: 1
          serviceAccountToken: 1024
          updateNode: 1024
          updateNodeStatus: 1024
          updatePodStatus: 1024
        enable: true
        load:
          CreatePodWorks: 4
          ServiceAccountTokenWorkers: 100
          UpdateRuleStatusWorkers: 4
          certificateSigningRequestWorkers: 4
          createLeaseWorkers: 1000
          createNodeWorkers: 100
          deletePodWorkers: 100
          patchNodeWorkers: 120
          patchPodWorkers: 100
          queryConfigMapWorkers: 100
          queryLeaseWorkers: 100
          queryNodeWorkers: 1000
          queryPersistentVolumeClaimWorkers: 4
          queryPersistentVolumeWorkers: 4
          querySecretWorkers: 100
          queryVolumeAttachmentWorkers: 4
          updateNodeStatusWorkers: 1
          updateNodeWorkers: 4
          updatePodStatusWorkers: 1
        nodeUpdateFrequency: 10
      iptablesManager:
        enable: true
        mode: external
      router:
        address: 0.0.0.0
        enable: false
        port: 9443
        restTimeout: 60
      syncController:
        enable: true
      taskManager:
        buffer:
          taskEvent: 1
          taskStatus: 1024
        enable: false
        load:
          taskWorkers: 1
```


### 🔑 顶层配置

#### `commonConfig`

* **monitorServer.bindAddress**
  CloudCore 内置的监控指标服务监听地址（通常用于 Prometheus 抓取 metrics）。
* **tunnelPort**
  Edge 节点通过隧道（CloudStream）访问云端时的端口。

#### `featureGates`

* **requireAuthorization**
  是否启用边缘节点连接的权限认证。

    * `true`：需要额外的认证授权。
    * `false`：仅凭证书即可。

---

### 🧩 modules（核心模块）

#### 1. **cloudHub**

边缘节点和云端交互的核心模块，负责维持长连接。

* **advertiseAddress**：对外公布的 CloudHub 服务 IP（边缘节点连接用）。
* **dnsNames**：证书里的 DNS 名称，用于 TLS 验证。
* **edgeCertSigningDuration**：边缘证书有效期（天）。
* **keepaliveInterval**：心跳间隔，维持与边缘的长连接。
* **nodeLimit**：最大允许接入的边缘节点数量。
* **tlsCAFile / tlsCAKeyFile / tlsCertFile / tlsPrivateKeyFile**：证书和私钥配置。
* **tokenRefreshDuration**：token 刷新周期（小时）。
* **writeTimeout**：消息写入的超时时间（秒）。
* **unixsocket**：是否启用本地 Unix Socket 通道。
* **websocket**：基于 WebSocket 的边缘连接配置（常用）。
* **https**：提供 HTTPS 连接的配置。
* **quic**：启用 QUIC 协议（性能更好，但默认关闭）。

---

#### 2. **cloudStream**

用于边缘节点 **端口转发、远程调试、exec、日志查看**。

* **enable**：是否启用。
* **streamPort**：边缘连接的流端口。
* **tlsStreamCAFile / tlsStreamCertFile / tlsStreamPrivateKeyFile**：流式通信 TLS 证书。
* **tlsTunnelCAFile / tlsTunnelCertFile / tlsTunnelPrivateKeyFile**：隧道通信 TLS 证书。
* **tunnelPort**：隧道端口。

---

#### 3. **deviceController**

设备管理模块，负责云端和边缘设备 CRD 同步。

* **enable**：是否启用。
* **buffer**：各种事件的缓冲区大小（避免高并发丢事件）。
* **load**：worker 数量，决定并发处理能力。

---

#### 4. **dynamicController**

负责 **动态 CRD（自定义资源）同步**，比如应用规则、消息路由等。

* **enable**：是否启用。

---

#### 5. **edgeController**

核心模块，负责 **K8s 资源下发和状态回传**。

* **enable**：是否启用。
* **buffer**：各种消息队列的缓冲区大小，比如 `createPod`、`updatePodStatus`。
* **load**：worker 数量（对应不同事件的处理并发度）。
* **nodeUpdateFrequency**：同步 Node 状态的周期（秒）。

---

#### 6. **iptablesManager**

管理 KubeEdge 的流量转发规则。

* **enable**：是否启用。
* **mode**：工作模式：

    * `internal` → 由 cloudcore 自己管理 iptables
    * `external` → 外部系统负责（常见于和 flannel/calico 集成）

---

#### 7. **router**

消息路由模块（MQTT/HTTP 规则引擎）。

* **enable**：是否启用。
* **address**：监听地址。
* **port**：监听端口。
* **restTimeout**：REST 请求超时时间（秒）。

---

#### 8. **syncController**

* **enable**：是否启用。
  主要负责 **CRD 同步和一致性**，保证边缘和云端资源匹配。

---

#### 9. **taskManager**

用于 **分布式任务调度**（实验性功能，默认关闭）。

* **enable**：是否启用。
* **buffer**：任务事件缓冲区。
* **load**：任务 worker 数量。

---

### 📌 总结

* **cloudHub** → 边缘连接通道（WebSocket/QUIC/HTTPS/Unix Socket）
* **cloudStream** → 远程调试和端口转发
* **deviceController** → 设备管理
* **dynamicController** → 动态 CRD 管理
* **edgeController** → Pod/Node/ConfigMap 等资源下发与状态同步
* **iptablesManager** → 流量规则管理
* **router** → 规则引擎消息路由
* **syncController** → CRD 同步一致性
* **taskManager** → 分布式任务调度


## 后记
### KubeEdge 边缘节点无法通过Service 创建NodePort类型
原因和 Kubernetes 与 KubeEdge 的架构有关

#### 1️⃣ NodePort 工作机制

在普通 Kubernetes 集群中：

* NodePort Service 会在 **每个 Node 的 kube-proxy 上**创建 iptables 或 ipvs 规则，把指定的 `nodePort` 映射到 Pod 的 targetPort。
* 也就是说，NodePort 实际是在节点网络层做端口映射，由 kube-proxy 负责。



#### 2️⃣ KubeEdge 的边缘节点限制

* 在 KubeEdge 中，**边缘节点上的 `edged` 并不会运行完整的 kube-proxy**。
* 默认情况下，边缘节点只负责调度 Pod、同步 Pod 状态、镜像拉取等。
* 边缘节点不会自动开放 NodePort 端口或做负载转发。
* NodePort 和 LoadBalancer 类型的 Service 仍然依赖 **云端的 kube-proxy 或负载均衡器** 处理流量。

所以在边缘节点：

* Pod 会创建并运行成功。
* 但 NodePort 不会在边缘节点生效，也不会监听端口。



#### 3️⃣ 解决方案

1. **使用 HostPort（你 Deployment 中已经用 hostPort）**

    * 直接在 Pod Spec 里设置 `hostPort`，Pod 会在边缘节点上监听指定端口。
    * 适合单节点或边缘节点直接访问 Pod 服务。
    * 例子：

      ```yaml
      ports:
        - containerPort: 8080
          hostPort: 8080
          name: http
      ```
    * 你的 Deployment 已经设置了 hostPort，边缘节点会监听这些端口。

2. **使用 CloudHub 代理访问 NodePort 服务**

    * 如果一定要使用 NodePort，访问需要经过云端节点（master）转发到边缘节点。

3. **使用 LoadBalancer（在支持 LB 的环境）**

    * 在边缘节点上，LB IP 或外部流量可以直接访问 Pod。
    * 但是需要 MetalLB 或类似组件支持。


#### 总结

* NodePort **在 KubeEdge 边缘节点上不会自动创建监听端口**。
* 如果想在边缘节点直接访问服务，请使用 **hostPort**。
* NodePort 主要用于云端节点或完整 Kubernetes 集群环境，不适合裸边缘节点直接暴露。


## 参考
1. [KubeEdge 官网](https://kubeedge.io/zh/)
2. [KubeEdge 和 Kubernetes 部署对应版本关系](https://github.com/kubeedge/kubeedge?tab=readme-ov-file#kubernetes-compatibility)

