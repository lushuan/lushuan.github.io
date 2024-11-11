---
title: "Helm修改configmap后应用自动重启生效"
subtitle: ""
date: 2024-07-21T12:06:37+08:00
lastmod: 2024-07-21T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Helm"]
tags: ["Helm"]
---
## 前言
通过Helm部署应用，修改configmap后，Deployment 副本控制器不会自动生效，需要手动删除pod 重建一下才会生效，这里实践通过添加标签的方式在通过只修改configmap也能联动Deployment自动重建

## 安装应用
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
# 以grafana 应用部署为例 --untar 拉取下来后自动解压
helm pull bitnami/grafana --untar 
# 这里做创建测试，关闭持久化，不创建pvc
helm install grafana ./grafana --set persistence.enabled=false

$ helm list
NAME   	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART          	APP VERSION
grafana	default  	4       	2024-07-21 15:28:58.208948145 +0800 CST	deployed	grafana-11.3.26	11.3.0     
$ kubectl get pod
NAME                       READY   STATUS    RESTARTS   AGE
grafana-6cc56576dc-97bvw   1/1     Running   0          138m

```

## 验证
1、修改配置
```shell
vim grafana/templates/deployment.yaml
```
2、添加label
```shell
labels: {{- include "common.labels.standard" ( dict "customLabels" .Values.commonLabels "context" $ ) | nindent 4 }}
    app.kubernetes.io/component: grafana
    configmap-hash: "{{ .Values.config | toYaml | sha256sum  | trunc 63 }}"  # 添加configmap-hash
```
3、修改configmap
```shell
vim grafana/templates/configmap.yaml
```
data 内容添加一行测试数据`GF_TEST: /opt/test`
```shell
root@k8s-master01:~/helm# kubectl get cm grafana-envvars -o yaml
apiVersion: v1
data:
  GF_AUTH_LDAP_ALLOW_SIGN_UP: "false"
  GF_AUTH_LDAP_CONFIG_FILE: /opt/bitnami/grafana/conf/ldap.toml
  GF_AUTH_LDAP_ENABLED: "false"
  GF_INSTALL_PLUGINS: ""
  GF_PATHS_CONFIG: /opt/bitnami/grafana/conf/grafana.ini
  GF_PATHS_DATA: /opt/bitnami/grafana/data
  GF_PATHS_LOGS: /opt/bitnami/grafana/logs
  GF_PATHS_PLUGINS: /opt/bitnami/grafana/data/plugins
  GF_PATHS_PROVISIONING: /opt/bitnami/grafana/conf/provisioning
  GF_SECURITY_ADMIN_USER: admin
  GF_TEST: /opt/test
```
4、升级
```shell
$ helm upgrade grafana ./grafana --set persistence.enabled=false
# 可以看到容器发生了重建
$  kubectl get pod
NAME                       READY   STATUS    RESTARTS   AGE
grafana-6cc56576dc-97bvw   1/1     Running   0          145m
grafana-7b8869fd4f-6jrd5   0/1     Running   0          4s
```
## 总结
Helm借助添加标签hash化，动态更新deployment


