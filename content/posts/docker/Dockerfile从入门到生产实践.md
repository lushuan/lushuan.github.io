---
title: "Dockerfile 使用指南：从 Go 二进制构建到生产镜像优化"
subtitle: ""
date: 2023-04-16T16:25:58+08:00
lastmod: 2024-06-26T16:25:58+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Docker"]
tags: ["Docker","Dockerfile"]
slug: "dockerfile-guide"
featuredImage: "/images/blog_cover/caoyuan/3.jpg"
summary: "本文以 Go 1.20 项目为例讲解 Dockerfile 的常用指令、CMD 与 ENTRYPOINT 的区别、多阶段构建、构建缓存与镜像优化等生产实践，并集中整理 Dockerfile 相关常见面试题。"
---

Dockerfile 是学习 Docker 时非常核心的一部分，真正掌握它的关键是理解应用如何从源码经过构建变成可运行的容器镜像。本文以 Go 1.20 项目为例，从基本概念开始逐步介绍常用指令、CMD 与 ENTRYPOINT 的区别、多阶段构建、构建缓存优化以及非 root 用户等生产实践，最后集中整理 Dockerfile 相关的常见面试问题。

<!--more-->

Dockerfile 是学习 Docker 时非常核心的一部分。真正掌握 Dockerfile，并不是把 `FROM`、`RUN`、`COPY`、`CMD` 等指令背下来，而是要理解：

> **一个应用程序是如何从源码，经过构建，最终变成一个可以运行的容器镜像。**

本文以 **Go 1.20** 为例，从 Dockerfile 的基本概念开始，逐步介绍常用指令、多阶段构建、镜像优化以及 `CMD` 和 `ENTRYPOINT` 的区别，最后再集中整理 Dockerfile 相关的常见面试问题。

---

## 一、Dockerfile 是什么？

可以把 Dockerfile 简单理解为：

> **Docker 镜像的构建说明书。**

我们平时运行：

```bash
docker build
```

Docker 就会读取 Dockerfile，根据其中的指令一步一步构建镜像。

例如一个 Go 项目的整个过程可以理解为：

```text
Go 源代码
    ↓
go build
    ↓
Go 二进制程序
    ↓
Dockerfile
    ↓
Docker Image
    ↓
docker run
    ↓
Container
```

Docker 官方将 Dockerfile 定义为：

> 一个包含构建 Docker 镜像所需指令的文本文件。

常见指令包括：

```text
FROM
RUN
COPY
ADD
WORKDIR
ENV
ARG
EXPOSE
USER
CMD
ENTRYPOINT
```

以及 `HEALTHCHECK`、`LABEL`、`VOLUME` 等。[Docker Documentation](https://docs.docker.com/reference/dockerfile?utm_source=chatgpt.com)

---

## 二、为什么需要 Dockerfile？

假设现在有一个 Go 服务：

```text
myapp
```

在服务器上直接运行可能是：

```bash
./myapp
```

但如果需要部署到不同服务器，就可能遇到：

```text
Go 环境是否一致？
Linux 发行版是否一致？
依赖库是否一致？
配置是否一致？
启动方式是否一致？
```

Docker 希望解决的就是：

> **把应用程序以及它运行所需要的环境一起打包。**

最终得到：

```text
Docker Image
```

然后无论在哪台支持 Docker 的机器上：

```bash
docker run myapp
```

都可以按照相同方式启动。

因此 Dockerfile 的核心价值不是“写几个 Docker 命令”，而是：

> **把应用构建和运行环境标准化。**

---

## 三、一个 Go 项目的基本结构

我们准备一个最简单的 Go HTTP 服务。

目录：

```text
hello-go/
├── go.mod
├── main.go
└── Dockerfile
```

`main.go`：

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello Docker!")
	})

	fmt.Println("server started at :8080")

	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}
```

初始化 Go 项目：

```bash
go mod init hello-go
```

然后直接运行：

```bash
go run .
```

访问：

```text
http://127.0.0.1:8080
```

得到：

```text
Hello Docker!
```

现在程序已经能够正常运行。

接下来就是：

> **如何把这个 Go 程序制作成 Docker 镜像？**

---

## 四、Dockerfile 最基本的结构

最简单的 Dockerfile 可以这样写：

```dockerfile
FROM ubuntu:22.04

WORKDIR /app

COPY hello-go /app/hello-go

EXPOSE 8080

CMD ["/app/hello-go"]
```

它表达的意思非常直观：

```text
FROM
 ↓
使用什么基础环境？

WORKDIR
 ↓
进入哪个目录？

COPY
 ↓
把什么文件复制进去？

EXPOSE
 ↓
应用使用什么端口？

CMD
 ↓
容器启动的时候运行什么？
```

Dockerfile 实际上就是按照这样的顺序描述：

> **我需要什么环境 → 放入什么文件 → 做什么处理 → 最终运行什么程序。**

---

## 五、FROM：指定基础镜像

最常见的 Dockerfile 第一行：

```dockerfile
FROM ubuntu:22.04
```

表示：

> 基于 Ubuntu 22.04 创建新的镜像。

对于 Go 项目还经常看到：

```dockerfile
FROM golang:1.20
```

它的意义是：

> 使用已经安装好 Go 1.20 的镜像。

例如：

```dockerfile
FROM golang:1.20
```

后面就可以直接执行：

```dockerfile
RUN go build
```

因为基础镜像中已经存在 Go 编译环境。

---

### 常见基础镜像

Go 项目中经常会遇到：

```text
golang
alpine
debian
ubuntu
distroless
scratch
```

它们承担的角色并不完全相同。

例如：

```text
golang
   ↓
适合编译

alpine / debian
   ↓
适合运行

distroless
   ↓
更加精简的运行环境

scratch
   ↓
几乎什么都没有
```

所以实际项目中经常出现：

```text
golang
   ↓
编译
   ↓
alpine / distroless / scratch
   ↓
运行
```

这就是后面要介绍的**多阶段构建**。

---

## 六、WORKDIR：设置工作目录

例如：

```dockerfile
WORKDIR /app
```

之后 Dockerfile 中很多操作都会以：

```text
/app
```

作为当前工作目录。

例如：

```dockerfile
WORKDIR /app

COPY . .

RUN go build -o server .

CMD ["./server"]
```

可以理解为：

```bash
cd /app
```

然后执行后面的操作。

相比：

```dockerfile
RUN cd /app
```

更推荐使用：

```dockerfile
WORKDIR /app
```

因为 `WORKDIR` 会明确设置后续指令使用的工作目录。[Docker Documentation](https://docs.docker.com/reference/dockerfile?utm_source=chatgpt.com)

---

## 七、COPY：复制文件

最常见的写法：

```dockerfile
COPY . .
```

表示：

> 把构建上下文中的文件复制到当前工作目录。

例如：

```dockerfile
WORKDIR /app

COPY . .
```

最终：

```text
宿主机

hello-go/
├── go.mod
├── main.go
└── Dockerfile

        ↓ COPY

容器

/app/
├── go.mod
├── main.go
└── Dockerfile
```

也可以指定具体文件：

```dockerfile
COPY go.mod go.sum ./
```

或者：

```dockerfile
COPY server /app/server
```

---

## 八、RUN：构建镜像时执行命令

例如：

```dockerfile
RUN go build -o server .
```

这条命令非常重要。

它不是：

> 容器启动的时候执行。

而是：

> **构建 Docker 镜像的时候执行。**

也就是：

```bash
docker build
```

过程中执行。

所以：

```dockerfile
RUN go build -o server .
```

最终产生：

```text
/app/server
```

然后这个文件会成为镜像的一部分。

可以简单记：

```text
RUN
↓
docker build 时执行
```

而：

```text
CMD
↓
docker run 时执行
```

---

## 九、CMD：容器启动时默认运行什么

例如：

```dockerfile
CMD ["./server"]
```

意思是：

> 创建容器时，如果没有指定其他命令，就运行 `./server`。

执行：

```bash
docker run hello-go:1.0
```

默认运行：

```text
./server
```

但是 `CMD` 是可以被覆盖的。

例如：

```bash
docker run -it hello-go:1.0 sh
```

此时 Docker 不再使用原来的：

```text
./server
```

而执行：

```text
sh
```

所以可以简单理解：

> **CMD 是默认启动命令。**

Docker 官方文档也明确说明，`CMD` 用于提供容器运行时的默认命令或默认参数。[Docker Documentation](https://docs.docker.com/reference/dockerfile?utm_source=chatgpt.com)

---

## 十、ENTRYPOINT：定义容器的主要入口

例如：

```dockerfile
ENTRYPOINT ["./server"]
```

它和 `CMD` 有一定区别。

如果：

```dockerfile
ENTRYPOINT ["./server"]
```

然后执行：

```bash
docker run hello-go:1.0
```

运行：

```text
./server
```

如果：

```bash
docker run hello-go:1.0 --port=9090
```

那么最终相当于：

```text
./server --port=9090
```

因此可以用一个非常简单的方式理解：

```text
ENTRYPOINT
    ↓
我是谁？

CMD
    ↓
默认给我什么参数？
```

例如：

```dockerfile
ENTRYPOINT ["/app/server"]

CMD ["--port=8080"]
```

默认：

```text
/app/server --port=8080
```

如果：

```bash
docker run hello-go:1.0 --port=9090
```

则：

```text
/app/server --port=9090
```

这也是实际项目中很常见的写法。

---

## 十一、CMD 和 ENTRYPOINT 为什么经常一起使用？

推荐理解成：

```text
ENTRYPOINT = 固定程序

CMD        = 默认参数
```

例如：

```dockerfile
ENTRYPOINT ["/app/server"]

CMD ["--config=/etc/server/config.yaml"]
```

正常运行：

```bash
docker run server:1.0
```

相当于：

```text
/app/server --config=/etc/server/config.yaml
```

如果临时换配置：

```bash
docker run server:1.0 \
    --config=/etc/server/test.yaml
```

则变成：

```text
/app/server --config=/etc/server/test.yaml
```

这种方式特别适合：

```text
Go Web 服务
命令行工具
Agent
Exporter
Worker
```

等类型的程序。

Docker 官方也推荐在需要固定可执行程序时使用 `ENTRYPOINT`，再配合 `CMD` 提供默认参数。[Docker Documentation](https://docs.docker.com/reference/dockerfile?utm_source=chatgpt.com)

---

## 十二、EXPOSE：声明容器端口

例如：

```dockerfile
EXPOSE 8080
```

表示：

> 这个应用预计监听 8080 端口。

但是它**不会自动把端口暴露到宿主机**。

真正进行端口映射：

```bash
docker run \
    -p 8080:8080 \
    hello-go:1.0
```

这里：

```text
宿主机 8080
    ↓
容器 8080
```

因此：

```dockerfile
EXPOSE 8080
```

和：

```bash
-p 8080:8080
```

是两个不同概念。

---

## 十三、ENV：设置环境变量

例如：

```dockerfile
ENV APP_ENV=prod
```

容器启动后：

```bash
echo $APP_ENV
```

得到：

```text
prod
```

Go 程序中也可以读取：

```go
os.Getenv("APP_ENV")
```

得到：

```text
prod
```

因此 `ENV` 更偏向：

> **运行时配置。**

---

## 十四、ARG：构建时参数

例如：

```dockerfile
ARG VERSION=1.0
```

构建：

```bash
docker build \
    --build-arg VERSION=2.0 \
    -t hello-go:2.0 .
```

Dockerfile 中可以使用：

```dockerfile
RUN echo "build version: ${VERSION}"
```

因此可以简单理解：

```text
ARG
 ↓
docker build 阶段

ENV
 ↓
容器运行阶段
```

这也是 Dockerfile 面试中的常见问题。

---

## 十五、USER：不要让业务程序默认使用 root

Docker 容器默认情况下可能以 root 用户运行。

例如：

```dockerfile
USER root
```

对于生产环境的业务程序，更推荐使用非 root 用户。

例如：

```dockerfile
USER 65532:65532
```

具体使用哪个 UID/GID，要根据基础镜像和应用需求决定。

核心思想是：

> **业务程序不需要 root 权限，就不要给它 root 权限。**

这属于容器安全实践的一部分。

---

## 十六、.dockerignore

如果执行：

```dockerfile
COPY . .
```

Docker 会把构建上下文中的文件发送给构建器。

因此建议配合：

```text
.dockerignore
```

例如：

```text
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

node_modules/
```

这样可以避免：

```text
.git
IDE 配置
日志
临时文件
构建产物
测试覆盖率
```

等无用内容进入构建上下文。

---

## 十七、Go 项目为什么特别适合多阶段构建？

这是 Dockerfile 中非常重要的一个概念。

Go 是编译型语言。

编译程序需要：

```text
Go SDK
Go Compiler
源码
依赖
构建工具
```

但程序运行的时候可能只需要：

```text
server
```

如果直接：

```dockerfile
FROM golang:1.20

COPY . .

RUN go build -o server .

CMD ["./server"]
```

最终镜像里面仍然存在完整的：

```text
Go SDK
Go Compiler
源码
依赖
server
```

但是运行一个已经编译好的 Go 程序：

> 根本不需要整个 Go SDK。

所以 Docker 提供了：

> **Multi-stage Build，多阶段构建。**

Docker 官方也明确推荐这种方式用于 Go 等编译型语言：在一个阶段完成编译，再将编译产物复制到最终运行镜像。[Docker Documentation](https://docs.docker.com/get-started/docker-concepts/building-images/multi-stage-builds/?utm_source=chatgpt.com)

---

## 十八、Go 1.20 多阶段构建案例

推荐先理解这个版本：

```dockerfile
# =========================
# Build
# =========================

FROM golang:1.20 AS builder

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build -o server .


# =========================
# Runtime
# =========================

FROM alpine:3.18

WORKDIR /app

COPY --from=builder /app/server ./server

EXPOSE 8080

ENTRYPOINT ["./server"]
```

整个过程：

```text
             Go 1.20
                │
                ▼
        ┌───────────────┐
        │   builder     │
        │               │
        │ go.mod        │
        │ go.sum        │
        │ source code   │
        │ Go compiler   │
        │      ↓        │
        │    server     │
        └───────┬───────┘
                │
          COPY --from
                │
                ▼
        ┌───────────────┐
        │    runtime    │
        │               │
        │    server     │
        │    Alpine     │
        └───────────────┘
```

最终运行镜像不再包含完整 Go 编译环境。

---

## 十九、COPY --from 是什么？

关键代码：

```dockerfile
FROM golang:1.20 AS builder
```

这里给第一个构建阶段起了一个名字：

```text
builder
```

后面：

```dockerfile
COPY --from=builder /app/server ./server
```

意思就是：

> 从 `builder` 这个阶段中，把 `/app/server` 复制到当前阶段。

所以：

```text
AS builder
```

和：

```text
COPY --from=builder
```

是一组非常重要的知识点。

---

## 二十、进一步使用 scratch

如果 Go 程序可以编译成静态二进制，还可以进一步使用：

```dockerfile
FROM golang:1.20 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build -o server .


FROM scratch

COPY --from=builder /app/server /server

EXPOSE 8080

ENTRYPOINT ["/server"]
```

此时最终镜像非常简单：

```text
scratch
  +
server
```

因为 `scratch` 本身几乎没有用户空间工具。

但是它也意味着：

```text
没有 shell
没有 ls
没有 ps
没有 curl
没有 wget
```

因此：

> **镜像越小不等于一定越好。**

生产环境需要在：

```text
镜像体积
安全性
可调试性
运行依赖
```

之间进行权衡。

---

## 二十一、Dockerfile 的构建缓存

再看这部分：

```dockerfile
COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN go build -o server .
```

为什么不直接：

```dockerfile
COPY . .

RUN go mod download

RUN go build -o server .
```

因为 Go 项目开发过程中：

```text
main.go
service.go
handler.go
model.go
```

等源码经常发生变化。

但是：

```text
go.mod
go.sum
```

变化频率相对较低。

因此把：

```dockerfile
COPY go.mod go.sum ./
RUN go mod download
```

放到前面，可以提高 Docker 构建缓存的利用率。

例如：

```text
第一次构建：

COPY go.mod go.sum
       ↓
go mod download
       ↓
COPY source
       ↓
go build


第二次只修改 main.go：

COPY go.mod go.sum
       ↓
缓存命中
       ↓
COPY source
       ↓
重新 go build
```

这样可以避免每次修改代码都重新下载 Go 依赖。

---

## 二十二、一个比较完整的 Go 1.20 Dockerfile

最终可以把前面的知识组合起来：

```dockerfile
# syntax=docker/dockerfile:1

# =========================
# Build Stage
# =========================

FROM golang:1.20 AS builder

WORKDIR /app

# 先复制依赖文件，提高缓存命中率
COPY go.mod go.sum ./

RUN go mod download

# 再复制源代码
COPY . .

# 编译 Linux amd64 二进制
RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
    -ldflags="-s -w" \
    -o /app/server \
    .


# =========================
# Runtime Stage
# =========================

FROM alpine:3.18

WORKDIR /app

# 如果程序需要 HTTPS 请求，保留 CA 证书
RUN apk add --no-cache ca-certificates

COPY --from=builder /app/server /app/server

EXPOSE 8080

# 使用非 root 用户
USER 65532:65532

ENTRYPOINT ["/app/server"]
```

然后：

```bash
docker build -t hello-go:1.0 .
```

运行：

```bash
docker run -d \
    --name hello-go \
    -p 8080:8080 \
    hello-go:1.0
```

查看：

```bash
docker ps
```

查看日志：

```bash
docker logs hello-go
```

测试：

```bash
curl http://127.0.0.1:8080
```

---

## 二十三、Dockerfile 实际开发中的优化思路

掌握 Dockerfile 后，可以从下面几个方向检查：

### 1. 基础镜像

```text
是否使用了不必要的大镜像？
```

### 2. 构建环境和运行环境

```text
是否可以使用多阶段构建？
```

### 3. 构建缓存

```text
COPY 顺序是否合理？
```

### 4. 构建上下文

```text
是否存在 .dockerignore？
```

### 5. 运行权限

```text
是否必须使用 root？
```

### 6. 镜像内容

```text
最终镜像是否包含不需要的编译器、源码、工具？
```

### 7. 启动方式

```text
CMD 和 ENTRYPOINT 是否合理？
```

### 8. 进程信号

```text
应用程序是不是容器的 PID 1？
```

尤其是 Go Web 服务，在 Kubernetes 中运行时，`SIGTERM`、优雅退出等问题最终都可能和容器的进程模型联系起来。

Docker 官方文档也特别指出，`ENTRYPOINT` 的 exec form 更适合将应用作为容器的可执行入口；shell form 可能导致应用不是 PID 1，并影响信号传递。[Docker Documentation](https://docs.docker.com/reference/dockerfile?utm_source=chatgpt.com)

---

## 二十四、Dockerfile 面试题

这一部分可以作为文章最后的独立章节。

### 1. Dockerfile 中 `RUN` 和 `CMD` 有什么区别？

```text
RUN
↓
构建镜像时执行

CMD
↓
容器启动时执行
```

例如：

```dockerfile
RUN go build -o server .
```

是：

```bash
docker build
```

阶段执行。

而：

```dockerfile
CMD ["./server"]
```

是：

```bash
docker run
```

阶段使用。

---

### 2. CMD 和 ENTRYPOINT 有什么区别？

最简单的回答：

```text
CMD：
默认命令 / 默认参数，可以被 docker run 后面的命令覆盖。

ENTRYPOINT：
定义容器的主要执行程序。
```

常见组合：

```dockerfile
ENTRYPOINT ["/app/server"]

CMD ["--port=8080"]
```

默认：

```text
/app/server --port=8080
```

执行：

```bash
docker run app --port=9090
```

变成：

```text
/app/server --port=9090
```

---

### 3. CMD 和 ENTRYPOINT 可以同时使用吗？

可以，而且非常常见。

```dockerfile
ENTRYPOINT ["/app/server"]

CMD ["--config=/etc/app/config.yaml"]
```

理解成：

```text
ENTRYPOINT = 程序

CMD = 默认参数
```

---

### 4. CMD 和 ENTRYPOINT 有哪些写法？

两者都有：

```dockerfile
CMD ["./server"]
```

以及：

```dockerfile
CMD ./server
```

前者是：

> exec form

后者是：

> shell form

对于 Go 服务等长期运行的应用，通常更推荐：

```dockerfile
ENTRYPOINT ["/app/server"]
```

这种 exec form。

---

### 5. COPY 和 ADD 有什么区别？

回答：

> 普通文件复制优先使用 `COPY`。

`ADD` 具有一些额外语义，而 `COPY` 的意图更加明确。

所以一般：

```dockerfile
COPY
```

优先于：

```dockerfile
ADD
```

---

### 6. ARG 和 ENV 有什么区别？

```text
ARG
↓
构建时变量

ENV
↓
运行时环境变量
```

例如：

```dockerfile
ARG VERSION=1.0

ENV APP_ENV=prod
```

---

### 7. EXPOSE 会自动开放端口吗？

不会。

```dockerfile
EXPOSE 8080
```

只是声明应用使用：

```text
8080
```

真正的端口映射：

```bash
docker run -p 8080:8080 app
```

---

### 8. 为什么 Go 项目适合多阶段构建？

因为：

```text
编译阶段
需要 Go SDK + 编译工具

运行阶段
通常只需要 Go 二进制
```

所以：

```text
golang
 ↓
build
 ↓
server
 ↓
alpine / distroless / scratch
```

可以减少最终镜像中的无用内容。[Docker Documentation](https://docs.docker.com/get-started/docker-concepts/building-images/multi-stage-builds/?utm_source=chatgpt.com)

---

### 9. `COPY --from=builder` 是什么？

例如：

```dockerfile
FROM golang:1.20 AS builder
```

定义：

```text
builder
```

构建阶段。

然后：

```dockerfile
COPY --from=builder /app/server /app/server
```

表示：

> 从 builder 阶段复制编译好的 server 到最终镜像。

---

### 10. 为什么 `COPY go.mod go.sum` 要放在 `COPY . .` 前面？

核心是：

> **提高构建缓存命中率。**

因为依赖文件变化频率通常低于业务源代码。

```dockerfile
COPY go.mod go.sum ./
RUN go mod download

COPY . .
```

是比较常见的 Go Dockerfile 写法。

---

### 11. 为什么不推荐业务程序直接使用 root？

因为如果应用存在安全漏洞，攻击者获得 root 权限后，潜在影响更大。

因此生产环境应该尽量：

```dockerfile
USER appuser
```

或者使用合适的非 root UID/GID。

---

### 12. `scratch` 是什么？

可以把：

```dockerfile
FROM scratch
```

理解成：

> **一个几乎什么都没有的空白基础镜像。**

特别适合能够独立运行的静态二进制。

Go 程序如果：

```bash
CGO_ENABLED=0
```

并且没有额外运行时依赖，就比较适合这种模式。

---

### 13. Dockerfile 可以写多个 CMD 吗？

语法上可以出现多个，但实际只有最后一个生效。

例如：

```dockerfile
CMD ["./server"]

CMD ["./test"]
```

最终使用：

```text
./test
```

所以通常应该只保留一个 `CMD`。Docker 官方文档也明确规定，一个 Dockerfile 中只有最后一个 `CMD` 会生效。[Docker Documentation](https://docs.docker.com/reference/dockerfile?utm_source=chatgpt.com)

---

### 14. 为什么生产环境经常看到 `ENTRYPOINT` + `CMD`？

因为这种设计非常适合：

```text
固定程序
+
可覆盖的默认参数
```

例如：

```dockerfile
ENTRYPOINT ["/app/server"]

CMD ["--config=/etc/app/config.yaml"]
```

这样既能固定应用，又允许部署环境覆盖默认参数。

---

## 二十五、最后形成一个 Dockerfile 学习框架

如果把本文内容压缩成一张图，可以记成：

```text
                       Dockerfile
                           │
          ┌────────────────┼────────────────┐
          │                │                │
        构建              文件             运行
          │                │                │
       FROM              COPY            CMD
       RUN               ADD             ENTRYPOINT
       ARG               .dockerignore   ENV
                                          EXPOSE
                                          USER
          │                │                │
          └────────────────┼────────────────┘
                           ↓
                     多阶段构建
                           ↓
              ┌────────────┴────────────┐
              ↓                         ↓
          Build Stage              Runtime Stage
          golang:1.20              alpine
              ↓                     distroless
           go build                 scratch
              ↓
          server
              │
              └────── COPY --from ──────→
```

真正掌握 Dockerfile，重点不是记住几十个指令，而是理解四件事情：

> **1. 怎么构建程序**  
> **2. 怎么把程序放进镜像**  
> **3. 怎么让容器启动程序**  
> **4. 怎么让最终镜像尽可能合理、安全、易维护**


