---
title: "基于 Gitlab Runner 构建 Gin-Vue-Admin 后端 server 服务CI/CD"
subtitle: ""
date: 2025-10-20T12:06:37+08:00
lastmod: 2025-10-20T12:06:37+08:00
draft: false
toc:
enable: true
weight: false
categories: ["Gitlab","CI/CD"]
tags: ["Gitlab","CI/CD"]
---
## 🚀 前言
记录基于 Gitlab 社区版搭建CI/CD，适用与中小团队的开发，功能包含prepare->build->push->deploy->dingTalk notify

![gitlab-pipeline-demo](/images/cicd/gitlab/gitlab-pipeline-1.png "Gitlab Pipelines")

## 🧩 环境信息

1. 操作系统：Kylin v10 (Ubuntu / CentOS 均可)
2. Kubernetes v1.28.0
3. Docker 24.0.5
4. Gitlab CE v14.1.8       (192.168.1.10)
5. gitlab-runner v18.4.0   (192.168.1.100)
6. Harbor v2.0             (192.168.1.20)
7. [Gin-Vue-Admin v2.8.6](https://github.com/flipped-aurora/gin-vue-admin/tree/v2.8.6)

目标：push/merge 到 master → 自动 build + push + k8s 部署 + 钉钉通知

## 🧱 总体流程

```
   GitLab
     ↓
 push / merge 到 master
     ↓
 GitLab Runner（安装在 192.168.1.100）
     ↓
 Docker build 镜像
     ↓
 Docker push 镜像只Harbor仓库
     ↓
 kubectl apply -f deployment.yaml
     ↓
 访问 NodePort 端口
```

## 🧰 Gitlab Runner 介绍和部署
篇幅有限，默认在搭建整个ci/cd 工具链中，其它组件已经提前部署好，这里选型gitlab-runner v18.4.0，理论上来讲建议Gitlab CE和gitlab-runner 版本相匹配，
这里直接选择了gitlab-runner v18.4.0 较高版本，在验证的过程中发现也是可以，并且插件版本向后兼容，并且还会介绍基于传统常见的Redhat 系列和Debian 系列的在线安装方式，
但是在国产数据库部署，这里选择二进制的方式进行部署

### 🧩 一、GitLab Runner 是什么？

**GitLab Runner** 是一个轻量级的守护进程，用来执行你在 `.gitlab-ci.yml` 中定义的任务。
它可以理解为：

> 💬 “GitLab 负责告诉 Runner 要做什么，Runner 负责真正去执行这些操作。”

举个例子：

当你把代码提交（push）到 GitLab 时：

1. GitLab 会检测到 `.gitlab-ci.yml`
2. 然后触发 pipeline
3. 通知 **GitLab Runner**
4. Runner 拉取代码、执行 build、测试、打包、部署等操作。

### 🏗️ 二、Runner 的工作流程

流程示意如下：

```
开发者 push 代码
       ↓
GitLab 触发 Pipeline
       ↓
Runner 拉取代码执行 Job
       ↓
返回执行日志和状态给 GitLab
```
### ⚙ 三、Runner 的组成结构

一个 GitLab Runner 服务可注册多个“执行器”（executor）来运行 Job。
执行器就是定义 Runner 如何执行 CI/CD 脚本的方式。

### 常见 Executor 类型

| 执行器类型                      | 说明                         | 适用场景      |
| -------------------------- | -------------------------- | --------- |
| **shell**                  | 直接在宿主机执行命令                 | 简单测试、本地环境 |
| **docker**                 | 每个 Job 启动一个独立的 Docker 容器执行 | 推荐、最常用    |
| **docker+machine**         | 动态创建临时 VM 执行任务             | 云环境       |
| **kubernetes**             | 在 K8s 集群中调度 Pod 执行 Job     | 云原生平台     |
| **ssh**                    | 通过 SSH 登录远程主机执行            | 异地部署或私有主机 |
| **virtualbox / parallels** | 虚拟机方式执行                    | 桌面实验环境    |

> 💡 大多数团队使用 `docker` executor，因为它最干净、易隔离、易复现。 这里使用ssh executor 执行器

### 🧰 四、Runner 的三种类型

| Runner 类型           | 说明              | 应用范围            |
| ------------------- | --------------- | --------------- |
| **Shared Runner**   | 对整个 GitLab 实例可用 | 多项目共享（通常由管理员配置） |
| **Group Runner**    | 对某个组下的所有项目可用    | 多项目组共用          |
| **Specific Runner** | 只对特定项目可用        | 小团队项目独享（推荐实验使用） |

### 🔧 五、安装与注册流程（以 Linux 为例）

#### 1️⃣ 安装 GitLab Runner

在 172.16.2.118 上执行：
Gitlab-runner 安装推荐和Gitlab 版本保持一致
```bash
# ubuntu 在线安装
sudo apt update
# 配置 runner 仓库
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner -y

# Redhat 在线安装
# 配置 runner 仓库
curl -s https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
sudo yum install gitlab-runner-18.4.0 -y

# 检查版本
gitlab-runner --version

# 在线安装后会创建了gitlab-runner用户，但是gitlab-runner用户权限有限
grep gitlab-runner /etc/passwd

# 启动建议使用 root 用户,不然在流水线脚本中引用了其它命令会提示权限问题(否则联系运维进行提权)
sudo gitlab-runner install --user=root --working-directory=/home/gitlab-runner

# 启动
gitlab-runner start

# 卸载
gitlab-runner uninstall
```
##### ✅ 国产操作系统最简、最兼容的解决办法（推荐）
> 🔧 直接安装官方「单文件二进制版」，不依赖任何 RPM 包。
> 此方法已在国产麒麟、统信、中标麒麟等系统上实测通过。


```bash
# 下载二进制文件
sudo curl -L --output /usr/local/bin/gitlab-runner \
https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

# 赋予执行权限
sudo chmod +x /usr/local/bin/gitlab-runner

# 创建运行用户
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

# 安装，这里还是使用root 权限
sudo gitlab-runner install --user=root --working-directory=/home/gitlab-runner
sudo gitlab-runner start

# 检查版本
gitlab-runner --version
```


- [gitlab-runner 官方下载地址](https://packages.gitlab.com/runner/gitlab-runner)
- [gitlab-runner 官网部署文档](https://gitlab.cn/docs/runner/install/)
- [gitlab-runner 官网使用二进制手动部署，可以适配国产操作系统](https://docs.gitlab.com/runner/install/linux-manually/#using-binary-file)

#### 2️⃣ 注册 Runner

```bash
# 直接配置，通过gitlab 获取runner token
sudo gitlab-runner register --url http://192.168.1.10/ --registration-token bixxxxxxhj8-xQ4V4

```

然后根据提示输入：

```
Please enter the gitlab-ci coordinator URL: https://gitlab.example.com/
Please enter the gitlab-ci token: <项目中显示的 Runner Token>
Please enter the gitlab-ci description: my-runner
Please enter the gitlab-ci tags: gva
Please enter the executor: docker
Please enter the default Docker image: docker:24.0.2

# 交互式配置
[root@master gitlab-runner]# gitlab-runner register
Runtime platform            arch=amd64 os=linux pid=4168532 revision=139a0ac0 version=18.4.0
Running in system-mode.                            
                                                   
Enter the GitLab instance URL (for example, https://gitlab.com/):
http://192.168.1.10/
Enter the registration token:
bixxxxxxhj8-xQ4V4
Enter a description for the runner:
[master]: cloudbackend
Enter tags for the runner (comma-separated):
gva
Enter optional maintenance note for the runner:
gavin
WARNING: Support for registration tokens and runner parameters in the 'register' command has been deprecated in GitLab Runner 15.6 and will be replaced with support for authentication tokens. For more information, see https://docs.gitlab.com/ci/runners/new_creation_workflow/ 
Registering runner... succeeded                     correlation_id=01K7R3R2FPKS1RPQGWR4G9DZWQ runner=biixrApv2
Enter an executor: parallels, docker, docker+machine, kubernetes, instance, shell, virtualbox, docker-windows, docker-autoscaler, custom, ssh:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
 
Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml" 
```

填写信息（示例）：

| 提示                                         | 示例输入                                        |
| ------------------------------------------ |---------------------------------------------|
| Please enter the gitlab-ci coordinator URL | `http://your.gitlab.domain/`                |
| Please enter the gitlab-ci token           | 从 GitLab 项目 → Settings → CI/CD → Runners 获取 |
| Please enter the gitlab-ci description     | `kylin-runner`                              |
| Please enter the gitlab-ci tags            | `gva` 可以理解为给gitlab runner实例命名               |
| Please enter the executor                  | `shell`                                     |

注册完成后，Runner 会出现在 GitLab → `Settings → CI/CD → Runners` 中。

#### 启动并验证：

```bash
sudo systemctl enable gitlab-runner
sudo systemctl start gitlab-runner
sudo systemctl status gitlab-runner
sudo gitlab-runner status
```

然后执行：

```bash
gitlab-runner list
```


### 🧩 六、Runner 与 `.gitlab-ci.yml` 的关系

在你的 `.gitlab-ci.yml` 文件中：

```yaml
build:
  stage: build
  tags:
    - gva       # Runner 标签（匹配 Runner）
  script:
    - echo "Build project"
```

🔹 GitLab 会根据 `tags: [gva]` 匹配到相应 Runner 来执行此 job。
如果没有匹配标签的 Runner，Job 会一直处于 *pending* 状态。

---
## ⚙️GitLab CI 配置文件 `.gitlab-ci.yml`

在项目根目录新建 `.gitlab-ci.yml` 文件，内容如下：
```
stages:
  - prepare   # 定义JOB全局环境变量
  - build     # 编译
  - push      # 镜像推送
  - deploy    # 部署
  - notify    # 通知

# 全局变量定义
variables:
  PROJECT_NAME: gin-vue-admin
  IMAGE_NAME: 192.168.1.20/library/cloudbackend       # 修改为你的 Harbor 项目地址
  K8S_DEPLOY_FILE: deploy/cloudbackend-deployment.yaml        # k8s 部署文件路径
  K8S_DEPLOY_CM_FILE: deploy/cloudbackend-configmap.yaml
  K8S_DEPLOY_SERVICE_FILE: deploy/cloudbackend-service.yaml
  K8S_NAMESPACE: default
  DOCKER_DRIVER: overlay2
  TIMEOUT: "60s"

# ==========================================
# 定义 JOB 全局变量，生成时间戳并保存为 artifact (一次生成，全局共享)
# ==========================================
prepare:
  stage: prepare
  tags:
    - gva
  script:
    - echo "IMAGE_TAG=$(date '+%Y%m%d%H%M%S')" > image_tag.env
    - cat image_tag.env
  artifacts:
    reports:
      dotenv: image_tag.env     # ✅ GitLab 会把这里的变量注入后续 Job 环境中
  only:
    - master

# ==========================================
# Stage 1: 构建镜像
# ==========================================
build:
  stage: build
  tags:
    - gva
  before_script:
    - docker info
  script:
    - echo "开始 $(date)"
    - echo "=== Step 1 开始构建镜像 ==="
    - docker build --network=host -t $IMAGE_NAME:$IMAGE_TAG .
    - docker image prune -f    # ✅ 构建后清理 无标签dangling 镜像，节省空间
    - echo "=== 镜像构建完成 $IMAGE_NAME:$IMAGE_TAG ==="
    - echo "结束 $(date)"
  only:
    - master

# ==========================================
# Stage 2: 推送镜像到 Harbor
# ==========================================
push:
  stage: push
  tags:
    - gva
  before_script:
    - echo "=== 登录 Harbor 仓库 ==="
    - echo $HARBOR_PASSWORD | docker login 192.168.1.20 -u $HARBOR_USER --password-stdin
  script:
    - echo "开始 $(date)"
    - echo "=== Step 2 推送镜像 ==="
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - echo "=== 镜像推送成功 $IMAGE_NAME:$IMAGE_TAG ==="
    - echo "结束 $(date)"
  only:
    - master

# ==========================================
# Stage 3: 部署到 k8s 并支持自动回滚
# ==========================================
deploy:
  stage: deploy
  tags:
    - gva
  before_script:
    - echo "=== 准备 K8s 环境 ==="
    - echo "当前命名空间 $K8S_NAMESPACE"
    - kubectl version --client
  script:
    - echo "开始 $(date)"
    - echo "=== Step 3 记录当前镜像版本 ==="
    - OLD_IMAGE=$(kubectl get deploy cloudbackend -n $K8S_NAMESPACE -o jsonpath='{.spec.template.spec.containers[0].image}')
    - echo "上一个版本镜像 $OLD_IMAGE"

    - echo "=== Step 4 更新 Deployment 镜像 ==="
    - kubectl apply -f $K8S_DEPLOY_CM_FILE -n $K8S_NAMESPACE
    - kubectl apply -f $K8S_DEPLOY_FILE -n $K8S_NAMESPACE
    - kubectl apply -f $K8S_DEPLOY_SERVICE_FILE -n $K8S_NAMESPACE
    - kubectl set image deployment/cloudbackend cloudbackend=$IMAGE_NAME:$IMAGE_TAG  -n $K8S_NAMESPACE

    - echo "=== Step 5 等待新版本 Pod 启动 ==="
    - kubectl rollout status deployment/cloudbackend -n $K8S_NAMESPACE --timeout=$TIMEOUT || ROLLBACK=true

    - |
      if [ "$ROLLBACK" = "true" ]; then
        echo "❌ 部署失败，自动回滚到上一个版本: $OLD_IMAGE"
        kubectl set image deployment/cloudbackend cloudbackend=$OLD_IMAGE -n $K8S_NAMESPACE
        kubectl rollout status deployment/cloudbackend -n $K8S_NAMESPACE --timeout=$TIMEOUT
        exit 1
      else
        echo "✅ 部署成功，新版本镜像: $IMAGE_NAME:$IMAGE_TAG"
        kubectl get pods -n $K8S_NAMESPACE -o wide
      fi
    - echo "结束 $(date)"
  only:
    - master

# ==========================================
# 通知阶段 (不影响主流程),并且提供钉钉稳健通知方案，
# 避免偶发性网络问题
# ==========================================
notify:
  stage: notify
  tags:
    - gva
  script:
    - 'echo "构建完成通知： $(date)"'
  after_script:
    - echo "当前 Job 状态  $CI_JOB_STATUS"
    - |
      if [ "$CI_JOB_STATUS" = "success" ]; then
        STATUS="✅ 成功"
        COLOR="green"
      else
        STATUS="❌ 失败"
        COLOR="red"
      fi
      
      MESSAGE="{
        \"msgtype\": \"markdown\",
        \"markdown\": {
          \"title\": \"[${CI_PROJECT_NAME}] 构建${STATUS}\",
          \"text\": \"### 🚀 ${CI_PROJECT_NAME}  CI/CD cloud通知\n
          > **Pipeline ID:** ${CI_PIPELINE_ID}\n
          > **分支:** ${CI_COMMIT_REF_NAME}\n
          > **镜像:** ${IMAGE_NAME}:${IMAGE_TAG}\n
          > **提交:** ${CI_COMMIT_SHORT_SHA}\n
          > **执行人:** ${GITLAB_USER_NAME}\n
          > **状态:** ${STATUS}\n
          > **时间:** $(date '+%Y-%m-%d %H:%M:%S')\n
          > [查看详情](${CI_PIPELINE_URL})\"
        }
      }"

      MAX_RETRY=3
      for i in $(seq 1 $MAX_RETRY); do
        echo "尝试发送第 $i 次通知..."
        RESPONSE=$(curl -s -w "%{http_code}" -o /tmp/ding.log \
          -H "Content-Type: application/json" \
          -d "$MESSAGE" "$DINGTALK_WEBHOOK")
        if [ "$RESPONSE" = "200" ]; then
          echo "✅ 钉钉通知成功"
          break
        else
          echo "⚠️ 第 $i 次失败，等待 5 秒重试"
          sleep 5
        fi
      done
  when: always   # ✅ 无论成功或失败都执行
  only:
    - master
```

注意：
- `.gitlab-ci.yaml` 脚本的秘钥环境变量推荐配置到gitlab 项目下的Variables下,项目->Settings->CI/CD->Variables,当然写死在`Gitlab-ci.yaml`中也可以

![gitlab-pipeline-demo](/images/cicd/gitlab/gitlab-pipeline-2.png "Gitlab Variables")

## 🐳 Dockerfile 示例（放在项目根目录）
如果你已有官方 GVA 项目，可参考下方后端构建 Dockerfile，该Dockerfile 优化了如下问题
1. DNS 解析不稳定问题
2. 使用国内镜像源进行加速代理
3. 优化使用框架依赖缓存，每次构建拉取时间较长问题
4. 优化构建后的image size 大小问题，减少一半

```
FROM golang:alpine as builder

WORKDIR /app
# 优化：先复制 go.mod / go.sum 仅用于依赖缓存
COPY go.mod go.sum ./


# 配置 Go 环境和国内代理
RUN go env -w GO111MODULE=on \
    && go env -w GOPROXY=https://goproxy.cn,https://proxy.golang.com.cn,https://goproxy.io,https://goproxy.tencentcloudapi.com,direct \
    && go env -w CGO_ENABLED=0 \
    && go mod download

# 再复制源代码（这样 go mod download 有缓存）
COPY . .

# 编译阶段：只编译 server，不运行 tidy（tidy 可在本地执行）
RUN go build -trimpath -ldflags="-s -w" -o server .

FROM alpine:latest

LABEL MAINTAINER="SliverHorn@sliver_horn@qq.com"
# 设置时区
ENV TZ=Asia/Shanghai

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
    && apk add --no-cache tzdata openntpd \
    && ln -sf /usr/share/zoneinfo/$TZ /etc/localtime \
    && echo $TZ > /etc/timezone

WORKDIR /go/src/github.com/flipped-aurora/gin-vue-admin/server

COPY --from=0 /app/server ./
COPY --from=0 /app/resource ./resource/
COPY --from=0 /app/config.docker.yaml ./
COPY --from=0 /app/config.yaml ./

# 挂载目录：如果使用了sqlite数据库，容器命令示例：docker run -d -v /宿主机路径/gva.db:/app/gva.db -p 8888:8888 --name gva-server-v1 gva-server:1.0
# VOLUME ["/app"]

EXPOSE 8888
ENTRYPOINT ["./server", "-c", "config.yaml"]
```
## 🧩 K8S 部署示例
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudbackend-config
data:
  config.yaml: |
    aliyun-oss:
      endpoint: yourEndpoint
      access-key-id: yourAccessKeyId
      access-key-secret: yourAccessKeySecret
     ... 省略
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudbackend
  labels:
    app: cloudbackend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudbackend
  template:
    metadata:
      labels:
        app: cloudbackend
    spec:
      containers:
        - name: cloudbackend
          image: "192.168.1.20/library/cloudbackend:v1"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8888
          env:
            - name: GIN_MODE
              value: "release"
            - name: ENVIRONMENT
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: TZ
              value: "Asia/Shanghai"
          volumeMounts:
            - name: config-volume
              mountPath: /go/src/github.com/flipped-aurora/gin-vue-admin/server/config.yaml
              subPath: config.yaml
      imagePullSecrets:
        - name: harbor-secret
      volumes:
        - name: config-volume
          configMap:
            name: cloudbackend-config
---
apiVersion: v1
kind: Service
metadata:
  name: cloudbackend
  labels:
    app: cloudbackend
spec:
  type: NodePort
  selector:
    app: cloudbackend
  ports:
    - name: http
      port: 8888
      nodePort: 30088
```
## 🧠 测试与验证

1️⃣ 推送到 master：

```bash
git add .
git commit -m "test gitlab ci"
git push origin master
```

2️⃣ 打开 GitLab 项目→ **CI/CD → Pipelines**
可看到：

* build ✅
* push ✅
* deploy ✅

3️⃣ 验证部署：

```bash
kubectl get pods
kubectl get svc
```

访问：

```
http://k8s_master_vip:30088
```

查看钉钉构建通知
![gitlab-pipeline-demo](/images/cicd/gitlab/gitlab-pipeline-3.png "DingTalk Notify")


## 🧩 常见问题排查

| 问题                          | 解决方法                                      |
| --------------------------- | ----------------------------------------- |
| `docker: command not found` | 在 Runner 节点安装 docker 并配置到 PATH。           |
| `kubectl not found`         | 确认已安装 kubectl 且能访问 k8s 集群。                |
| Job 一直 pending              | 检查 Runner 是否注册到项目、是否有 tag 匹配。             |
| 镜像无法拉取                      | 将 `imagePullPolicy` 设置为 `Never` 或推送到私有仓库。 |

## 🧩 踩坑
在提交`.gitlab-ci.yaml` 调试的过程中，一直提示如下报错
```
Syntax is incorrect. CI configuration validated, including all configuration added with the includes keyword
```
### 原因分析:
GitLab CI 的 YAML 解析器在 14.x 的某些版本里，对 script 中包含冒号 : 的行解析异常
尤其是这种形式
```
script:
  - echo "=== Step 1: 开始构建镜像 ==="
```
解析器把 === Step 1: 里的冒号当成 YAML key，导致报错：
```lua
jobs:build:script config should be a string or a nested array of strings
```
注意，这不是标准 YAML 问题，而是 GitLab CI Lint 在特定版本对冒号敏感导致的“虚假报错”。
### 解决方案
方法 1：去掉冒号
```
script:
  - echo "=== Step 1 开始构建镜像 ==="
```
方法 2：用单引号包裹整个字符串（推荐）
```
script:
  - 'echo "=== Step 1: 开始构建镜像 ==="'
```
外层使用单引号 '...'，里面的冒号就不会被解析器误认为 key。

##  ✅ 总结

| 模块    | 技术                      |
| ----- | ----------------------- |
| CI 触发 | GitLab 内置 webhook       |
| 执行器   | GitLab Runner（shell 模式） |
| 构建方式  | Dockerfile 构建           |
| 部署方式  | kubectl apply 到 k8s     |
| 暴露方式  | NodePort 服务             |
| 整体特性  | 零轮询、自动触发、可视化、可扩展        |

