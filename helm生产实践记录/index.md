# Helm 生产实践记录

记录一下helm 在生产中的实践

Helm 可以帮助我们管理 Kubernetes 应用程序 - Helm Charts 可以定义、安装和升级复杂的 Kubernetes 应用程序，Charts 包很容易创建、版本管理、分享和分布。Helm 对于 Kubernetes 来说就相当于 yum 对于 Centos 来说，如果没有 yum 的话，我们在 Centos 下面要安装一些应用程序是极度麻烦的，同样的，对于越来越复杂的 Kubernetes 应用程序来说，如果单纯依靠我们去手动维护应用程序的 YAML 资源清单文件来说，成本也是巨大的。

## 安装
由于 Helm V2 版本必须在 Kubernetes 集群中安装一个 Tiller 服务进行通信，这样大大降低了其安全性和可用性，所以在 V3 版本中移除了服务端，采用了通用的 Kubernetes CRD 资源来进行管理，这样就只需要连接上 Kubernetes 即可，而且 V3 版本已经发布了稳定版

这里使用的是helm v3版本
## 示例
Helm 其实就是读取的 kubeconfig 文件来访问集群的。


```shell
# 查看版本
$ helm version
version.BuildInfo{Version:"v3.8.1", GitCommit:"5cb9af4b1b271d11d7a97a71df3ac337dd94ad37", GitTreeState:"clean", GoVersion:"go1.17.5"}
# 添加一个 chart 仓库
$ helm repo add stable http://mirror.azure.cn/kubernetes/charts/
"stable" has been added to your repositories
$ helm repo list
NAME    URL
stable  http://mirror.azure.cn/kubernetes/charts/

# 用 search 命令来搜索可以安装的 chart 包，已经集成了将近300个可安装的chart包
zoms@crms-10-10-192-220[/tmp]$ helm search repo stable|more
NAME                                    CHART VERSION   APP VERSION             DESCRIPTION
stable/acs-engine-autoscaler            2.2.2           2.1.1                   DEPRECATED Scales worker nodes within agent pools
stable/aerospike                        0.3.5           v4.5.0.5                DEPRECATED A Helm chart for Aerospike in Kubern...
stable/airflow                          7.13.3          1.10.12                 DEPRECATED - please use: https://github.com/air...

# 安装之前先进行更新
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈

# 安装一个mysql 应用
helm install stable/mysql --generate-name

# 了解 MySQL 这个 chart 包的一些特性
helm show chart stable/mysql

# 了解 MySQL 这个 chart 包更多的特性
helm show all stable/mysql

# 查看已安装的release
helm ls

# 删除release
helm uninstall mysql-1575619811

# 删除release 时保留历史记录，方便取消删除 release（使用 helm rollback 命令）
helm uninstall mysql-1575619811 --keep-history

```

## 定制
helm show values 命令来查看一个 chart 包的所有可配置的参数选项

查看本地chart包可配置的参数选项
```shell
$ helm show values ./nms-grafana/ -n zcm9
# Default values for nms-grafana.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates

fullnameOverride: nms-grafana

replicaCount: 1

restartPolicy: Always
....
```

查看仓库chart包的所有可配置的参数选项,安装之前先看下镜像源是否可先pull下来，以及镜像的版本号

```shell
$ helm show values stable/mysql
## mysql image version
## ref: https://hub.docker.com/r/library/mysql/tags/
##
image: "mysql"
imageTag: "5.7.30"

strategy:
  type: Recreate

busybox:
  image: "busybox"

```

上面我们看到的所有参数都是可以用自己的数据来覆盖的，可以在安装的时候通过 YAML 格式的文件来传递这些参数
```shell
$ cat config.yaml
mysqlUser:
  user0
mysqlPassword: user0pwd
mysqlDatabase: user0db
persistence:
  enabled: false
$ helm install -f config.yaml mysql stable/mysql
NAME: mysql
LAST DEPLOYED: Fri Dec  6 17:46:56 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
mysql.default.svc.cluster.local
```

release 安装成功后查看对应的Pod信息
```shell
$ kubectl get pod -l release=mysql
NAME                    READY   STATUS            RESTARTS   AGE
mysql-ddd798f48-gnrzd   0/1     PodInitializing   0          119s
$ kubectl describe pod  mysql-ddd798f48-gnrzd
......
Environment:
      MYSQL_ROOT_PASSWORD:  <set to the key 'mysql-root-password' in secret 'mysql'>  Optional: false
      MYSQL_PASSWORD:       <set to the key 'mysql-password' in secret 'mysql'>       Optional: false
      MYSQL_USER:           user0
      MYSQL_DATABASE:       user0db
......
```
可以看到环境变量 MYSQL_USER=user0，MYSQL_DATABASE=user0db 的值和我们上面配置的值是一致的

在安装过程中，有两种方法可以传递配置数据：
- --values（或者 -f）：指定一个 YAML 文件来覆盖 values 值，可以指定多个值，最后边的文件优先
- --set：在命令行上指定覆盖的配置



## 更多安装方式
helm install 命令可以从多个源进行安装：

- chart 仓库（类似于上面我们提到的）
- 本地 chart 压缩包（helm install foo-0.1.1.tgz）
- 本地解压缩的 chart 目录（helm install foo path/to/foo）
- 在线的 URL（helm install fool https://example.com/charts/foo-1.2.3.tgz）


## 升级和回滚
当新版本的 chart 包发布的时候，或者当你要更改 release 的配置的时候，你可以使用 helm upgrade 命令来操作。升级需要一个现有的 release，并根据提供的信息对其进行升级。因为 Kubernetes charts 可能很大而且很复杂，Helm 会尝试以最小的侵入性进行升级，它只会更新自上一版本以来发生的变化：
```shell
$ helm upgrade -f config.yaml mysql stable/mysql
Release "mysql" has been upgraded. Happy Helming!
NAME: mysql
LAST DEPLOYED: Fri Dec  6 21:06:11 2019
NAMESPACE: default
STATUS: deployed
REVISION: 2
...
```

查看升级后的新设置是否生效

```shell
$ helm get values mysql
USER-SUPPLIED VALUES:
mysqlDatabase: user0db
mysqlPassword: user0pwd
mysqlRootPassword: passw0rd
mysqlUser: user0
persistence:
  enabled: false
```

另外如生产基本上是只替换镜像tag
```shell
$ sudo helm get values nms-grafana -n zcm9
USER-SUPPLIED VALUES:
global:
  nodeSelector:
    aik: zcm9
  repository: 10.10.192.220:52800
nms-activemq:
  image:
    tag: C_20220628200611
nms-grafana:
  image:
    tag: C_20220726192751-v9.0.1
...
```

查看当前release 当前版本信息以及查看某个release 的历史
```shell
$ sudo helm -n zcm9 ls
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
minio                   zcm9            1               2022-08-11 19:27:49.792207699 +0800 CST failed          minio-10.0.4                    2022.1.8
nms-activemq            zcm9            1               2022-11-09 13:31:42.996290693 +0800 CST deployed        nms-activemq-0.1.0              9.0.44
nms-grafana             zcm9            29              2023-05-04 20:34:46.578214434 +0800 CST deployed        nms-grafana-0.1.0               aik
...

$ sudo helm -n zcm9  history nms-grafana
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION
20              Thu May  4 19:52:14 2023        superseded      nms-grafana-0.1.0       aik             Upgrade complete
21              Thu May  4 19:53:14 2023        superseded      nms-grafana-0.1.0       aik             Upgrade complete
22              Thu May  4 19:58:59 2023        superseded      nms-grafana-0.1.0       aik             Upgrade complete
...
```

回滚
```shell
# 上面可知本地安装的应用nms-grafana 迭代至了版本29,如果要回滚至指定版本可以通过一下方式
$ helm -n zcm9 rollback nms-grafana  21
```



