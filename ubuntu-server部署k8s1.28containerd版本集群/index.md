# Ubuntu-server部署k8s1.28(containerd版本)集群

## 整体流程
1. 部署准备
2. k8s 集群容器运行时 Containerd 准备
3. k8s 集群部署
   1. k8s 集群软件 apt 源准备
   2. k8s 集群软件安装
   3. k8s 集群初始化
   4. k8s 集群worker 节点加入
4. k8s 集群网络插件 Calico 准备
5. 部署应用验证集群可用性

## 集群环境信息
- 操作系统Ubuntu22.04
- 部署 k8s 版本为v1.28
- 容器运行时为 Containerd 1.7.11

## 1. 主机准备
虚拟机是通过vmvare 创建的，搭建的k8s 集群是非高可用模式，下方提供部署脚本。
部署过程中由于2024年6月份 docker 镜像仓库不再对大陆提供服务，需要进行离线部署，主要是 calico 这一块会有镜像源缺失,对应章节会有补充说明及解决方案

### 1.1 操作系统
| 操作系统 | 版本                             | 说明   |
| :------- | :------------------------------- | :----- |
| ubuntu   | ubuntu-22.04.4-live-server-amd64 | 最小化 |
### 1.2 主机硬件配置说明

| ip           | CPU  | 内存 | 硬盘 | 角色   | 主机名        |
| :----------- | :--- | :--- | :--- | :----- | :------------ |
| 192.168.1.11 | 2C   | 4G   | 30G  | master | k8s-nastert01 |
| 192.168.1.7  | 2C   | 4G   | 30G  | node   | k8s-node01     |

### 1.3 主机配置
所有节点执行
#### 1.3.1 主机名配置
```shell
#master 节点
hostnamectl set-hostname k8s-nastert01
#ndoe 节点
hostnamectl set-hostname k8s-node01
```
#### 1.3.2 主机ip 地址配置
vmvare 创建虚拟机网络适配器选择桥接模式(自动)，需要手动配置ip地址，否则当虚拟机重启后ip 地址会发生变化
k8s-master01 节点
```shell
sudo cp /etc/netplan/00-installer-config.yaml  /etc/netplan/00-installer-config.yaml_before
cat << EOF > /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      addresses:
      - 192.168.1.11/16
      nameservers:
        addresses:
        - 114.114.114.114 # 国内移动联通
        - 8.8.8.8		  # 谷歌
        search: []
      routes:
      - to: default
        via: 192.168.1.1  # 电脑本地网关地址
  version: 2
EOF
# 使配置生效
sudo netplan apply
```
k8s-node01 节点
```shell
sudo cp /etc/netplan/00-installer-config.yaml  /etc/netplan/00-installer-config.yaml_before
cat << EOF > /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      addresses:
      - 192.168.1.7/16
      nameservers:
        addresses:
        - 114.114.114.114 # 国内移动联通
        - 8.8.8.8		  # 谷歌
        search: []
      routes:
      - to: default
        via: 192.168.1.1
  version: 2
EOF
# 使配置生效
sudo netplan apply
```
#### 1.3.3 主机名和ip 地址解析

```shell
cat >> /etc/hosts << EOF
> 192.168.1.11 k8s-master01
> 192.168.1.7 k8s-node01
> EOF
```
#### 1.3.4 chrony时间同步配置
k8s-master01 节点执行
```shell
$ cp /etc/chrony/chrony.conf  /etc/chrony/chrony.conf_bak
$ cat > /etc/chrony/chrony.conf << EOF
confdir /etc/chrony/conf.d
server ntp.aliyun.com iburst
allow 192.168.1.1/24
sourcedir /run/chrony-dhcp
sourcedir /etc/chrony/sources.d
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
ntsdumpdir /var/lib/chrony
logdir /var/log/chrony
maxupdateskew 100.0
rtcsync
makestep 1 3
leapsectz right/UTC
EOF

$ date
Thu Jun 20 10:10:56 AM CST 2024
# 设置时区
$ timedatectl set-timezone Asia/Shanghai
# 部署时间同步服务器，并设置开机自启
$ apt install ntpdate chrony   -y
$ ntpdate time2.aliyun.com
$ systemctl start chrony
$ systemctl status chrony
$ systemctl enable chrony
# 查看 chronyd 时间同步服务的源的统计信息
$ chronyc sourcestats -v
```
k8s-node01 节点执行,修改 server 为 master 节点ip
```shell
$ cp /etc/chrony/chrony.conf  /etc/chrony/chrony.conf_bak
$ cat > /etc/chrony/chrony.conf << EOF
confdir /etc/chrony/conf.d
server 192.168.1.11 iburst
sourcedir /run/chrony-dhcp
sourcedir /etc/chrony/sources.d
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
ntsdumpdir /var/lib/chrony
logdir /var/log/chrony
maxupdateskew 100.0
rtcsync
makestep 1 3
leapsectz right/UTC
EOF

$ date
Thu Jun 20 10:15:56 AM CST 2024
# 设置时区
$ timedatectl set-timezone Asia/Shanghai
# 部署时间同步服务器，并设置开机自启
$ apt install ntpdate chrony   -y
$ ntpdate time2.aliyun.com
$ systemctl start chrony
$ systemctl status chrony
$ systemctl enable chrony
# 查看 chronyd 时间同步服务的源的统计信息
$ chronyc sourcestats -v
```
#### 1.3.5 配置内核转发及网桥过滤
```shell
# 转发IPv4并让iptables看到桥接流量
$ sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 手动加载
$ modprobe overlay
$ modprobe br_netfilter
# 检测是否加载
$ lsmod | egrep "overlay"
$ lsmod | egrep "br_netfilter"

# 添加网桥过滤及内核转发配置,设置所需的sysctl参数，参数在重新启动后保持不变
$ sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

#检查sysctl是否成功应用：
$ sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

# 手动加载
#sudo sysctl -p /etc/sysctl.d/k8s.conf
# or
$ sudo sysctl --system
```
#### 1.3.6 安装ipset和ipvsadm
```shell
$ apt install ipset ipvsadm -y

# 配置ipvsadm 内核模块加载
$ cat << EOF > /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

# 手动加载模块
$ modprobe -- ip_vs
$ modprobe -- ip_vs_rr
$ modprobe -- ip_vs_wrr
$ modprobe -- ip_vs_sh
$ modprobe -- nf_conntrack
```
#### 1.3.7 关闭SWAP分区
```shell
# 临时关闭
$ swapoff -a
# 永久关闭
$ sed  -i '/swap/ s/^\(.*\)$/#\1/g' /etc/fstab
```
#### 1.3.8 所有设备允许 root 用户
```shell
$ cat >> /etc/ssh/sshd_config << EOF
PermitRootLogin yes
PasswordAuthentication yes
EOF
```
#### 1.3.9 关闭防火墙
```shell
#查看防火墙状态
$ sudo systemctl status ufw
#关闭防火墙
$ sudo systemctl stop ufw
#禁止防火墙开机启动
$ sudo systemctl disable ufw
```
## 2. 部署Conatinerd 服务
所有主机都要执行
### 2.1 下载及安装

```shell
# 部署 containerd
$ wget https://github.com/containerd/containerd/releases/download/v1.7.11/cri-containerd-static-1.7.11-linux-amd64.tar.gz
#tar Czxvf /usr/local/ cri-containerd-static-1.7.11-linux-amd64.tar.gz
$ sudo tar -zxvf cri-containerd-cni-1.7.11-linux-amd64.tar.gz -C /
```
{{< admonition warning >}}
这里选择的适配k8s 容器运行时的containerd,有一个前缀cri(container runtime interface)，另外根据自己的操作系统选择对应的架构，常用的是amd64
{{< /admonition >}}

### 2.2  配置Containerd服务
```shell
$ mkdir -p /etc/containerd
$ containerd config default > /etc/containerd/config.toml
# 修改为国内代理镜像仓库
$ sed -i 's/registry.k8s.io\/pause:3.8/registry.cn-hangzhou.aliyuncs.com\/google_containers\/pause:3.9/g' /etc/containerd/config.toml
# 驱动使用为SystemdCgroup
$ sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

```
### 2.3 设置containerd开机自启动
```shell
$ systemctl enable --now containerd
#验证版本
$ containerd --version
```
## 3. k8s集群部署
所有节点都执行
### 3.1  k8s配置集群软件apt源及安装
```shell
#下载用于 Kubernetes 软件包仓库的公共签名密钥。所有仓库都使用相同的签名密钥，因此你可以忽略URL中的版本
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 添加 Kubernetes apt 仓库。 请注意，此仓库仅包含适用于 Kubernetes 1.28 的软件包
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 更新 apt 包索引并安装使用 Kubernetes apt 仓库所需要的包
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# k8s 集群软件安装
# 3.2.1 更新 apt 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本(防止自动更新)：
$ sudo apt-get update
$ apt-get install -y kubelet kubeadm kubectl
# 并锁定其版本(防止自动更新)
$ apt-cache madison kubeadm
$ sudo apt-mark hold kubelet kubeadm kubectl

```
### 3.2 查看apt 仓库是否缓存k8s 安装组件
```shell
apt-cache policy kubeadm|head - 10
```
### 3.3 部署k8s 集群
#### master 节点
1. 修改kubeadm 初始化配置文件
```shell
#默认初始化配置文件生成
$ kubeadm config print init-defaults > kube-config.yaml
# 修改配置文件
$ sed -i "s/advertiseAddress: 1.2.3.4/advertiseAddress: 192.168.1.11/g" kube-config.yaml
$ sed -i "s/name: node/name: k8s-master01/g" kube-config.yaml
$ sed -i "s/imageRepository: k8s.gcr.io/imageRepository: $ registry.cn-hangzhou.aliyuncs.com\/google_containers/g" kube-config.yaml
$ sed -i "s/imageRepository: registry.k8s.io/imageRepository: $ registry.cn-hangzhou.aliyuncs.com\/google_containers/g" kube-config.yaml
# 追加制定pod ip网段
$ sed -i  '/serviceSubnet/a\  podSubnet: 10.244.0.0/16' kube-config.yaml
```
2. 添加ipvs 和 kubelet 配置,`kube-config.yaml`为生成的配置文件

```shell
$ cat << EOF >> kube-config.yaml
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
EOF
```
3. 提前下载需要的镜像信息
```shell
# 查看kubeadm 所需要的镜像列表
$ kubeadm config images list
# 提前下载镜像,使用中国区镜像加速
$ kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers
# 查看下载的镜像，镜像保存在k8s.io namesapce 下
$ ctr -n=k8s.io i ls
```
4. 开始部署，并将log 打印到本地文件中
```shell
# 使用部署配置文件初始化k8s 集群
$ kubeadm init --config  kube-config.yaml|tee -a kubeadm_init.log

#输出内容如下：
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.11:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:fb0b8abf6780b7dff4f155edd2ed74a80a085081e25bb5469b906410a7a80398
```
4. 执行成功后
根据提示还需要再执行以下命令
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```
{{< admonition tip >}}
执行失败查看报错后调整，使用`kubeadm reset`还原后重新执行
{{< /admonition >}}

5. master 节点验证
```shell
$ kubectl get node -o wide
NAME           STATUS   ROLES           AGE     VERSION    INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-master01   Ready    control-plane   7d12h   v1.28.11   192.168.1.11   <none>        Ubuntu 22.04.4 LTS   5.15.0-94-generic   containerd://1.7.11
```
6. kube-config.yaml 完成配置文件
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.1.11  # 修改内容
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock  # 修改内容
  imagePullPolicy: IfNotPresent
  name: k8s-master01   			# 修改内容
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers  # 修改内容
kind: ClusterConfiguration
kubernetesVersion: 1.28.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16   # 添加内容
scheduler: {}
# 添加内容
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```
#### node 节点 join
这里手动指定cri-socket
```shell
kubeadm join 192.168.1.11:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash \
        --cri-socket=unix:///run/containerd/containerd.sock
```
## 4. k8s集群部署网络插件 calico
常见的网络插件有多种，比如flannel、Calico和Cilium，本次实验使用Calico使用。

首先访问Calico帮助文档https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

### master 节点
calico 部署使用的是operator 的方式，且不再和kube-system 共用一个namespace，另起了一个namesapce calico-system,引入了istio 相关组件服务，详情可以看下calico官网

配置插件需要的环境
```shell
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
```
下载客户端资源文件并修改pod 网段地址，因为我们的pod的网段和配置文件不一样
```shell
# 下载客户端资源文件
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml
# 修改pod的网段地址 podSubnet
sed -i 's/cidr: 192.168.0.0/cidr: 10.244.0.0/g' custom-resources.yaml
kubectl create -f custom-resources.yaml
```
查看集群 calico pod 创建过程，大约会耗时三分钟,部署成功后，所有node 节点为Ready 状态
```shell
watch kubectl get all -o wide -n calico-system
```
{{< admonition tip >}}
以上方法是在dockerhub 网络仓库正常访问情况下执行的，如果dockerhub 网络仓库访问异常可以使用离线部署的方式

2024.6 月份docker hub 被墙了后，就不能方便的下载 docker 镜像了，这里使用离线的方式进行部署。
{{< /admonition >}}

calico v3.27 相关镜像列表
```
docker.io/calico/cni:v3.27.2
docker.io/calico/kube-controllers:v3.27.2
docker.io/calico/node:v3.27.2
docker.io/calico/pod2daemon-flexvol:v3.27.2
docker.io/calico/typha:v3.27.2
# 下面三个镜像 release-v3.27.2 没有包含进来
docker.io/calico/csi:v3.27.2
docker.io/calico/apiserver:v3.27
docker.io/calico/node-driver-registrar:v3.27.2
```

手动镜像导入
```shell
sudo ctr -n k8s.io images ls|grep calico

sudo ctr -n k8s.io images import calico-cni.tar  
sudo ctr -n k8s.io images import calico-dikastes.tar  
sudo ctr -n k8s.io images import calico-flannel-migration-controller.tar  
sudo ctr -n k8s.io images import calico-kube-controllers.tar  
sudo ctr -n k8s.io images import calico-node.tar  
sudo ctr -n k8s.io images import calico-pod2daemon.tar  
sudo ctr -n k8s.io images import calico-typha.tar
# # 下面三个镜像 release-v3.27.2 没有包含进来,需要手动导入
sudo ctr -n k8s.io images import csi.tar
sudo ctr -n k8s.io images import apiserver.tar
sudo ctr -n k8s.io images import node-driver-registrar.tar
```

参考：[calico_3.27 quick install 官网](https://docs.tigera.io/calico/3.27/getting-started/kubernetes/quickstart)

## 5. 部署nginx 验证k8s 集群可用性
nginx-deploy yaml 文件
```yaml
cat << EOF > nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - image: registry.cn-shenzhen.aliyuncs.com/xiaohh-docker/nginx:1.25.4
        name: nginx
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-deploy
  name: nginx-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30080
  selector:
    app: nginx-deploy
  type: NodePort
EOF
```
生效
```
kubectl apply -f nginx-deploy.yaml
# 然后访问你任何一个节点的IP地址加上nodeport 30080就可以访问这个部署的nginx
kubectl get service -o wide
```
浏览器访问`http://192.168.1.11:30080` 提示 `Welcome to nginx!` 表示生效成功

## 6. 部署crictl(optional)
crictl 是Kubelet容器接口（CRI）的CLI和验证工具: 这里部署以下 containerd客户端crictl
```
$ VERSION="v1.28.0"
$ wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
$ sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
$ rm -f crictl-$VERSION-linux-amd64.tar.gz
$ cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
EOF
# 测试
$ crictl pods
$ crictl images
内容
docker.io/calico/apiserver                                        v3.27.2             5d4a0194d5324       94MB
docker.io/calico/cni                                              v3.27.2             bbf4b051c5078       195MB
docker.io/calico/csi                                              v3.27.2             b2c0fe47b0708       17.4MB
docker.io/calico/dikastes                                         v3.27.2             236c3ef9745d9       40.5MB
docker.io/calico/flannel-migration-controller                     v3.27.2             99b8d3ea2440b       125MB
docker.io/calico/kube-controllers                                 v3.27.2             849ce09815546       75.6MB
docker.io/calico/node-driver-registrar                            v3.27.2             73ddb59b21918       22.6MB
docker.io/calico/node                                             v3.27.2             50df0b2eb8ffe       346MB
docker.io/calico/pod2daemon-flexvol                               v3.27.2             ea79f2d96a361       15.4MB
docker.io/calico/typha                                            v3.27.2             2ec97bc370c17       68.4MB

....
```
## 完整部署脚本
以作参考,环境不同，相应参数需要手动调整
```
#！/bin/bash

# 本机名称
HOST_NAME=k8s-master01
# ipv4 地址
HOST_IP=192.168.1.11
HOST_NODE_IP=192.168.1.7

# 设置静态主机ip,手动处理 https://cloud.tencent.com/developer/article/1933335
function ch0_set_static_ip() {
echo "main ch0_set_static_ip start"
sudo apt install net-tools -y
ifconfig
route -n
sudo cp /etc/netplan/00-installer-config.yaml  /etc/netplan/00-installer-config.yaml_before
cat << EOF > /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      addresses:
      - 192.168.1.11/16
      nameservers:
        addresses:
        - 114.114.114.114
        - 8.8.8.8
        search: []
      routes:
      - to: default
        via: 192.168.1.1
  version: 2
EOF
# 使配置生效
sudo netplan apply
echo "main ch0_set_static_ip start"
}

# 1、k8s 集群主机准备
function ch1_config_hosts(){
echo "main ch1_config_hosts start"
# 部署文件目录
mkdir ${HOME}/kubernetes_install
cd ${HOME}/kubernetes_install

# 1.1 设置主机名称
hostnamectl set-hostname ${HOST_NAME}

# 1.2 所有设备允许 root 用户
cat >> /etc/ssh/sshd_config << EOF
PermitRootLogin yes
PasswordAuthentication yes
EOF

# 1.3 关闭swap 分区
swapoff -a
sed  -i '/swap/ s/^\(.*\)$/#\1/g' /etc/fstab

# 1.4 主机名和ip 地址解析配置
sed -i "2a ${HOST_IP} ${HOST_NAME}" /etc/hosts
sed -i "3a ${HOST_NODE_IP} k8s-node01" /etc/hosts


# 1.5 时间同步配置,时区
cp /etc/chrony/chrony.conf  /etc/chrony/chrony.conf_bak
cat > /etc/chrony/chrony.conf << EOF
confdir /etc/chrony/conf.d
server ntp.aliyun.com iburst
allow 192.168.1.1/24
sourcedir /run/chrony-dhcp
sourcedir /etc/chrony/sources.d
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
ntsdumpdir /var/lib/chrony
logdir /var/log/chrony
maxupdateskew 100.0
rtcsync
makestep 1 3
leapsectz right/UTC
EOF

timedatectl set-timezone Asia/Shanghai
apt install ntpdate chrony   -y
ntpdate time2.aliyun.com
systemctl start chrony
systemctl status chrony
systemctl enable chrony
# 查看 chronyd 时间同步服务的源的统计信息
chronyc sourcestats -v

# 1.6 配置内核转发及网桥过滤
# 转发IPv4并让iptables看到桥接流量
sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 手动加载
modprobe overlay
modprobe br_netfilter
# 检测是否加载
lsmod | egrep "overlay"
lsmod | egrep "br_netfilter"

# 添加网桥过滤及内核转发配置,设置所需的sysctl参数，参数在重新启动后保持不变
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

#检查sysctl是否成功应用：
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

# 手动加载
#sudo sysctl -p /etc/sysctl.d/k8s.conf
# or
sudo sysctl --system

# 1.7 安装ipset 及ipvsadm
apt install ipset ipvsadm -y

# 配置ipvsadm 内核模块加载
cat << EOF > /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

# 手动加载模块
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack

# 1.8 关闭防火墙
#查看防火墙状态
sudo systemctl status ufw
#关闭防火墙
sudo systemctl stop ufw
#禁止防火墙开机启动
sudo systemctl disable ufw

echo "main ch1_config_hosts end"
}


# 2、k8s 集群 containerd 准备
function ch2_containerd_init(){
echo "main ch2_containerd_init start"

# 部署 containerd
wget https://github.com/containerd/containerd/releases/download/v1.7.11/cri-containerd-static-1.7.11-linux-amd64.tar.gz
#tar Czxvf /usr/local/ cri-containerd-static-1.7.11-linux-amd64.tar.gz
sudo tar -zxvf cri-containerd-cni-1.7.11-linux-amd64.tar.gz -C /

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/registry.k8s.io\/pause:3.8/registry.cn-hangzhou.aliyuncs.com\/google_containers\/pause:3.9/g' /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# 使用systemd托管containerd
# 生成system service文件
cat<<EOF|tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF

# 加载文件并启动
systemctl daemon-reload
systemctl enable --now containerd
containerd --version
systemctl status containerd
echo "main ch2_containerd_init end"
}

# 3、k8s 集群部署
function ch3_k8s_cluster_install(){
# 3.1 k8s 集群软件apt 源准备
# 3.1.1 下载用于 Kubernetes 软件包仓库的公共签名密钥。所有仓库都使用相同的签名密钥，因此你可以忽略URL中的版本
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 3.1.2 添加 Kubernetes apt 仓库。 请注意，此仓库仅包含适用于 Kubernetes 1.28 的软件包
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 3.1.3 更新 apt 包索引并安装使用 Kubernetes apt 仓库所需要的包
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# 3.2 k8s 集群软件安装
# 3.2.1 更新 apt 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本(防止自动更新)：
sudo apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-cache madison kubeadm
sudo apt-mark hold kubelet kubeadm kubectl

# 3.2.2 更新 apt 包索引
# 查看软件列表(验证)
apt-cache policy kubeadm|head - 10

}

# 4. k8s master 节点初始化
function ch4_k8s_cluster_master_install(){
echo "main ch4_k8s_cluster_master_install start"

# 4.1 查看kubeadm 版本和修改配置
kubeadm version
kubeadm config print init-defaults > kube-config.yaml
sed -i "s/advertiseAddress: 1.2.3.4/advertiseAddress: ${HOST_IP}/g" kube-config.yaml
sed -i "s/name: node/name: ${HOST_NAME}/g" kube-config.yaml
sed -i "s/imageRepository: k8s.gcr.io/imageRepository: registry.cn-hangzhou.aliyuncs.com\/google_containers/g" kube-config.yaml
sed -i "s/imageRepository: registry.k8s.io/imageRepository: registry.cn-hangzhou.aliyuncs.com\/google_containers/g" kube-config.yaml
sed -i  '/serviceSubnet/a\  podSubnet: 10.244.0.0/16' kube-config.yaml

cat << EOF >> kube-config.yaml
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
EOF

# 4.2 查看kubeadm 所需要的镜像列表
kubeadm config images list
# 提前下载镜像,使用中国区镜像加速
kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers
# 查看下载的镜像，镜像保存在k8s.io namesapce 下
ctr -n=k8s.io i ls

# 4.3 使用部署配置文件初始化k8s 集群
kubeadm init --config  kube-config.yaml|tee -a kubeadm_init.log

# init 执行失败排查原因后需要reset 再次执行
# kubeadm reset
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf

echo "main ch4_k8s_cluster_master_install end"

}

# 5. worker 节点join,woker 节点操作
function ch5_k8s_cluster_woker_join(){
echo "main ch5_k8s_cluster_woker_join start"

# 5.1 k8s 集群 worker 节点join，joiin 信息从 ${HOME}/kubeadm_init.log 日志中获取
kubeadm join 192.168.1.11:6443 --token ihfbml.gd8lru2vifna04pr \
    --discovery-token-ca-cert-hash sha256:da2282a2f30c3a96b89565ff59dbd206cec1c36ff1794da81e2870cbd0862d40 \
    --cri-socket=unix:///run/containerd/containerd.sock

# join 后工作节点目前还是notready 状态,pod 目前还是pending 状态，因为是没有部署网络插件
kubectl get nodes

echo "main ch5_k8s_cluster_woker_join start"

}

# 6. 部署 calico 网络插件
function ch6_k8s_calico_install(){
echo "main ch6_k8s_calico_install start"
mkdir -p ${HOME}/kubernetes_install/deploy_calico
cd ${HOME}/kubernetes_install/deploy_calico

wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml

cat << EOF >> /etc/hosts
199.232.68.133 raw.githubusercontent.com
199.232.68.133 user-images.githubusercontent.com
199.232.68.133 avatars2.githubusercontent.com
199.232.68.133 avatars1.githubusercontent.com
EOF

# 下载客户端资源文件
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml
# 修改pod的网段地址 podSubnet
sed -i 's/cidr: 192.168.0.0/cidr: 10.244.0.0/g' custom-resources.yaml
kubectl create -f custom-resources.yaml
cd ..
echo "main ch6_k8s_calico_install end"

# 查看集群pod 创建过程，大约会耗时三分钟
watch kubectl get all -o wide -n calico-system

}

# 7. 部署 nginx 应用验证 k8s 集群的可用性
function ch7_k8s_nginx_install(){
echo "main ch7_k8s_nginx_install start"

cat << EOF > nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - image: registry.cn-shenzhen.aliyuncs.com/xiaohh-docker/nginx:1.25.4
        name: nginx
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-deploy
  name: nginx-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30080
  selector:
    app: nginx-deploy
  type: NodePort
EOF

# 如果是单master 节点则手动移除node 污点
#kubectl taint node k8s-master01 node-role.kubernetes.io/control-plane:NoSchedule-
# 添加污点
#kubectl taint node k8s-master01 node-role.kubernetes.io/control-plane:NoSchedule


kubectl apply -f nginx-deploy.yaml
# 然后访问你任何一个节点的IP地址加上nodeport就可以访问这个部署的nginx
kubectl get service -o wide

echo "main ch7_k8s_nginx_install end"
}

# 8. crictl 是Kubelet容器接口（CRI）的CLI和验证工具: 部署 containerd客户端crictl
function ch8_k8s_nginx_install(){
VERSION="v1.28.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
EOF
# 测试
crictl pods
crictl images


}

# k8s v1.28 容器运行时containerd v1.7.11, 网络插件calico v3.27.2
function main(){
   echo "main start"

   # 所有节点安装
   #ch0_set_static_ip # 设置静态ip 手动处理
   ch1_config_hosts
   #ch2_containerd_init
   #ch3_k8s_cluster_install
   # master 节点
   #ch4_k8s_cluster_master_install
   # woker 节点 手动操作
   #ch5_k8s_cluster_woker_join
   # master节点 部署calico 节点
   #ch6_k8s_calico_install
   # master 节点 部署 nginx 应用验证 k8s 集群的可用性
   #ch7_k8s_nginx_install

  # 其它必要工具部署
  ch8_k8s_nginx_install

   echo "main end"
}


main


```
