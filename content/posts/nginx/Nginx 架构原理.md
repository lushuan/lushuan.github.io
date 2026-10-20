---
title: "Nginx 架构原理"
subtitle: ""
date: 2022-01-16T12:06:37+08:00
lastmod: 2022-06-29T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Nginx"]
tags: ["Nginx"]
slug: "nginx-arch"
featuredImage: "/images/nginx/cover.jpg"
summary: "深入解析 Nginx 的 master/worker 多进程模型与事件驱动架构，讲解进程信号管理、reload 热更新与热升级机制以及模块化体系。"
---

本文深入解析 Nginx 的架构原理，涵盖 master/worker 多进程模型、事件驱动的异步非阻塞处理机制与进程信号管理，并详细讲解 Nginx reload 配置热更新、程序热升级的实现原理以及模块化体系结构。

<!--more-->

## 一、基本原理
### Nginx 进程模型
Nginx 启动后以 daemon 形式在后台运行，后台进程包含一个 master 进程和多个 worker 进程。如下图所示：
```
[root@www html]# ps -ef --forest|grep nginx|grep -v grep
root      59984      1  0 13:23 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx     59985  59984  0 13:23 ?        00:00:00  \_ nginx: worker process
nginx     59986  59984  0 13:23 ?        00:00:00  \_ nginx: worker process
nginx     59987  59984  0 13:23 ?        00:00:00  \_ nginx: worker process
nginx     59988  59984  0 13:23 ?        00:00:00  \_ nginx: worker process

```
Nginx 是由一个 master 管理进程，多个 worker 进程处理工作的多进程模型。基础架构设计，如下图所示：

![nginx-architecture](/images/nginx/nginx-architecture.png "Nginx 架构")

Master 负责管理 worker 进程，worker 进程负责处理网络事件。整个框架被设计为一种依赖事件驱动、异步、非阻塞的模式。

why ——为什么选择 master/worker 这种模型？

如此设计的优点有：

1. 可以充分利用多核机器，增强并发处理能力。
2. 多 worker 间可以实现负载均衡。
3. Master 监控并统一管理 worker 行为。在 worker 异常后，可以主动拉起 worker 进程，从而提升了系统的可靠性。并且由 Master 进程控制服务运行中的程序升级、配置项修改等操作，从而增强了整体的动态可扩展与热更的能力。

### Nginx 进程结构

![image-20240722141618078](/images/nginx/nginx-process-structure.png "Nginx 进程结构")

进程结构分类
1. 单进程结构 适用于开发测试环境
2. 多进程结构 适用于生产环境

- 所有的 worker 进程是处理真正的请求的
- master 进程是监控 worker 进程是不是在正常的工作
- 缓存是要在多个进程间进行共享的
- 这些进程间的通讯都是使用共享内存来解决的

问题：

Nginx 为什么会有多个 worker 进程？
	因为 Nginx 是采用事件驱动的模型以后，它希望每一个 worker 进程从头到尾占有一颗 CPU，所以我们要把 worker 进程的数量配置和服务器上的 CPU 核数一致

Nginx 服务器，正常运行过程中：

1. 多进程：一个 Master 进程、多个 Worker 进程
2. Master 进程：管理 Worker 进程
      1. 对外接口：接收外部的操作（信号）
      2. 对内转发：根据外部的操作的不同，通过信号管理 Worker
      3. 监控：监控 worker 进程的运行状态，worker 进程异常终止后，自动重启 worker 进程
3. Worker 进程：所有 Worker 进程都是平等的
      1. 实际处理：网络请求，由 Worker 进程处理；
      2. Worker 进程数量：在 nginx.conf 中配置，一般设置为核心数，充分利用 CPU 资源，同时，避免进程数量过多，避免进程竞争 CPU 资源，增加上下文切换的损耗。

HTTP 连接建立和请求处理过程：

1. Nginx 启动时，Master 进程，加载配置文件
2. Master 进程，初始化监听的 socket
3. Master 进程，fork 出多个 Worker 进程
4. Worker 进程，竞争新的连接，获胜方通过三次握手，建立 Socket 连接，并处理请求

Nginx 高性能、高并发：

1. Nginx 采用：多进程 + 异步非阻塞方式（IO 多路复用 epoll）
2. 请求的完整过程：
    1. 建立连接
    2. 读取请求：解析请求
    3. 处理请求
    4. 响应请求
3. 请求的完整过程，对应到底层，就是：读写 socket 事件
#### 进程结构示例演示
可以看到当执行 `nginx -s reload` 重载命令后，会重新生成三个进程，进程 id`59985 59986 59987 59988` 变更为 `62907 62908 62909 62910`，同理执行 `kill -SIGHUP 59984` 是一样的效果
```
[root@www ~]# ps -ef --forest|grep nginx|grep -v grep
root      59984      1  0 13:24 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx     59985  59984  0 13:24 ?        00:00:00  \_ nginx: worker process
nginx     59986  59984  0 13:24 ?        00:00:00  \_ nginx: worker process
nginx     59987  59984  0 13:24 ?        00:00:00  \_ nginx: worker process
nginx     59988  59984  0 13:24 ?        00:00:00  \_ nginx: worker process
[root@www ~]# nginx -s reload
[root@www ~]# ps -ef --forest|grep nginx|grep -v grep
root      59984      1  0 13:24 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx     62907  59984  0 17:26 ?        00:00:00  \_ nginx: worker process
nginx     62908  59984  0 17:26 ?        00:00:00  \_ nginx: worker process
nginx     62909  59984  0 17:26 ?        00:00:00  \_ nginx: worker process
nginx     62910  59984  0 17:26 ?        00:00:00  \_ nginx: worker process

```
单独发送终止信号终止一个 `worker` 进程会发生什么？
```
[root@www ~]# ps -ef --forest|grep nginx|grep -v grep
root      59984      1  0 13:24 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx     62918  59984  0 17:31 ?        00:00:00  \_ nginx: worker process
nginx     62919  59984  0 17:31 ?        00:00:00  \_ nginx: worker process
nginx     62920  59984  0 17:31 ?        00:00:00  \_ nginx: worker process
nginx     62921  59984  0 17:31 ?        00:00:00  \_ nginx: worker process
[root@www ~]# kill -SIGTERM 62918
[root@www ~]# ps -ef --forest|grep nginx|grep -v grep
root      59984      1  0 13:24 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx     62919  59984  0 17:31 ?        00:00:00  \_ nginx: worker process
nginx     62920  59984  0 17:31 ?        00:00:00  \_ nginx: worker process
nginx     62921  59984  0 17:31 ?        00:00:00  \_ nginx: worker process
nginx     62928  59984  0 17:33 ?        00:00:00  \_ nginx: worker process
```
### Master 进程
#### 核心逻辑
master 进程的主逻辑在 ngx_master_process_cycle，核心关注源码：
```
ngx_master_process_cycle(ngx_cycle_t cycle)
{
    ...
    ngx_start_worker_processes(cycle, ccf-worker_processes,
                                        NGX_PROCESS_RESPAWN);
    ...


    for ( ;; ) {
        if (delay) {...}

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle-log, 0, sigsuspend);

        sigsuspend(&set);

        ngx_time_update();

        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle-log, 0,
                             wake up, sigio %i, sigio);

        if (ngx_reap) {
            ngx_reap = 0;
            ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle-log, 0, reap children);
            live = ngx_reap_children(cycle);
        }

        if (!live && (ngx_terminate  ngx_quit)) {...}

        if (ngx_terminate) {...}

        if (ngx_quit) {...}

        if (ngx_reconfigure) {...}

        if (ngx_restart) {...}

        if (ngx_reopen) {...}

        if (ngx_change_binary) {...}

        if (ngx_noaccept) {
            ngx_noaccept = 0;
            ngx_noaccepting = 1;
            ngx_signal_worker_processes(cycle,
                                                  ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
        }
    }
 }
```
由上述代码，可以理解，master 进程主要用来管理 worker 进程，具体包括如下 4 个主要功能：

1.接受来自外界的信号。其中 master 循环中的各项标志位就对应着各种信号，如：ngx_quit 代表 QUIT 信号，表示优雅的关闭整个服务。

2.向各个 worker 进程发送信号。比如 ngx_noaccept 代表 WINCH 信号，表示所有子进程不再接受处理新的连接，由 master 向所有的子进程发送 QUIT 信号量。

3.监控 worker 进程的运行状态。比如 ngx_reap 代表 CHILD 信号，表示有子进程意外结束，这时需要监控所有子进程的运行状态，主要由 ngx_reap_children 完成。

4.当 worker 进程退出后（异常情况下），会自动重新启动新的 worker 进程。主要也是在 ngx_reap_children

### Nginx 进程管理：信号
多进程之间通讯可以使用信号、共享内存等，当进行 Nginx 进程间的管理时通常只使用信号

![image-20240722142156195](/images/nginx/nginx-process-signal.png "Nginx 进程管理：信号")

这里可以看到 worker 进程接收到的信号和 master 进程接收到信号基本上是一一对应的，为什么不直接对 worker 进程发送信号呢，是因为需要通过 master 进程来进行管理，这也是正常的处理方式。
## 二、Nginx reload 热更新

![image-20240722151323960](/images/nginx/nginx-reload.png "Nginx reload热更新")

### 不停机载入新配置

![image-20240722151535108](/images/nginx/nginx-without-restart-and-reload.png "Nginx 不停机载入新配置")
### 热重载-配置热更

![nginx-hot-configure-reload](/images/nginx/nginx-hot-configure-reload.jpg "Nginx 热重载-配置热更")

Nginx 热更配置时，可以保持运行中平滑更新配置，具体流程如下：

- 更新 nginx.conf 配置文件，向 master 发送 SIGHUP 信号或执行 nginx -s reload
- Master 进程使用新配置，启动新的 worker 进程
- 使用旧配置的 worker 进程，不再接受新的连接请求，并在完成已存在的连接后退出

### Nginx 热升级-程序热更
![nginx-hot-configure-reload](/images/nginx/nginx-hot-configure-reload-2.jpg "Nginx 版本热升级")

Nginx 热升级过程如下：

1. 将旧 Nginx 文件换成新 Nginx 文件（注意备份）
2. 向 master 进程发送 USR2 信号（平滑升级到新版本的 Nginx 程序）
3. master 进程修改 pid 文件号，加后缀.oldbin
4. master 进程用新 Nginx 文件启动新 master 进程，此时新老 master/worker 同时存在。
5. 向老 master 发送 WINCH 信号，关闭旧 worker 进程，观察新 worker 进程工作情况。若升级成功，则向老 master 进程发送 QUIT 信号，关闭老 master 进程；若升级失败，则需要回滚，向老 master 发送 HUP 信号（重读配置文件），向新 master 发送 QUIT 信号，关闭新 master 及 worker。

## 三、Worker 进程
核心逻辑

worker 进程的主逻辑在 ngx_worker_process_cycle，核心关注源码：
```
ngx_worker_process_cycle(ngx_cycle_t cycle, void data)
{
    ngx_int_t worker = (intptr_t) data;

    ngx_process = NGX_PROCESS_WORKER;
    ngx_worker = worker;

    ngx_worker_process_init(cycle, worker);

    ngx_setproctitle(worker process);

    for ( ;; ) {

        if (ngx_exiting) {...}

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle-log, 0, worker cycle);

        ngx_process_events_and_timers(cycle);

        if (ngx_terminate) {...}

        if (ngx_quit) {...}

        if (ngx_reopen) {...}
    }
}
```
由上述代码，可以理解，worker 进程主要在处理网络事件，通过 ngx_process_events_and_timers 方法实现，其中事件主要包括：网络事件、定时器事件。

## 四、Nginx 的事件处理模型
request：Nginx 中 HTTP 请求。

基本的 HTTP Web Server 工作模式：

1. 接收请求：逐行读取请求行和请求头，判断有请求体后，读取请求体
2. 处理请求
3. 返回响应：根据处理结果，生成相应的 HTTP 响应（响应行、响应头、响应体）

Nginx 也是这个套路，整体流程一致。

![nginx-request-process](/images/nginx/nginx-request-process.png "Nginx 请求流程")
## 五、Nginx 模块
### Nginx 模块化体系结构

![nginx-architecture-2](/images/nginx/nginx-architecture-2.png "Nginx 模块化体系结构")
Nginx 的模块根据其功能基本上可以分为以下几种类型：

- event module: 搭建了独立于操作系统的事件处理机制的框架，及提供了各具体事件的处理。包括 ngx_events_module， ngx_event_core_module 和 ngx_epoll_module 等。Nginx 具体使用何种事件处理模块，这依赖于具体的操作系统和编译选项。
- phase handler: 此类型的模块也被直接称为 handler 模块。主要负责处理客户端请求并产生待响应内容，比如 ngx_http_static_module 模块，负责客户端的静态页面请求处理并将对应的磁盘文件准备为响应内容输出。
- output filter: 也称为 filter 模块，主要是负责对输出的内容进行处理，可以对输出进行修改。例如，可以实现对输出的所有 HTML 页面增加预定义的 footbar 一类的工作，或者对输出的图片的 URL 进行替换之类的工作。
- upstream: upstream 模块实现反向代理的功能，将真正的请求转发到后端服务器上，并从后端服务器上读取响应，发回客户端。upstream 模块是一种特殊的 handler，只不过响应内容不是真正由自己产生的，而是从后端服务器上读取的。
- load-balancer: 负载均衡模块，实现特定的算法，在众多的后端服务器中，选择一个服务器出来作为某个请求的转发服务器。
### Nginx 模块介绍
Nginx 一个非常重要的特性就是拥有丰富的模块，有核心的模块，拓展的模块和第三方拓展模块。
Nginx 模块主要可以分为以下几类：

核心模块：
1. HTTP 模块：用来发布 HTTP web 服务网站的模块。
2. event 模块：用来处理 Nginx 访问请求，并进行回复。

基本模块：
- HTTP Access 模块：用来进行虚拟主机发布访问模块，起到记录访问日志。
- HTTP FastCGI 模块：用于和 PHP 程序进行交互的模块，负责将来访问 Nginx 的 PHP 请求转发到后端的 PHP 上。
- HTTP Proxy 模块：配置反向代理转发的模块，负责向后端传递参数。
- HTTP Rewrite 模块：支持 Rewrite 规则重写，支持域名跳转。

Nginx 模块中的内聚和抽象

![image-20240722155619491](/images/nginx/nginx-module-inner-and-outer.png "Nginx 模块中的内聚和抽象")

Nginx 模块分类

![image-20240722155747512](/images/nginx/nginx-diffrent-modules.png "Nginx 模块分类")

模块的使用：

如我们想要打开 gzip 压缩，可以在 Nginx 模块官网上找到对应的[模块参考](https://nginx.org/en/docs/)`ngx_http_gzip_module`，查看使用示例

## 六、Nginx 进程间架构图

![image-20240722155037255](/images/nginx/nginx-interprocess.png "Nginx 进程间架构图")

## 七、附录
- [百万并发下 Nginx 的优化之道](https://zhuanlan.zhihu.com/p/49415781)
- [Nginx vs Apache](https://www.oschina.net/translate/nginx-vs-apache)

## 八、参考

- https://aosabook.org/en/v2/nginx.html
- https://www.fatalerrors.org/a/analysis-of-nginx-architecture.html
- [Nginx 平台初探](https://tengine.taobao.org/book/chapter_02.html)
- https://www.bilibili.com/video/BV1KP4y1M75J?p=5&vd_source=7e8c74e6ef5f4645e4247ef4120b7971
- https://www.bilibili.com/video/BV1rY411E795/?spm_id_from=333.788.recommend_more_video.0&vd_source=7e8c74e6ef5f4645e4247ef4120b7971