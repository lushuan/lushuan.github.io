# Kubernetes 记录一次 Apiserver 和 Etcd 连接的随机端口占用冲突问题


## 背景
项目现场通过 kubeadm 部署一套 k8s 临时环境 版本是v1.22, 因为是临时环境，部署方式是一主两从的模式，
部署后十天客户直接将主节点的内存进行了扩容，导致现场的应用服务无法访问。

## 现象
打开访问应用门户，提示 nginx 网关访问报错
`503 Service Temporarily Unavailable`

## 定位
通过 kubectl 查看 pod 状态发现很多应用是 CrashLoopBackOff,猜测是系统基础组件出现异常,查看应用日志提示
jdbc 连接 MySQL 异常，查看MySQL 日志发现端口52306被占用。

排查为 apiserver和etcd连接的随机端口占用了52306
```yaml
# sudo netstat -nap|grep 52306
tcp        0      0 127.0.0.1:52306         127.0.0.1:2379          ESTABLISHED 30166/kube-apiserve
tcp        0      0 127.0.0.1:2379          127.0.0.1:52306         ESTABLISHED 23788/etcd
```

## 问题处理
1. 确认端口占用情况：执行 sudo netstat -lnp | grep <port> 命令查看指定端口是否被占用
2. 释放端口：如果该端口已被占用，可以通过 sudo lsof -i:<port> 命令查找占用端口的进程
3. 临时处理，pod 重建 kill 掉 etcd 的静态 pod，端口可能还是会重新被占用
4. 永久处理，主机 k8s.conf 预留端口

### 保留预留端口
```shell
$ cat /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
kernel.core_pattern=/tmp/zcore/core.%h~%e
fs.inotify.max_user_watches = 1048576
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_local_reserved_ports = 52000-52999
```
`net.ipv4.ip_local_reserved_ports`新增预留端口配置，保留端口范围不被占用，告诉内核保留端口范围从 52000 到 52999，以便这些端口不会被普通应用程序占用
## /etc/sysctl.d/k8s.conf 配置项说明
在 Kubernetes 1.22 版本的 /etc/sysctl.d/k8s.conf 配置文件中，可能包含以下一些配置项：

1、 net.bridge.bridge-nf-call-ip6tables：

- 含义：控制是否将 IPv6 数据包传递给 iptables 的 netfilter 框架进行处理。如果设置为 1，则表示启用；如果设置为 0，则表示禁用。
- 默认值：1

2、 net.bridge.bridge-nf-call-iptables：

- 含义：控制是否将数据包传递给 iptables 的 netfilter 框架进行处理。如果设置为 1，则表示启用；如果设置为 0，则表示禁用。
- 默认值：1

3、 net.ipv4.ip_forward：

- 含义：控制 Linux 内核是否允许 IP 数据包转发。如果设置为 1，则表示启用 IP 转发；如果设置为 0，则表示禁用 IP 转发。
- 默认值：0

4、 net.ipv4.conf.all.forwarding：

- 含义：控制所有网络接口的 IP 转发功能。如果设置为 1，则表示启用 IP 转发；如果设置为 0，则表示禁用 IP 转发。
- 默认值：0  

5、 net.ipv4.conf.default.forwarding：

- 含义：控制默认网络接口的 IP 转发功能。如果设置为 1，则表示启用 IP 转发；如果设置为 0，则表示禁用 IP 转发。
- 默认值：0

以上是常见的示例配置项和默认值，但实际的配置项和默认值可能会根据操作系统和 Kubernetes 版本而有所不同。在实际使用中，请根据文档和操作系统的要求进行正确配置。
