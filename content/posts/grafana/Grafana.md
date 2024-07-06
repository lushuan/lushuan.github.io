---
title: "Grafana"
subtitle: ""
date: 2022-08-17T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Grafana"]
tags: ["Grafana"]
---
## Grafana简介
Grafana 是一个监控仪表系统，它是由 Grafana Labs 公司开源的的一个系统监测 (System Monitoring) 工具。它可以大大帮助你简化监控的复杂度，你只需要提供你需要监控的数据，它就可以帮你生成各种可视化仪表。同时它还有报警功能，可以在系统出现问题时通知你。

Grafana 本身是非常轻量级的，不会占用大量资源，此外 Grafana 需要一个数据库来存储其配置数据，比如用户、数据源和仪表盘等，目前 Grafana 支持 SQLite、MySQL、PostgreSQL 3 种数据库，默认使用的是 SQLite，该数据库文件会存储在 Grafana 的安装位置，所以需要对 Grafana 的安装目录进行持久化。

当然如果我们想要部署一个高可用版本的 Grafana 的话，那么使用 SQLite 数据库就不行了，需要切换到 MySQL 或者 PostgreSQL，我们可以在 Grafana 配置的 [database] 部分找到数据库的相关配置，Grafana 会将所有长期数据保存在数据库中，然后部署多个 Grafana 实例使用同一个数据库即可实现高可用。

### Grafana的核心功能和架构
Grafana提供了一个丰富的图表库，包括时序数据图、柱状图、饼图等多种类型，使其能够展示各种指标数据。
用户可以通过拖放的方式自定义仪表板，实现对数据的实时监控和分析。Grafana的前端界面使用AngularJS和React构建，后端则主要采用Go语言开发，确保了其高性能和灵活性。
## Grafana的安装与初步配置
Grafana支持多种安装方式，包括Docker容器、预编译的二进制包、源代码编译等，可以满足不同用户的需求。

### Docker 安装
使用Docker安装Grafana是一种快速而便捷的方法。用户只需要准备一个Docker环境，然后运行以下命令即可：
```shell
docker run -d --name grafana -p 3000:3000 grafana/grafana:9.2.20
```
此命令会下载Grafana的Docker镜像，并在容器中启动Grafana服务，监听本地的3000端口。

安装完成后，需要对Grafana进行初步配置，包括设置监听端口、配置数据库等。这些配置可以在Grafana的配置文件grafana.ini中进行。

### kubernetes 安装
kubernetes 部署并切换数据源为MySQL,这样容器重建后配置的大屏不会丢失且企业生产环境也是通过这种方式来使用，方便dashboard大屏脚本在数据库中的管理。


`grafana-configmap.yaml`，数据库中配置MySQL 数据源连接信息，Grafana 默认使用的数据库是 sqlite3,这里做一下变更
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana
  namespace: monitoring
data:
  grafana.ini: |
    #################################### Database ####################################
    [database]
    # You can configure the database connection by specifying type, host, name, user and password
    # as separate properties or as on string using the url properties.
    # Either "mysql", "postgres" or "sqlite3", it's your choice
    type = mysql
    host = 192.168.1.11:3306
    name = grafana
    user = root
    # If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
    password = abc@123A
    # Use either URL or the previous fields to configure the database
    # Example: mysql://user:secret@host:port/database
    url = mysql://root:password@192.168.1.11:3306/grafana
```

`grafana-deploy.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
    component: core
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
        component: core
    spec:
      containers:
      - image: grafana/grafana:9.2.20
        name: grafana
        imagePullPolicy: IfNotPresent
        #securityContext:
        #  runAsUser: 0
        #  fsGroup: 0
        # env:
        resources:
          # keep request = limit to keep this container in guaranteed class
          limits:
            cpu: 100m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 100Mi
        env:
          # The following env variables set up basic auth twith the default admin user and admin password.
          - name: GF_AUTH_BASIC_ENABLED
            value: "true"
          - name: GF_AUTH_ANONYMOUS_ENABLED
            value: "false"
          - name: GF_PATHS_DATA
            value: "/var/lib/grafana"
          - name: GF_PATHS_CONFIG
            value: "/etc/grafana/grafana.ini"
          # - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          #   value: Admin
          # does not really work, because of template variables in exported dashboards:
          # - name: GF_DASHBOARDS_JSON_ENABLED
          #   value: "true"
        readinessProbe:
          httpGet:
            path: /login
            port: 3000
          initialDelaySeconds: 30
          timeoutSeconds: 1
        volumeMounts:   # 数据持久化
        - name: config-volume
          mountPath: /etc/grafana
#        - name: grafana-persistent-storage
#          mountPath: /var
      volumes:
      - name: config-volume
        configMap:
          name: grafana
      #- name: grafana-persistent-storage
      #  hostPath:
      #    path: /grafana-data
      #    type: DirectoryOrCreate
      #- name: grafana-persistent-storage
      #  emptyDir: {}
```

`grafana-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
    component: core
spec:
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30009  
  selector:
    app: grafana
    component: core
```

部署
```shell
kubectl create namespace monitoring
kubectl create -f grafana-configmap.yaml
kubectl create -f grafana-deploy.yaml
kubectl create -f grafana-service.yaml
```

部署成功

```shell
# kubectl -n monitoring get pod,svc
NAME                           READY   STATUS    RESTARTS   AGE
pod/grafana-5d8c7d5c8c-znffb   1/1     Running   0          6m18s

NAME              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/grafana   NodePort   10.110.17.117   <none>        3000:30009/TCP   11m
```

成功部署后，数据库会反向生成相关的表信息

![grafan-1](/images/grafana/grafana-1.png)

## 访问功能界面
`http://ip:30009` 登录是默认账号口令为'admin/admin',第一次登录时提示修改密码
![grafan-2](/images/grafana/grafana-2.png)

登录成功界面

![grafan-3](/images/grafana/grafana-3.png)

## 数据源深入集成
![grafan-4](/images/grafana/grafana-4.png)

在Grafana中，数据源的集成是构建有效监控和分析系统的关键步骤。Grafana支持众多流行的数据存储和监控工具作为数据源，包括时序数据库Prometheus, InfluxDB，日志和文档存储如Elasticsearch，以及传统的SQL数据库如MySQL和PostgreSQL。

## 导入 Dashboard
为了能够快速对系统进行监控，我们可以直接复用别人的 Grafana Dashboard，在 Grafana 的官方网站上就有很多非常优秀的第三方 Dashboard，我们完全可以直接导入进来即可。
比如我们想要对所有的集群节点进行监控，也就是 node-exporter 采集的数据进行展示，这里我们就可以导入 https://grafana.com/grafana/dashboards/8919 这个 Dashboard。

在侧边栏点击 "+"，选择 Import，在 Grafana Dashboard 的文本框中输入 8919 即可导入：

![grafan-5](/images/grafana/grafana-5.png)

{{< admonition >}}
在导入之前需要先创建数据源，大屏是适配的Prometheus 数据源
{{< /admonition >}}

## Grafana 相关组件概念了解
### 面板介绍

面板（Panel）是 Grafana 中基本可视化构建块，每个面板都有一个特定于面板中选择数据源的查询编辑器，每个面板都有各种各样的样式和格式选项，面板可以在仪表板上拖放和重新排列，它们也可以调整大小。

Panel 是 Grafana 中最基本的可视化单元，每一种类型的面板都提供了相应的查询编辑器(Query Editor)，让用户可以从不同的数据源（如 Prometheus）中查询出相应的监控数据，并且以可视化的方式展现，Grafana 中所有的面板均以插件的形式进行使用。

目前内置支持的面板包括：Time series（时间序列）是默认的也是主要的图形可视化面板、State timeline（状态时间表）状态随时间变化 、Status history（状态历史记录）、Bar chart（条形图）、Histogram（直方图）、Heatmap（热力图）、Pie chart（饼状图）、Stat（统计数据）等

- 图表 Time series 
- 数据统计 stat
- 仪表盘 gauge
- 表格 table
- 饼状图 Pie Chart
- 热图 Heatmap
###  图形面板

- 数据源（Data Source）
- 添加面板（Add Panel）
- 添加参数（Dashboard settings）

### 图形定制

- 多个查询(query editor)
- [转换(Transform)](https://grafana.com/docs/grafana/latest/panels-visualizations/query-transform-data/transform-data/)
- 模板变量

### 文本面板
```
# 这是一级标题

## 这是二级标题

这是正文内容，**我是加粗**，`我是强调`。

> 还可以添加引用的内容

![这是一张图片](https://p8s.io/docs/assets/img/illustration.png)
```
渲染后的图片
![grafana-6](/images/grafana/grafana-6.png)

### 其它
#### 安全与维护

- 用户认证与授权
- 数据备份与恢复
#### 注释
Grafana 中提供了一个 Annotations 注释的方法，通过该方式可以在图形上标记一个点，当将鼠标悬停在上面的时候，可以获得该注释的具体信息，我们可以在 Panel 面板中某一个点添加注释，也可以在某一个区域内添加注释。
#### 告警
Grafana 除了支持丰富的数据源和图表功能之外，还支持告警功能，该功能也使得 Grafana 从一个数据可视化工具成为了一个真正的监控利器。Grafana 可以通过 Alerting 模块的配置把监控数据中的异常信息进行告警，告警的规则可以直接基于现有的数据图表进行配置，在告警的时候也会把出现异常的图表进行通知，使得我们的告警通知更加友好。

## 参考
- [Grafana 官网](https://grafana.com/docs/grafana/latest/)
- [Grafana Dashboard模板-官网](https://grafana.com/grafana/dashboards/?pg=hp&plcmt=lt-box-dashboards)
