---
title: "Docker 入门指南：从核心概念到容器化应用实践"
subtitle: ""
date: 2022-11-02T16:25:58+08:00
lastmod: 2023-03-07T16:25:58+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Docker"]
tags: ["Docker"]
slug: "docker-guide"
featuredImage: "/images/docker/cover.jpg"
summary: "本文介绍 Docker 的基本概念、镜像管理、容器运行以及 Dockerfile 编写。"
---

本文介绍 Docker 的基本概念、镜像管理与容器运行。
<!--more-->


## 一、Docker 是什么？

Docker 是一种用于运行和管理容器化应用的技术。

它解决的一个核心问题是：

> **如何让应用及其运行环境保持一致，从而避免“在我电脑上能运行，换一台机器就不能运行”的问题。**

一个应用通常依赖很多外部环境，例如：

```
Node.js 版本
Python 版本
Go 版本
Nginx
MySQL
PostgreSQL
系统库
第三方依赖
环境变量
配置文件
```

如果开发环境和生产环境的版本不同，就可能出现：

```
开发环境：
Node.js 20
MySQL 8
Python 3.11

生产环境：
Node.js 18
MySQL 5.7
Python 3.9
```

即使代码完全相同，最终运行结果也可能不同。

Docker 的思路是：

```
应用程序
+
依赖环境
+
运行配置
        ↓
    Docker Image
        ↓
    Container
```

把应用运行所需要的环境标准化，然后使用容器运行。

---

## 二、Docker 的四个核心对象

Docker 中最重要的四类对象是：

```
Image
Container
Volume
Network
```

可以先建立这样一个整体认识：

```
                    Docker
                      │
        ┌─────────────┼─────────────┐
        │             │             │
      Image       Container       Volume
        │             │             │
      模板          运行实例       持久化数据
                      │
                      │
                    Network
                      │
                  容器之间通信
```

其中最需要理解的是：

> **Image 是模板，Container 是运行实例。**

---

## 三、Image：Docker 镜像

Docker Image 可以理解为：

> **创建容器所使用的模板。**

例如：

```
nginx:latest
redis:7
mysql:8
golang:1.18
```

这些都是 Docker Image。

一个镜像中通常包含：

```
应用程序
运行时环境
依赖
文件系统
配置
启动信息
```

例如：

```
golang:1.18
```

可以理解为：

```
Linux 基础环境
+
Go 1.18
+
Go 编译工具
+
相关运行环境
```

然后基于这个 Image 创建 Container。

---

## 四、Container：Docker 容器

Container 是镜像运行后的实例。

可以用一个非常简单的关系理解：

```
Image
  ↓ docker run
Container
```

例如：

```
docker run nginx
```

Docker 会：

```
nginx Image
    ↓
创建
    ↓
nginx Container
```

因此：

```
Image      = 模板
Container  = 实例
```

同一个 Image 可以创建多个 Container：

```
             nginx:latest
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
   nginx-1    nginx-2    nginx-3
```

这也是容器化部署能够快速扩展的基础。

---

## 五、Image 和 Container 的关系

可以类比成：

```
面向对象：

Class
  ↓
Object
```

对应：

```
Docker：

Image
  ↓
Container
```

Image 本身不是正在运行的应用。

真正运行的是 Container。

---

## 六、Docker Registry：镜像仓库

镜像可以保存到 Registry 中。

最常见的是：

```
Docker Hub
```

除此之外，还有：

```
Harbor
GitLab Container Registry
阿里云容器镜像服务
AWS ECR
GitHub Container Registry
```

可以把 Registry 理解成：

> **Docker 镜像的远程仓库。**

例如：

```
docker pull nginx
```

实际上就是：

```
本地 Docker
    ↓
Registry
    ↓
下载 nginx Image
```

构建好的镜像也可以：

```
docker push
```

上传到镜像仓库。

因此一个典型的 CI/CD 流程可以是：

```
Git
 ↓
CI
 ↓
docker build
 ↓
Docker Image
 ↓
docker push
 ↓
Container Registry
 ↓
服务器 / Kubernetes
 ↓
docker pull
 ↓
运行 Container
```

---

## 七、Dockerfile：如何制作镜像

Dockerfile 是：

> **描述如何构建 Docker Image 的文件。**

例如：

```
FROM golang:1.18

WORKDIR /app

COPY . .

RUN go build -o server .

EXPOSE 8080

CMD ["./server"]
```

Docker 根据 Dockerfile：

```
FROM
 ↓
选择基础镜像

COPY
 ↓
复制代码

RUN
 ↓
执行构建命令

EXPOSE
 ↓
声明应用端口

CMD
 ↓
定义默认启动命令
```

最终：

```
Dockerfile
    ↓ docker build
Docker Image
    ↓ docker run
Container
```

---

## 八、Dockerfile 常用指令

最常用的指令包括：

| 指令 | 作用 |
| --- | --- |
| `FROM` | 指定基础镜像 |
| `RUN` | 构建镜像时执行命令 |
| `COPY` | 复制文件 |
| `ADD` | 复制文件，具有额外功能 |
| `WORKDIR` | 设置工作目录 |
| `ENV` | 设置环境变量 |
| `ARG` | 定义构建参数 |
| `EXPOSE` | 声明应用端口 |
| `CMD` | 定义默认启动命令 |
| `ENTRYPOINT` | 定义容器入口程序 |
| `USER` | 指定运行用户 |
| `VOLUME` | 声明数据卷 |

其中最值得优先掌握：

```
FROM
COPY
RUN
WORKDIR
CMD
ENTRYPOINT
ENV
ARG
EXPOSE
USER
```

---

## 九、.dockerignore

当 Dockerfile 中存在：

```
COPY . .
```

Docker 会把构建上下文中的文件用于构建。

因此通常应该配置：

```
.dockerignore
```

例如：

```
.git
.gitignore
.idea
.vscode

*.log

tmp/
logs/

dist/
build/

coverage/
```

这样可以避免：

```
Git 历史
IDE 配置
日志
临时文件
构建产物
测试数据
```

等无用内容进入构建上下文。

如果其中包含：

```
.env
```

还可以避免将包含敏感信息的文件意外复制到镜像中。

但是需要注意：

> `.dockerignore` 主要解决的是构建上下文和复制问题，不能替代真正的密钥管理方案。

---

## 十、为什么要关注镜像大小？

镜像越大，通常意味着：

```
构建时间更长
上传时间更长
下载时间更长
CI/CD 时间更长
存储成本更高
```

例如：

```
1 GB Image
```

如果需要频繁：

```
build
push
pull
deploy
```

这些额外的数据传输都会累积成本。

因此 Docker 镜像通常希望：

> **在满足应用运行需求的前提下尽可能精简。**

---

## 十一、基础镜像应该怎么选择？

常见基础镜像包括：

```
Ubuntu
Debian
Alpine
Distroless
Scratch
```

它们并不存在绝对的“最好”。

例如：

```
Ubuntu / Debian
    ↓
兼容性较好
工具比较完整

Alpine
    ↓
体积较小
用户空间比较精简

Distroless
    ↓
运行环境进一步精简

scratch
    ↓
几乎没有用户空间
```

因此选择基础镜像时应该考虑：

```
程序运行依赖
C/C++ 动态库
证书
时区
调试需求
安全要求
镜像大小
```

而不是单纯追求：

> “镜像越小越好。”

---

## 十二、多阶段构建

多阶段构建是 Dockerfile 中非常重要的实践。

尤其对于：

```
Go
Java
Rust
C/C++
```

等需要编译的项目。

例如 Go：

```
FROM golang:1.18 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build -o server .

FROM alpine:3.18

WORKDIR /app

COPY --from=builder /app/server ./server

EXPOSE 8080

ENTRYPOINT ["./server"]
```

这里实际上有两个阶段：

```
第一阶段
golang:1.18
    ↓
负责编译

第二阶段
alpine
    ↓
负责运行
```

最终运行镜像中不需要保留：

```
Go Compiler
Go SDK
源码
编译工具
```

只需要：

```
server
+
运行时需要的文件
```

因此可以显著减少最终镜像内容。

---

## 十三、Docker Container 的基本操作

启动一个容器：

```
docker run nginx
```

后台运行：

```
docker run -d nginx
```

指定名称：

```
docker run -d \
  --name nginx \
  nginx
```

查看运行中的容器：

```
docker ps
```

查看所有容器：

```
docker ps -a
```

查看容器日志：

```
docker logs nginx
```

持续查看日志：

```
docker logs -f nginx
```

进入正在运行的容器：

```
docker exec -it nginx sh
```

查看容器详细信息：

```
docker inspect nginx
```

停止：

```
docker stop nginx
```

删除：

```
docker rm nginx
```

强制删除：

```
docker rm -f nginx
```

---

## 十四、容器为什么需要端口映射？

假设容器中的 Web 服务监听：

```
8080
```

Dockerfile：

```
EXPOSE 8080
```

运行：

```
docker run -p 8080:8080 app
```

表示：

```
宿主机 8080
      ↓
容器 8080
```

需要特别注意：

```
EXPOSE 8080
```

只是声明端口。

它并不会自动创建：

```
宿主机 → 容器
```

的端口映射。

真正完成映射的是：

```
-p 8080:8080
```

---

## 十五、Docker Volume：数据持久化

容器有一个非常重要的特点：

> **容器本身通常应该被认为是可以随时销毁和重新创建的。**

因此，如果把重要数据直接存放在容器可写层：

```
Container
   ↓
删除
   ↓
数据可能丢失
```

对于：

```
数据库
上传文件
持久化数据
```

显然是不合适的。

Docker Volume 就是解决这个问题的。

例如：

```
docker volume create db-data
```

然后：

```
docker run \
  -v db-data:/var/lib/mysql \
  mysql
```

此时：

```
Container
    │
    ↓
Volume
    │
    ↓
持久化数据
```

即使容器被删除：

```
docker rm mysql
```

Volume 仍然可以保留。

之后重新创建容器：

```
新 Container
     ↓
挂载原来的 Volume
     ↓
继续使用原来的数据
```

---

## 十六、Docker Network：容器之间如何通信？

Docker Network 用于：

> **连接和隔离 Docker 容器。**

例如：

```
web
 │
 │ HTTP
 ↓
api
 │
 │ TCP
 ↓
mysql
```

可以创建一个网络：

```
docker network create app-net
```

然后：

```
docker run -d \
  --network app-net \
  --name api \
  api:1.0
```

数据库：

```
docker run -d \
  --network app-net \
  --name mysql \
  mysql:8
```

同一个 Docker Network 中的容器可以通过容器名称进行通信。

例如：

```
api → mysql:3306
```

而不需要依赖容器不断变化的 IP 地址。

---

## 十七、Docker Compose

当应用只有一个容器时：

```
docker run ...
```

通常还能接受。

但如果一个项目需要：

```
Web
API
MySQL
Redis
Nginx
```

每次手动执行大量 `docker run` 命令就比较麻烦。

Docker Compose 的作用就是：

> **用一个 YAML 文件描述多个容器及其关系。**

例如：

```
services:
  api:
    build: ./api
    ports:
      - "8080:8080"

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - db-data:/var/lib/mysql

volumes:
  db-data:
```

然后：

```
docker compose up
```

就可以启动整个应用。

后台运行：

```
docker compose up -d
```

停止并删除相关资源：

```
docker compose down
```

查看服务：

```
docker compose ps
```

查看日志：

```
docker compose logs
```

需要注意：

> 文章原始版本发表于 2020 年，当时大量使用 `docker-compose` 命令；现代 Docker Compose 已经统一使用 `docker compose` 子命令形式。

因此现在更推荐：

```
docker compose up
```

而不是：

```
docker-compose up
```

---

## 十八、环境变量

容器中的配置通常不应该全部写死在镜像里。

例如：

```
数据库地址
数据库密码
Redis 地址
运行环境
服务端口
第三方服务地址
```

可以通过环境变量传递。

例如：

```
docker run \
  -e APP_ENV=prod \
  -e DB_HOST=mysql \
  my-app:1.0
```

Docker Compose 中：

```
services:
  api:
    image: my-api:1.0
    environment:
      APP_ENV: prod
      DB_HOST: mysql
```

也可以使用：

```
services:
  api:
    env_file:
      - .env
```

但需要注意：

> `.env` 文件中如果存在密码、Token、API Key 等敏感信息，不应该直接提交到 Git 仓库。

生产环境还应该考虑专门的 Secret 管理方案。

---

## 十九、Docker 中的数据和配置应该如何理解？

可以形成一个非常重要的原则：

```
Image
 ↓
应用程序和运行环境
```

```
Container
 ↓
应用运行实例
```

```
Volume
 ↓
持久化数据
```

```
Environment / Config
 ↓
运行配置
```

```
Network
 ↓
容器之间的通信
```

因此：

> **不要把 Container 当成传统服务器。**

传统服务器可能：

```
运行很久
修改配置
手动安装软件
保存数据
```

而容器更倾向于：

```
构建 Image
     ↓
运行 Container
     ↓
发现版本问题
     ↓
重新构建 Image
     ↓
重新创建 Container
```

也就是：

> **修改镜像，而不是长期修改正在运行的容器。**

---

## 二十、Docker 的典型应用流程

一个比较完整的 Docker 开发流程：

```
             编写代码
                ↓
            Dockerfile
                ↓
          docker build
                ↓
             Image
                ↓
          本地测试
                ↓
          docker push
                ↓
        Container Registry
                ↓
         服务器 / K8s
                ↓
           docker pull
                ↓
           Container
```

如果加入 CI/CD：

```
Git Push
   ↓
CI Pipeline
   ↓
Unit Test
   ↓
docker build
   ↓
Image Scan
   ↓
docker push
   ↓
Registry
   ↓
Deploy
   ↓
Kubernetes
```

Docker 在现代云原生体系中，更多是：

> **应用交付和运行时封装的一环。**

而不是整个部署体系本身。

---

## 二十一、Docker 和虚拟机的区别

Docker 容器和虚拟机都可以提供隔离环境，但实现方式不同。

虚拟机：

```
Physical Machine
       ↓
Hypervisor
       ↓
Virtual Machine
       ↓
Guest OS
       ↓
Application
```

容器：

```
Physical Machine
       ↓
Host OS / Linux Kernel
       ↓
Container Runtime
       ↓
Container
       ↓
Application
```

最核心的区别是：

> **容器通常共享宿主机的 Linux Kernel，而虚拟机拥有自己的 Guest OS。**

因此容器通常：

```
启动更快
资源开销更低
镜像更小
部署更加轻量
```

而虚拟机提供的 OS 级隔离更完整。

所以不能简单理解成：

```
Docker = 更先进的虚拟机
```

它们解决的问题存在重叠，但底层机制不同。

---

## 二十二、Docker Engine

Docker Engine 可以理解为 Docker 的核心运行环境。

其中：

```
Docker CLI
    ↓
Docker API
    ↓
Docker Daemon
    ↓
Container / Image / Network / Volume
```

开发者最常使用：

```
docker
```

命令行工具。

例如：

```
docker ps
docker images
docker build
docker run
docker exec
docker logs
```

CLI 最终通过 Docker Engine 提供的 API 与 Docker daemon 交互。

---

## 二十三、Docker 中最重要的几个认知

如果把整个 Docker 入门内容浓缩一下，可以得到下面这套模型：

```
                    Docker
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
      Image         Container       Registry
        │              │
      模板           运行实例
                       │
              ┌────────┴────────┐
              ↓                 ↓
           Volume            Network
            数据              通信
```

应用开发：

```
Dockerfile
    ↓
docker build
    ↓
Image
```

应用运行：

```
Image
    ↓
docker run
    ↓
Container
```

数据持久化：

```
Container
    ↓
Volume
```

容器通信：

```
Container
    ↓
Network
```

镜像共享：

```
Image
    ↓
Registry
```

多容器应用：

```
Docker Compose
```

大规模容器编排：

```
Kubernetes
```

---

## 二十四、几个非常值得记住的 Docker 原则

### 原则一：Image 和 Container 不要混淆

```
Image = 模板
Container = 实例
```

### 原则二：容器应该尽量无状态

需要保存的数据：

```
Volume
数据库
对象存储
外部存储
```

不要依赖容器自身保存重要数据。

### 原则三：尽量使用不可变镜像

不要长期：

```
docker exec container ...
```

然后在里面手工安装一堆软件。

更推荐：

```
修改 Dockerfile
    ↓
重新 build
    ↓
新 Image
    ↓
重新部署
```

### 原则四：生产镜像尽量精简

可以使用：

```
多阶段构建
.dockerignore
精简基础镜像
只复制运行所需文件
非 root 用户
```

### 原则五：配置与镜像分离

例如：

```
Image
    ↓
代码 + 运行环境

Environment / Config
    ↓
环境相关配置
```

这样同一个镜像：

```
my-app:1.0
```

可以部署到：

```
开发环境
测试环境
生产环境
```

而不需要为每个环境重新构建一份镜像。

---

## 二十五、Docker 学习路线

如果按照实际开发使用来学习 Docker，可以按照下面的顺序：

```
① Docker 基本概念
   │
   ├── Image
   ├── Container
   ├── Registry
   ├── Volume
   └── Network
   ↓
② Dockerfile
   │
   ├── FROM
   ├── RUN
   ├── COPY
   ├── WORKDIR
   ├── CMD
   └── ENTRYPOINT
   ↓
③ Container 操作
   │
   ├── run
   ├── ps
   ├── logs
   ├── exec
   ├── stop
   └── rm
   ↓
④ 镜像优化
   │
   ├── .dockerignore
   ├── Build Cache
   ├── Multi-stage Build
   └── 非 root
   ↓
⑤ Volume / Network
   ↓
⑥ Docker Compose
   ↓
⑦ Registry
   ↓
⑧ CI/CD
   ↓
⑨ Kubernetes
```

其中对于 Go 开发者，建议重点掌握：

```
Dockerfile
多阶段构建
镜像优化
容器生命周期
Volume
Network
Docker Compose
Registry
```

然后再进一步进入：

```
Docker
   ↓
Kubernetes
   ↓
Helm
   ↓
CI/CD
   ↓
容器安全
   ↓
可观测性
```

这样 Docker 就不再是孤立的一项技术，而是整个云原生工程体系中的一个组成部分。

---

## 二十六、总结

Docker 最核心的思想可以浓缩成一句话：

> **把应用程序及其运行所需要的环境封装成标准化的镜像，再通过容器运行这个镜像。**

理解下面这条链路，就基本建立了 Docker 的整体认知：

```
Source Code
     ↓
Dockerfile
     ↓
docker build
     ↓
Docker Image
     ↓
Docker Registry
     ↓
docker pull
     ↓
Docker Container
     ↓
Volume / Network
     ↓
Application
```

而对于实际开发，最重要的不是记住所有 Docker 命令，而是理解：

```
Image       → 应用运行模板
Container   → Image 的运行实例
Dockerfile  → Image 的构建说明
Registry    → Image 的存储和分发
Volume      → 数据持久化
Network     → 容器通信
Compose     → 多容器应用管理
Kubernetes  → 大规模容器编排
```

当这些概念建立起来之后，再学习 Dockerfile、多阶段构建、Compose、Kubernetes，就会从“记命令”变成“理解系统”。











