---
title: "一文了解 Kubernetes RBAC 机制和实践"
subtitle: ""
date: 2024-04-27T12:06:37+08:00
lastmod: 2024-05-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
---

## 前言 
在了解RBAC 之前先了解一下Kubernetes API 对象，RBAC 最终操作的对象还是Kubernetes API 对象，本质还是对Kubernetes API的访问控制，如下RBAC 是在 Api Server 中是通过授权插件默认引入的。
` /etc/kubernetes/manifests/kube-apiserver.yaml `
```
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.1.11
    - --secure-port=6443   # api server 监听的安全端口(https 协议)
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC             # 启用RBAC 和Node 授权插件
    - --client-ca-file=/etc/kubernetes/pki/ca.crt # 启动x509 数字证书验证
    - --enable-admission-plugins=NodeRestriction  # 额外启用的准入控制器列表
    - --enable-bootstrap-token-auth=true	# 启用bootstrap 的token 认证
    - --requestheader-extra-headers-prefix=X-Remote-Extra- # 代理认证相关的配置
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - ...
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub # 指定用于签署承载令牌的秘钥文件

```
### Kubernetes API 概念
Kubernetes API 是通过 HTTP 提供的基于资源 (RESTful) 的编程接口。 它支持通过标准 HTTP 动词（POST、PUT、PATCH、DELETE、GET）检索、创建、更新和删除主要资源。

对于某些资源，API 包括额外的子资源，允许细粒度授权（例如：将 Pod 的详细信息与检索日志分开）， 为了方便或者提高效率，可以以不同的表示形式接受和服务这些资源。

### 资源 URI
- /api/v1/namespaces
- /api/v1/pods
- /api/v1/namespaces/my-namespace/pods
- /apis/apps/v1/deployments
- /apis/apps/v1/namespaces/my-namespace/deployments
- /apis/apps/v1/namespaces/my-namespace/deployments/my-deployment

`kubectl get --raw`命令查看
```shell
# kubectl get --raw /
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/apiextensions.k8s.io",
    ...
}
```
### 启动代理
代理一个本地的api server 接口
```shell
# kubectl proxy
Starting to serve on 127.0.0.1:8001

```
然后就可以直接访问api server 了，如新开一个会话查看集群中的daemonset 资源信息，这只是一个测试，正常的开发不会通过如下方式进行测试，而是有专门的sdk 如`client-go`
```shell
curl http://127.0.0.1:8001/apis/apps/v1/daemonsets
```

查看kubenetes api 资源类型，资源类型是否namespaced 还是集群范围的
```shell
$ kubectl api-resources
NAME                              SHORTNAMES                                      APIVERSION                             NAMESPACED   KIND
bindings                                                                          v1                                     true         Binding
componentstatuses                 cs                                              v1                                     false        ComponentStatus
configmaps                        cm                                              v1                                     true         ConfigMap
endpoints                         ep                                              v1                                     true         Endpoints
events                            ev                                     
....
```

下面就进入正体RBAC,以上的介绍就是为了说明RBAC 最终的操作还是kubernetes api 对象，当然也不止是kubernetes api 各类资源对象，还有非资源类型的URL.
## RBAC 授权模型

kubernetes 系统的RBAC 授权插件将角色分为Role和ClusterRole两类，它们都是kubernetes 内置支持的API 资源类型，其中role 作用于名称空间级别，用于承载名称空间内的资源权限集合，而clusterrole 则能够同时承载名称空间和集群级别的资源权限集合。

利用Role 和ClusterRole 两类角色进行赋权时，需要用到另外两种资源RoleBinding和ClusterRoleBinding,它们同样是由Apiserver内置支持的资源类型。

### Kubernetes RBAC 是什么？
RBAC 是一个特定的权限管理模型，它把可以施加在"资源对象"上的"动作''称为"许可权限",这些许可权限能按需组合在一起构建出"角色"及其职能，并通过为"用户账户和组账户"分配到一个到多个角色完成权限委派。这些能够发出动作的用户在RBAC中称为"主体(subject)"

![RBAC中的用户、角色、权限](/images/kubernetes/rbac/rbac-2.png "RBAC中的用户、角色、权限")

简单来说，RBAC 就是一种访问控制模型，它以角色为中心界定"谁"(subject)能够"操作"(verb)哪个或哪类"对象"(object)。

动作发出者即"主体"，通常以"账号"为载体，在kubernetes 中，它可以是普通账户(user),也可以是服务账户(service account)."动作"表示要执行的具体操作，包括创建、删除、修改和查看等行为，对于API Server来说，即PUT、POST、DELETE、GET等请求方法。而"对象"则是指管理操作能够施加的目标实体，对 kubernetes API 来说主要指各类资源对象以及非资源类型URL.


### 为什么RBAC很重要，有什么优势?
RBAC是基于角色的访问控制，是一种基于个人用户的角色来管理对计算机或网络资源的访问的方法。

相对于其他授权模式，RBAC具有如下优势：
- 对集群中的资源和非资源权限均有完整的覆盖。
- 整个RBAC完全由几个API对象完成， 同其他API对象一样， 可以用kubectl或API进行操作。
- 可以在运行时进行调整，无须重新启动API Server。

####  kubernetes 访问控制
![kubernetes 访问控制流程图](/images/kubernetes/rbac/kubernetes-access-control-diagram.png "用户、服务账户、认证、授权和准入控制")
### 设计
![RBAC 设计](/images/kubernetes/rbac/rbac-4.jpg "RBAC Design")

### RBAC 基础概念术语
#### 术语
- **Subjects**: 主体，谁可以访问kubernetes API.
- **Resources**: 哪些 kubernetes API 资源对象可以被访问.
- **Verbs**: 可以对这些 Kubernetes API 做哪些操作(CRUD)

可以简单的说RBAC是一种身份和访问管理形式，它涉及一组权限或模板，这些权限或模板决定谁(Subjects)可以执行什么(Verbs)，以及在哪里(命名空间或集群)。

补充：

- Subject:主体，对应集群中尝试操作的对象，集群中定义了 3 种类型的主体资源:
  - User Account:用户，这是有外部独立服务进行管理的，管理员进行私钥的分配，用户可以使用KeyStone 或者 Goolge 帐号，甚至一个用户名和密码的文件列表也可以。对于用户的管理集群内部没有一个关联的资源对象，所以用户不能通过集群内部的 API 来进行管理
  - Group:组，这是用来关联多个账户的，集群中有一些默认创建的组，比如 cluster-admin
  - Service Account: 服务帐号

#### Service Account
ServiceAccount 为 Pod 中运行的进程提供了一个身份，Pod 内的进程可以使用其关联服务账号的身份，向集群的 APIServer 进行身份认证。
通过下图可以了解Service Account 的位置和作用

![Service Account](/images/kubernetes/rbac/rbac-7.png "Service Account")


#### Role 和 ClusterRole
Kubernetes 系统的RBAC 授权插件将角色分为Role 和 ClusterRole 两类，它们都是kubernetes 内置支持的API 资源类型，其中Role 用作于空间级别，用于承载名称空间内的资源权限集合，而ClusterRole则能够同时承载名称空间和集群级别的资源权限集合。

Role 无法承载集群级别的资源类型的操作权限，这类的资源包括集群级别的资源(例如Node)、非资源类型的端点(例如/healthz),以及作用于所有名称空间的资源(例如跨名称空间获取任何资源的权限)等。

查看不属于任何特定的命名空间，而是在整个集群范围内管理和配置的API 资源对象
```shell
$ kubectl api-resources --namespaced=false
```
#### Rule
规则，规则是一组属于不同 API Group 资源上的一组操作的集合,在Role 和 ClusterRole资源上定义的rule 也称为PolicyRule,即策略规则，它可以内嵌的字段有如下几个
- apiGroups
- resources
- resourceNames
- nonResourceURLs
- verbs

#### RoleBinding 和 ClusterRoleBinding
利用Role 和 ClusterRole 进行角色赋权时，需要用到另外两种资源RoleBinding和ClusterRoleBinding,它们同样是API Server 内置的支持的资源类型。
- RoleBinding 用于将Role 绑定到一个或一组用户之上，它隶属于且仅能作用于其所在的单个名称空间。
- RoleBinding 可以引用同一名称空间中的role,也可以引用集群级别的ClusterRole,但是引用ClusterRole的许可权限会降级到仅能在Rolebingding所在的名称空间生效。
- ClusterRoleBinding 则用于将ClusterRole 绑定到用户和组，它作用于集群全局，且仅能够引用ClusterRole.

简单来说就是把声明的Subject 和我们的 Role 进行绑定的过程(给某个用户绑定上操作的权限)，二者的区别也是作用范围的区别:RoleBinding 只会影响到当前 namespace 下面的资源操作权限，而ClusterRoleBinding 会影响到所有的 namespace。

四者关系如图：

![角色和角色绑定](/images/kubernetes/rbac/rbac-3.png "Role、RoleBinding、ClusterRole和ClusterRoleBinding")

#### Service Account 和 token 令牌
注：sa 是service account 的缩写
##### kubernetes <= 1.20 版本
sa 创建后会自动创建一个secret,该secret 会包含一个token,这个token 就是serviceaccount的令牌，而且是永久有效的

#####  kubernetes >= 1.21 && <= 1.23 版本
sa 创建后依然会自动创建一个secret,但是pod 里面使用的sa token不是该secret里面包含的令牌了，而是kubelet 去tokenrequest api 请求的有时间限制的token

#####  kubernetes >= 1.24
sa 创建后不会自动创建一个secret 进行关联，然后和前面这个方式一样，通过tokenrequest api 请求token，每个一小时会自动刷新一次令牌，pod 容器内令牌路径为`/var/run/secrets/kubernetes.io/serviceaccount/token`

```shell
$ kubectl exec -it nginx-deploy-5947669f9-9h9vg -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6IjlsalZmUVpQLXEzRjhLSk9hMnpOV0FKWEJMT2RGam15MDIyR1BlMXNkSjAifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzUyMjIxMTQ1LCJpYXQiOjE3MjA2ODUxNDUsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueC1kZXBsb3ktNTk0NzY2OWY5LTloOXZnIiwidWlkIjoiMThlODJmZTUtNzFjZS00ZTBmLWEwYTItNTNlYWJhYjRjNzgxIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJkZWZhdWx0IiwidWlkIjoiNzFlYjM5MzEtMWU5Ny00MGI0LWFmNTItODdiZWYwZjQ2MjVhIn0sIndhcm5hZnRlciI6MTcyMDY4ODc1Mn0sIm5iZiI6MTcyMDY4NTE0NSwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGVmYXVsdCJ9.0VZaQGqg1bCxzBIaRMXQ3doYWsu6g4PYsnTPoIDjCuF4IufFq-eG9YppkhjR21cfg-SJpV4h1maW1axB3yevpf9PYFxuEDRrLY4_qpkJW5UR-r_nz0D99iiQCisb2znrDSjChGFpUnStQ1TMgQfkejPgRh08RqKLNJE1QhQ6dncebALBmWeXEdi2vWkXUOT9KlvZxP7Bk2gYaQH39PhxiHb_b2nLSPwfiVNvK0Lpxe54oCQE4WH3qSWTg0GQ0nLn74E0fgSkyegBJhmPzyPcJ-iR8Je5qL6SKyNjHftu1m49q40RrNkeh0ogLBcD0qobpMWWw-XvtiK-bcwQt1qgLA
$
```
通过[jwt](https://jwt.io/)令牌解析查看

![Service Account 和 Token](/images/kubernetes/rbac/rbac-4.png  "Service Account 和 Token")

最新版本的令牌是一小时刷新一次，并且exp 是一年后到期，如果创建的ServiceAcccount 设置不自动挂载可以通过如下设置
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-access-sa
automountServiceAccountToken: false
```
`for pod`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  serviceAccountName: no-access-sa
  automountServiceAccountToken: false
```

### Kubernetes RBAC 实例
#### Create a Service Account
```shell
$ kubectl create sa sa-demo
serviceaccount/sa-demo created
$ kubectl create sa sa2
serviceaccount/sa2 created
```
如果创建一个Pod 资源，不绑定serviceaccount 则使用namesapce命名空间下的默认的serviceaccount 也就是default
```shell
$ kubectl get pod nginx-deploy-5947669f9-9h9vg -o yaml|grep -i serviceAccount
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
  serviceAccount: default
  serviceAccountName: default
```

#### Role and ClusterRole
##### Role In RBAC
在`default` 名称空间中定义一个名称为`pod-reader`的Role 资源，且只能访问`default`名字空间下的`secrets`资源，对该资源只能做"get", "watch"和"list" 操作
`pod-reader_role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apigroups: [""]  # 表示核心API 群组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]  # 也可以使用['*']
```
`read_secret_role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-secret
  namespace: default
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```
##### Cluster Role In RBAC
Roles in RBAC 是用来访问指定某一个命名空间下的资源，如果想要提升到集群级别那么就需要使用ClusterRole
`secret-reader-clusterole.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

####   RoleBinding and ClusterRoleBinding
##### Role Binding In RBAC
![Role Binding In RBAC](/images/kubernetes/rbac/rbac-5.png  "Role Binding In RBAC")
`read_pod_rolebinding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: sa-demo
  namespace: default
roleRef:
  # "roleRef" 指定关联是  Role / ClusterRole
  kind: Role    # 这里 必须指定 Role or ClusterRole
  name: pod-reader  # 这里绑定你想匹配的 Role or ClusterRole 的名称
  apiGroup: rbac.authorization.k8s.io
```

`read_secret_rolebinding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secret
  namespace: default
subjects:
- kind: ServiceAccount
  name: sa2
  namespace: default
roleRef:
  kind: Role 
  name: read-secret
  apiGroup: rbac.authorization.k8s.io
```
##### Cluster Role Binding In RBAC
Cluster Role Binding in RBAC 用来赋予subject 集群级别的权限，对所有的名字空间都有访问的权限

![Cluster Role Binding In RBAC](/images/kubernetes/rbac/rbac-6.png  "Cluster Role Binding In RBAC")

`secret-reader-ClusterRoleBinding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 允许任何人在"manager"组中都可以访问任何namespace 下的secrets api 对象
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: ServiceAccount
  name: sa2
  namespace: default
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```


#### 权限检查
-  检查用户或者服务账户是否有对应的权限，可以使用命令`kubectl auth can-i`
-  例如： 检查 service account sa-demo 在namespace "default" 是否有查看secrets 的权限
```shell
$ kubectl auth can-i get secrets -n default --as=system:serviceaccount:default:sa-demo
yes
```

名字空间下检查两个service account  的权限
```shell
$ kubectl auth can-i get pods -n default --as=system:serviceaccount:default:sa-demo
yes
$ kubectl auth can-i get secrets -n default --as=system:serviceaccount:default:sa-demo
no
$ kubectl auth can-i get pods -n default --as=system:serviceaccount:default:sa2
no
$ kubectl auth can-i get secrets -n default --as=system:serviceaccount:default:sa2
yes
```
集群级别检查service account sa 的权限
```shell
$ kubectl auth can-i get secrets -n default --as=system:serviceaccount:default:sa2
$ kubectl auth can-i get secrets -n kube-system --as=system:serviceaccount:default:sa2
yes
$ kubectl auth can-i get daemonsets -n kube-system --as=system:serviceaccount:default:sa2
no
```
#### 只能访问某个 namespace 的普通用户
创建一个User Account，只能访问 kube-system 这个命名空间，对应的用户信息如下所示：
```
username: lushuan
group: ikubernetes
```
创建用户凭证
我们前面已经提到过，Kubernetes 没有 User Account 的 API 对象，不过要创建一个用户帐号的话也是挺简单的，利用管理员分配给你的一个私钥就可以创建了，这个我们可以参考官方文档中的方法，这里我们来使用 0penssL 证书来创建一个 User，当然我们也可以使用更简单的 cfssl 工具来创建:给用户 lushuan 创建一个私钥，命名成 lushuan.key:
```shell
$ openssl genrsa -out lushuan.key 2048
```
使用我们刚刚创建的私钥创建一个证书签名请求文件:lushuan.csr，要注意需要确保在-subj参数中指定用户名和组(CN 表示用户名，0表示组):
```shell
$ openssl req -new -key lushuan.key -out lushuan.csr -subj "/CN=lushuan/O=ikubernetes"
```
然后找到我们的 Kubernetes 集群的 CA 证书，我们使用的是 kubeadm 安装的集群，CA 相关证书位于`/etc/kubernetes/pki/` 目录下面，如果你是二进制方式搭建的，你应该在最开始搭建集群的时候就已经指定好了 CA 的目录，我们会利用该目录下面的ca.crt和 ca.key两个文件来批准上面的证书请求。生成最终的证书文件，我们这里设置证书的有效期为500天:

基于 ca.key、ca.crt 和 lushuan.csr 等三个文件生成服务端证书lushuan.crt
```shell
$ openssl x509 -req -in lushuan.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out lushuan.crt -days 500
Certificate request self-signature ok
subject=CN = lushuan, O = ikubernetes

```
将创建的证书文件和私钥文件将其嵌入到凭证中
```shell
$ kubectl config set-credentials lushuan --client-certificate=lushuan.crt --client-key=lushuan.key
User "lushuan" set.
```
将创建的证书文件和私钥文件将其嵌入到上下文中
```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.1.11:6443
CoreDNS is running at https://192.168.1.11:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl config set-cluster "cluster-demo" --insecure-skip-tls-verify=true --server=https://192.168.1.11:6443
Cluster "cluster-demo" set.
$ kubectl config get-clusters
NAME
cluster-demo
kubernetes

# 创建上下文lushuan-contex，指定对应的集群，命名空间和用户，表示了用户lushuan 具有了在kube-system 命名空间下操作的权限
$ kubectl config set-context lushuan-context --cluster=cluster-demo --namespace=kube-system --user=lushuan
Context "lushuan-context" created.

```
查看创建结果
```shell
$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER        AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes     kubernetes-admin   
          lushuan-context               cluster-demo   lushuan            kube-system

```
验证,在我们执行`kubectl get pod` 是使用的默认的上下文，此时我们指定上面创建的上下文`lushuan-context`
```shell
$ kubectl get pod
No resources found in default namespace.
$ kubectl get pod --context=lushuan-context
Error from server (Forbidden): pods is forbidden: User "lushuan" cannot list resource "pods" in API group "" in the namespace "kube-system"

# 或者直接切换上下文
$ kubectl config use-context lushuan-context
Switched to context "lushuan-context".
kubectl get pod --context=lushuan-context
```
分享报错信息为无法list (操作)资源pods,在哪个组下,在哪个名字空间下，说白了就是需要操作的资源的相关权限，后面就需要配置RBAC了，会用到上面的提到的相关概念。
创建`lushuan-role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: lushuan-role
  namespace: kube-system
rules: #权限的规则
- apiGroups: ["","apps"]
  resources: ["deployments","replicasets","pods"]
  verbs: ["get", "list","watch","create","update","patch","delete"]  # 也可以使用['*']

```
创建`lushuan-rolebinding.yaml` 角色绑定文件
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata :
  name: lushuan-rolebindirg
  namespace: kube-system
subjects:
  - kind: User  # 外部用户
    name: lushuan
    apiGroup: ""
roleRef:     # 管理权限声明的角色
  kind: Role
  name: lushuan-role
  apiGroup: rbac.authorization.k8s.io    # 留空字符串也可以，则使用当前的apiGroup
```
创建角色和角色绑定并验证，现在就是可以正常访问了。
```shell
$ kubectl create -f lushuan-role.yaml
$ kubectl create -f lushuan-rolebinding.yaml
$ kubectl get pod --context=lushuan-context
NAME                                   READY   STATUS    RESTARTS         AGE
coredns-6554b8b87f-rf7kb               1/1     Running   4 (6h47m ago)    4d
coredns-6554b8b87f-wjt6p               1/1     Running   10 (6h47m ago)   4d
etcd-k8s-master01                      1/1     Running   51 (6h47m ago)   4d
kube-apiserver-k8s-master01            1/1     Running   46 (6h47m ago)   4d
kube-controller-manager-k8s-master01   1/1     Running   47 (6h47m ago)   3d23h
kube-proxy-78db5                       1/1     Running   36 (6h47m ago)   4d
kube-scheduler-k8s-master01            1/1     Running   45 (6h47m ago)   4d
```
如果访问daemonsets 资源呢，是否可以正常访问
```shell
$ kubectl get daemonsets --context=lushuan-context
Error from server (Forbidden): daemonsets.apps is forbidden: User "lushuan" cannot list resource "daemonsets" in API group "apps" in the namespace "kube-system"
```
我们可以看到没有权限获取，因为我们并没有为当前操作用户指定其他对象资源的访问权限，是符合我们的预期的。
这样我们就创建了一个只有单个命名空间访问权限的普通 User。

## 总结
- Role 和 RoleBinding 用于同一个namespace
- RoleBindings 可以存在于不同的命名空间，以授予服务账户权限。
- RoleBinding 可以引用同一名称空间中的role,也可以引用集群级别的ClusterRole,但是引用ClusterRole的许可权限会降级到仅能在Rolebingding所在的名称空间生效。
- ClusterRoleBinding 则用于将ClusterRole 绑定到用户和组，它作用于集群全局，可以访问集群所有资源，但是仅能够引用ClusterRole.
- ClusterRoleBinding 不可以引用Role

## 参考
1. https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/
2. https://kubernetes.io/zh-cn/docs/reference/using-api/api-concepts/
3. https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/certificates/#openssl
4. [什么是基于角色的访问控制（RBAC）？](https://www.redhat.com/zh/topics/security/what-is-role-based-access-control#rbac-%E7%9A%84%E4%BC%98%E5%8A%BF)
5. [RBAC权限系统分析、设计与实现](https://cloud.tencent.com/developer/article/1802329)