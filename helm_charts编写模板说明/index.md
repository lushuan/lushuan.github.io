# Helm charts 编写模板说明

## 预置条件

适用于Kubernetes v1.12.1及以上，Helm v3.4.2及以上。

以`test-cmdb`应用为例进行说明,`test-cmdb`为部署在k8s中的web 应用

## 目录结构
```
test-cmdb  
├── Chart.yaml  
├── files  
│   └── application.properties  
├── templates  
│   ├── configmap.yaml  
│   ├── deployment.yaml  
│   ├── _helpers.tpl  
│   ├── ingress.yaml  
│   ├── NOTES.txt  
│   └── service.yaml  
└── values.yaml  
​  
2 directories, 9 files
```
目录及文件描述：

| 类型 | 名称 | 描述 |
| ---- | ---- | ---- |
| D | test-cmdb | 应用名称，根据实际业务应用进行命名。 |
| F | Chart.yaml | 文件包含了该chart的描述。 |
| D | files | 配置文件目录 |
| F | application.properties | 配置文件 |
| D | templates | 目录包括了模板文件。当Helm评估chart时，会通过模板渲染引擎将所有文件发送到templates/目录中。 然后收集模板的结果并发送给Kubernetes。 |
| F | configmap.yaml | configmap模板 |
| F | deployment.yaml | deployment模板 |
| F | _helpers.tpl | 放置可以通过chart复用的模板辅助对象 |
| F | ingress.yaml | ingress模板。 |
| F | NOTES.txt | chart的"帮助文本"。会在用户执行helm install时展示。 |
| F | service.yaml | service模板。 |
| F | values.yaml | chart的默认值。会在用户执行helm install 或 helm upgrade时被覆盖。 |

其中，类型：D，目录；F，文件。

## 配置资源

一个完整的应用部署通常由应用名称、镜像名称及tag、环境变量、资源限额、端口监听、端口映射、目录/文件挂载、配置文件、网关规则等组成。通过helm将应用模板化，仅需要简单配置些许地方即可。

* 应用名称
    
    | 文件名 | key |
    | ---- | ---- |
    | Chart.yaml | name |
    | values.yaml | fullnameOverride |
    
* 镜像名称及tag
    
    | 文件名 | key |
    | ---- | ---- |
    | values.yaml | image.repository |
    | values.yaml | image.tag |
    
    当前的模板默认是以镜像名称与应用名称保持一致为前提的，如镜像`10.45.80.1/iplatform/test-cmdb:C_20220214144619`的名称为`test-cmdb`与应用名称`test-cmdb`保持一致。假如实际的应用名与镜像名称不一致，请自行修改`_helpers.tpl`中的fullimage部分。
    
    修改前
    ```
    {{- define "zcm-tpl.fullimage" -}}  
    {{- .Values.global.repository }}/{{ .Values.image.repository }}:{{ index .Values .Chart.Name "image" "tag" | default .Chart.AppVersion }}  
    {{- end }}
    ```
    修改后
    ```
    {{- define "zcm-tpl.fullimage" -}}  
    {{- .Values.global.repository }}/{{ .Values.image.repository }}:{{ index .Values "name-template" "image" "tag" | default .Chart.AppVersion }}  
    {{- end }}
    ```
    其中，`name-template`为实际的镜像名称，该镜像信息配置在`$HELM_PATH/charts/values.yaml`中，如果该文件中没有匹配的信息，当在部署时会提示异常。
    
* 环境变量
    
    | 文件名 | key |
    | ---- | ---- |
    | values.yaml | env.name |
    | values.yaml | env.value |
    | values.yaml | env.valueFrom.configMapKeyRef.name |
    | values.yaml | env.valueFrom.configMapKeyRef.key |
    
* 资源限额
    
    | 文件名 | key |
    | ---- | ---- |
    | values.yaml | resources.requests.cpu |
    | values.yaml | resources.requests.memory |
    | values.yaml | resources.limits.cpu |
    | values.yaml | resources.limits.memory |
    
* 端口监听
    
    | 文件名 | key |
    | ---- | ---- |
    | values.yaml | ports.containerPort |
    | values.yaml | ports.protocol |
    | values.yaml | ports.name |
    
* 端口映射
    
    | 文件名 | key |
    | ---- | ---- |
    | values.yaml | service.ports.name |
    | values.yaml | service.ports.port |
    | values.yaml | service.ports.targetPort |
    
* 目录/文件挂载
    
    | 文件名 | key |
    | ---- | ---- |
    | values.yaml | volumeMounts.name |
    | values.yaml | volumeMounts.mountPath |
    | values.yaml | volumeMounts.readOnly |
    | values.yaml | volumeMounts.subPath |
    | values.yaml | volumes.name |
    | values.yaml | volumes.hostpath.path |
    
* 配置文件
    
    | 文件名 | key |
    | ---- | ---- |
    | values.yaml | cm.name |
    | values.yaml | cm.configmap.name |
    
* 网关规则

    | 文件名 | key |
    | ---- | ---- |
    | values.yaml | ingress.annotations |
    | values.yaml | ingress.tls |
    | values.yaml | ingress.hosts.paths.path |
    | values.yaml | ingress.hosts.paths.pathType |
    | values.yaml | ingress.hosts.paths.port |

## 部署测试

将准备好的chart放到ZCM部署主机`$HELM_PATH/charts`目录下，镜像推送到对应环境的镜像仓库中，并在`$HELM_PATH/charts/values.yaml`中配置镜像信息，
如镜像`10.45.80.1/iplatform/zcm-crms:T_20220411015401`，需配置对应的信息如下：
```
zcm-crms:  
  image:  
    tag: T_20220411015401
```
执行部署操作
```
cd $HELM_PATH/charts  
helm install zcm-crms ./zcm-crms -f values.yaml -n backend
```
也可执行其他相关操作

升级
```
helm upgrade zcm-crms ./zcm-crms -f values.yaml -n backend
```
卸载
```
helm uninstall zcm-crms -n backend
```
