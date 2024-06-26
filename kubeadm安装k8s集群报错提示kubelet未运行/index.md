# Kubeadm 安装k8s集群报错提示 kubelet 未运行


## 背景
通过 kubeadm 安装k8s集群报错
操作系统环境信息
```shell
$ cat /etc/os-release 
NAME="Ubuntu"
VERSION="18.04.5 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.5 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
```

kubeadm init 安装报错信息
```shell
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused.

        Unfortunately, an error has occurred:
                timed out waiting for the condition

        This error is likely caused by:
                - The kubelet is not running
                - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

        If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
                - 'systemctl status kubelet'
                - 'journalctl -xeu kubelet'

        Additionally, a control plane component may have crashed or exited when started by the container runtime.
        To troubleshoot, list all containers using your preferred container runtimes CLI.

        Here is one example how you may list all Kubernetes containers running in docker:
                - 'docker ps -a | grep kube | grep -v pause'
                Once you have found the failing container, you can inspect its logs with:
                - 'docker logs CONTAINERID'
```
## 排查思路
查看官网介绍为 docker 和 kubelet  服务中的 cgroup 驱动不一致，有两种方法  
方式一：驱动向 docker 看齐  
方式二：驱动为向 kubelet 看齐  
如果docker 不方便重启则统一向 kubelet看齐，并重启对应的服务即可

## 解决方式
### docker 配置文件
这里采取的是方式二，docker 默认驱动为 cgroupfs ,只需要添加
```
 "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
```
修改后配置文件
```
$ cat /etc/docker/daemon.json 
{
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "bip":"172.12.0.1/24",
  "registry-mirrors": [
    "http://docker-registry-mirror.kodekloud.com"
  ]
}
```
重启docker
`systemctl restart docker`
### kublete 配置文件
grep 截取一下,可以看得出来kubelet默认 cgoup 驱动为systemd
```
$ cat /var/lib/kubelet/config.yaml |grep group
cgroupDriver: systemd
```
重启kubelet （optional）
`systemctl restart kubelet`



## 参考

- [配置cgroup驱动](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#configure-cgroup-driver)  
- [Docker中的Cgroup Driver:Cgroupfs 与 Systemd](https://www.cnblogs.com/hongdada/p/9771857.html)  
- [为什么要修改docker的cgroup driver](http://www.manongjc.com/detail/20-nytgptlajikhhpx.html)
