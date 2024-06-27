# Kubernetes ConfigMap多文件挂载至同一个pod内目录实践

## 理解ConfigMap
为了能够准确和深刻理解Kubernetes ConfigMap的功能和价值，我们需要从Docker说起。我们知道，Docker通过将程序、依赖库、数据及配置文件“打包固化”到一个不变的镜像文件中的做法，解决了应用的部署的难题，但这同时带来了棘手的问题，即配置文件中的参数在运行期如何修改的问题。我们不可能在启动Docker容器后再修改容器里的配置文件，然后用新的配置文件重启容器里的用户主进程。为了解决这个问题，Docker提供了两种方式：

在运行时通过容器的环境变量来传递参数；
通过Docker Volume将容器外的配置文件映射到容器内。
这两种方式都有其优势和缺点，在大多数情况下，后一种方式更合适我们的系统，因为大多数应用通常从一个或多个配置文件中读取参数。但这种方式也有明显的缺陷：我们必须在目标主机上先创建好对应的配置文件，然后才能映射到容器里。上述缺陷在分布式情况下变得更为严重，因为无论采用哪种方式，写入（修改）多台服务器上的某个指定文件，并确保这些文件保持一致，都是一个很难完成的目标。此外，在大多数情况下，我们都希望能集中管理系统的配置参数，而不是管理一堆配置文件。针对上述问题，Kubernetes给出了一个很巧妙的设计实现，如下所述。

首先，把所有的配置项都当作keyvalue字符串，当然value可以来自某个文本文件，比如配置项password=123456、user=root、host=192.168.8.4用于表示连接FTP服务器的配置参数。这些配置项可以作为Map表中的一个项，整个Map的数据可以被持久化存储在Kubernetes的Etcd数据库中，然后提供API以方便Kubernetes相关组件或客户应用CRUD操作这些数据，上述专门用来保存配置参数的Map就是Kubernetes ConfigMap资源对象。

## 痛点改进
接下来，Kubernetes提供了一种内建机制，将存储在etcd中的ConfigMap通过Volume映射的方式变成目标Pod内的配置文件，不管目标Pod被调度到哪台服务器上，都会完成自动映射。进一步地，如果ConfigMap中的key/value数据被修改，则映射到Pod中的“配置文件”也会随之自动更新。于是，Kubernetes
ConfigMap就成了分布式系统中最为简单（使用方法简单，但背后实现比较复杂）且对应用无侵入的配置中心。

## 实践
需求
将一个configmap 内的两个文件挂载在一个pod的不同目录下

- /app/conf/grafanaoracle.properties
- /app/start-timeshift.sh
创建一个configmap
```yaml
apiVersion: v1
data:
  grafanaoracle.properties: |
    name=kevin
    nage=19
  start-timeshift.sh: |
    #!/bin/sh
    cd /app/influxdb-timeshift-proxy
    INFLUXDB=nms-influxdb:8086 /usr/bin/npm run start
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/managed-by: Helm
  name: test-config

```
创建一个 deploy
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    zcm-app: test-configmap
  name: test-configmap
spec:
  replicas: 1
  selector:
    matchLabels:
      zcm-app: test-configmap
  template:
    metadata:
      labels:
        zcm-app: test-configmap
    spec:
      containers:
      - env:
        - name: CLOUD_APP_NAME
          value: paas_test-configmap
        image: nginx
        imagePullPolicy: IfNotPresent
        name: test-configmap
        ports:
        - containerPort: 9999
          name: http-oracle
          protocol: TCP
        volumeMounts:       # 关键配置 开始
        - mountPath: /app/start-timeshift.sh
          name: properties
          readOnly: true
          subPath: start-timeshift.sh
        - mountPath: /app/conf/grafanaoracle.properties
          name: properties
          readOnly: true
          subPath: grafanaoracle.properties
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: grafanaoracle.properties  # key 和 path 同名即可
            path: grafanaoracle.properties
          - key: start-timeshift.sh
            path: start-timeshift.sh
          name: test-configmap
        name: properties 	# 关键配置 结束

```
