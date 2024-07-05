---
title: "Jenkins 入门介绍"
subtitle: ""
date: 2023-04-18T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Jenkins"]
tags: ["Jenkins"]
---
在了解Jenkins 之前先了解一下CI&CD
## cicd 介绍
持续集成(Continous Intergrations) 和持续部署(Continuous Deployment)的区别
- 持续集成(Continous Intergrations) 和持续部署(Continuous Deployment)是DevOps中域软件交付和
部署相关的两个重要概念
- 尽管他们的目标是类似的，但在具体含义上存在一些区别

### 持续集成CI
- 持续集成(Continous Intergrations) 是一种开发实践，通过将每次的代码改动频繁地集成到
共享代码库中，并进行自动化构建和测试，以确保团队开发的代码能够快速且无误地集成在一起

- 持续集成的目标是通过频繁的集成代码来快速发现和解决集成问题，减少冲突和错误，提供开发效率和
代码质量
### 持续部署CD
- 持续部署(Continuous Deployment)是在持续集成的基础上更进一步，指的是将经过
持续集成和测试的代码自动部署到生成环境中，使其可以立即提供用户使用
- 持续部署强调自动化的部署过程，通过自动化测试和验证确保代码的稳定性和可靠性，
使新功能、修复或者改进能够实时交付给用户

## Jenkins 介绍
Jenkins是一款开源 CI&CD 软件，用于自动化各种任务，包括构建、测试和部署软件。

Jenkins 支持各种运行方式，可通过系统包、Docker 或者通过一个独立的 Java 程序。

说明 Jenkins 是CI&CD 实现方式的一种，还有其它很多种类似的工具,不过使用于特定的场景，只不过在业界 Jenkins
是使用最广泛的一种

### Jenkins 同类CICD工具了解
1. **GitLab CI/CD**:
   - GitLab 的集成 CI/CD 功能，可以与 GitLab 代码仓库紧密集成，支持自动化构建、测试和部署流程。它具有易用性和配置简单的优点。

2. **CircleCI**:
   - CircleCI 是一个托管的持续集成和部署服务，支持多种语言和框架。它通过配置文件（YAML）定义构建流水线，并提供丰富的插件和集成支持。

3. **Travis CI**:
   - Travis CI 是一个面向开源项目的持续集成服务，支持 GitHub 和 GitLab 等代码托管平台。它使用 `.travis.yml` 文件定义构建步骤和环境设置。

4. **TeamCity**:
   - JetBrains 的 TeamCity 是一个功能强大的持续集成和部署服务器，支持多种项目类型和复杂的构建流程配置。它具有良好的可扩展性和集成能力。

5. **GitHub Actions**:
   - GitHub Actions 是 GitHub 提供的一种基于事件的自动化工作流服务，可以实现代码检查、测试和部署等操作。它与 GitHub 仓库紧密集成，支持自定义的工作流程定义。

6. **Bamboo**:
   - Atlassian 的 Bamboo 是一个企业级的持续集成和部署工具，支持构建和自动化测试、部署到多种环境等。它与其他 Atlassian 产品（如 Jira 和 Bitbucket）集成良好。

7. **Drone**:
   - Drone 是一个现代化的持续集成和交付平台，支持 Docker 容器化构建，并提供易用的配置文件和插件系统。它适合于基于容器的项目和微服务架构。

8. **Buildkite**:
   - Buildkite 是一个分布式的持续集成和交付平台，注重于灵活性和可扩展性。它通过代理机制实现构建作业的并行执行，适用于大规模和复杂的构建流程。

每个工具都有其独特的优势和适用场景，选择合适的工具通常取决于项目的具体需求、团队的技术栈和预算等因素。

## 安装
在官网的用户手册上提供了多种部署Jenkins的方式，下面提供一下基于Kubernetes 部署的Jenkins

jenkins-deploy.yaml
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: kube-ops
spec:
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      #hostNetwork: true  # 解决 updates.jenkins.io 异常问题
      terminationGracePeriodSeconds: 10
      serviceAccount: jenkins
      initContainers:
        - name: fix-permissions
          image: docker.io/library/busybox:latest # busybox:1.35.0
          imagePullPolicy: IfNotPresent
          command: ['sh', '-c', 'chown -R 1000:1000 /var/jenkins_home']
          securityContext:
            privileged: true
          volumeMounts:
            - name: jenkinshome
              mountPath: /var/jenkins_home
      containers:
      - name: jenkins
        image: localhost/jenkins/jenkins:2.452.2 #v1 -> v2.297  #jenkins/jenkins:lts 生成使用 lts 稳定版本
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:
        - name: jenkinshome
          subPath: jenkins
          mountPath: /var/jenkins_home
      securityContext:
        fsGroup: 1000
      volumes:
        - name: jenkinshome
          hostPath:
            path: /opt/jenkins
      #volumes:
      #- name: jenkinshome
      #  persistentVolumeClaim:
      #    claimName: opspvc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: kube-ops
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  type: NodePort
  ports:
  - name: web
    port: 8080
    targetPort: web
    nodePort: 30002
  - name: agent
    port: 50000
    targetPort: agent
```
jenkins-rbac.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: kube-ops

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins
  namespace: kube-ops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: kube-ops
```
在k8s 中部署的命令
```shell
mkdir /opt/jenkins
sudo chown -R 1000:1000 /opt/jenkins
sudo chmod -R 755 /opt/jenkins

kubectl create namespace kube-ops
kubectl -n kube-ops create -f rbac.yaml
kubectl -n kube-ops create -f jenkins-deploy.yaml
kubectl -n kube-ops get pod
```
{{< admonition >}}
这种是通过直接挂载目录的形式实现的，生产环境是多master 节点，建议使用 nfs 的 pvc 的方式进行挂载
{{< /admonition >}}

- [其它方式安装Jenkins-官网](https://www.jenkins.io/zh/doc/book/installing/)

## 修改镜像源
由于国内墙的原因，下载一些插件会遇到网络问题，需要修改一下镜像源
`系统管控-> 插件管理-> 高级配置`

代理镜像源信息：
`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`

![jenkins-update-center](/images/cicd/jenkins-update-center.png)

对应服务器上配置文件
```shell
root@k8s-master01:/opt/jenkins/jenkins# cat hudson.model.UpdateCenter.xml 
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json</url>
  </site>
```

## 修改 Jenkins 配置
为了方便的使用Jenkins，需要安装一写常用的插件，以下列的有点多，根据需要安装常用的插件，以下基本上都会用得到的，不了解的插件可以到插件市场上
搜索一下看下介绍。

1. Locale
2. Localization: Chinese
3. Git
4. Git Parameter
5. Git Pipeline for blue Ocean
6. Gitlab
7. Kubernetes
8. Kubernetes CLI
9. Kubernetes Credentials
10. Image Tag Parameter
11. Active Choices
12. Credentials
13. Credentials Binding
14. Blue Ocean  (Jenkins 外部UI的页面)
15. Blue Ocean Pineline Editor
16. Blue Ocean Core Js
17. Pipeline SCM API for Blue Ocean
18. Dashboard for Blue Ocean
19. Build with Parameters
20. Dynamic Extended Choice Parameter
21. Extended Choice Parameter
22. List Git Branches Parameter
23. Pipeline
24. Pipelinie: Declarative
25. Publish Over SSH
26. SonarQube Scanner
27. Localization: Chinese(Simplified)
28. Dingtalk
29. Gitlab Plugin (jenkins和Gitlab 建立安全认证)
30. Timestamper
31. AnsiColor
32. SSH
33. SSH Pipeline Steps

- [Jenkins 插件官网](https://plugins.jenkins.io/)


## 参考
- https://www.jenkins.io/zh/doc/