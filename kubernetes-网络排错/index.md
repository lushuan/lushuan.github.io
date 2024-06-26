# Kubernetes 网络排障

记录一下kuberntes 遇到的问题场景及排查方向
## DNS 解析异常
5 秒延时
如果DNS查询经常延时5秒才返回，通常是遇到内核 conntrack 冲突导致的丢包

### 解析超时
如果容器内报 DNS 解析超时，先检查下集群 DNS 服务 (kube-dns/coredns) 的 Pod 是否 Ready，如果不是，查看日志信息。如果运行正常，再具体看下超时现象。


## Service 无法解析
集群 DNS 没有正常运行(kube-dns或CoreDNS)

检查集群 DNS 是否运行正常:
- kubelet 启动参数 --cluster-dns 可以看到 dns 服务的 cluster ip:
```shell
$ ps -ef | grep kubelet
... /usr/bin/kubelet --cluster-dns=172.16.14.217 ...
```
或者放置配置文件中
```shell
$ cat /var/lib/kubelet/config.yaml|grep clusterDNS -A 2
clusterDNS:
- 172.16.0.10
clusterDomain: cluster.local
```

- 找到 dns 的 service:
```shell
$ kubectl get svc -n kube-system | grep 172.16.14.217
kube-dns              ClusterIP   172.16.14.217   <none>        53/TCP,53/UDP              47d
```

- 看是否存在 endpoint:
```shell
$ kubectl -n kube-system describe svc kube-dns | grep -i endpoints
Endpoints:         172.16.0.156:53,172.16.0.167:53
Endpoints:         172.16.0.156:53,172.16.0.167:53
```

- 检查 endpoint 的 对应 pod 是否正常:
```shell
$ kubectl -n kube-system get pod -o wide | grep 172.16.0.156
kube-dns-898dbbfc6-hvwlr            3/3       Running   0          8d        172.16.0.156   10.0.0.3
```

### Pod 与 DNS 服务之间网络不通
检查下 pod 是否连不上 dns 服务，可以在 pod 里 telnet 一下 dns 的 53 端口:
```shell
# 连 dns service 的 cluster ip
$ telnet 172.16.14.217 53
```
如果检查到是网络不通，就需要排查下网络设置:
- 检查节点的安全组设置，需要放开集群的容器网段
- 检查是否还有防火墙规则，检查 iptables



