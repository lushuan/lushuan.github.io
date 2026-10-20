---
title: "博客文章 Front Matter 规范化批量整理记录"
subtitle: ""
date: 2026-10-19T10:00:00+08:00
lastmod: 2026-10-19T10:00:00+08:00
draft: true
toc:
  enable: true
  auto: true
weight: false
categories: ["Hugo"]
tags: ["Hugo","LoveIt","Front Matter"]
slug: "frontmatter-cleanup-record"
summary: "记录一次对 content/posts 全站文章的批量规范化整理：统一 summary、toc、slug、featuredImage 字段，正文补充摘要分隔符，并将 Permalinks 切换为 slug 路由。"
---

本文记录 2026-10-19 对博客 `content/posts` 下全部文章进行的一次批量 Front Matter 规范化整理，包括整理规则、执行范围、封面图片方案、Permalinks 调整以及每篇文章的 slug 明细，便于后续新增文章时保持一致的写法。

<!--more-->

## 一、整理规则

本次整理对每篇文章执行以下 5 项规则：

1. **description → summary**：Front Matter 中如存在 `description` 字段则移除，统一改为 `summary` 字段（一到两句中文摘要）。经逐篇解析，全部文章的 Front Matter 中均无 `description` 字段（此前 grep 命中的 5 处均为正文代码块中的示例内容，如 Chart.yaml 模板），故实际无需删除，统一新增了 `summary`。
2. **正文摘要 + 分隔符**：阅读正文后，在文章最前面插入一段 2-4 句的中文摘要，随后以 `<!--more-->` 分隔，用于列表页摘要展示与手动分页。
3. **toc 补全 auto**：`toc` 统一规范为：

   ```yaml
   toc:
     enable: true
     auto: true
   ```

   已有 `auto: true` 的跳过；同时修复了约 10 篇文章中 `enable: true` 顶格无缩进的坏 YAML（涉及 Git、windows、kubeedge、gitlab、mysql、kubernetes、linux 等目录）。
4. **引入 slug**：为每篇文章添加简短的英文 kebab-case `slug`（基于主题而非文件名，如 `dockerfile-guide`、`k8s-rbac`、`nginx-arch`），已校验全站 115 个 slug 无重复。
5. **封面图片 featuredImage**：按主题下载封面图至 `static/images/<topic>/cover.jpg`，内容充实的教程/指南/深度解析文章通过 `featuredImage: "/images/<topic>/cover.jpg"` 引入；篇幅很短的排障笔记、报错记录、单点技巧片段（共 29 篇）判断为无必要引入封面，未添加。

## 二、执行范围

- 处理范围：`content/posts` 下全部目录，共 **114 篇**文章（docker 8、Git 2、gpu 1、openssh 1、windows 1、kubernetes 27、linux 35、ansible 3、cicd 4、devops 2、helm 4、hugo 1、influxdb 1、k3s 1、kafka 1、kubeedge 1、grafana 1、mysql 5、nginx 6、prometheus 2、python 4、redis 1、学习方法 1、安全 1）。
- 跳过：`about/about.md` 及 hugo 目录下 3 篇演示文章（`demo.md`、`first_post.md`、`second.md`）；`docker/Docker入门指南.md` 此前已是目标格式，作为参考样例未改动。
- 其中 **86 篇**（含样例 1 篇）引入了 `featuredImage`，**29 篇**短笔记未引入。

## 三、封面图片方案

新增 23 个主题的封面图，统一保存为 `static/images/<topic>/cover.jpg`（`docker/cover.jpg` 原已存在）：

- 主题清单：ansible、cicd、devops、git、gpu、grafana、helm、hugo、influxdb、k3s、kafka、kubeedge、kubernetes、linux、methodology（学习方法）、mysql、nginx、openssh、prometheus、python、redis、security（安全）、windows。
- 图片要求：横版、宽 ≥1000px、JPEG，全部通过文件头与大小校验。
- 来源说明：Wikimedia/Unsplash/Pexels 在当前网络环境不可达，最终从 wallpapercave.com 按主题合集下载；其中 cicd、grafana、helm、hugo、influxdb、k3s、kubeedge、methodology、openssh、redis 等无同名合集的主题，从相关合集（devops、数据分析、云计算、IoT、数据库等）间接选图，如不满意可单独替换。

## 四、Permalinks 调整

配合 slug 生效，`config.toml` 的 Permalinks 由 `posts = ":filename"` 调整为：

```toml
[Permalinks]
  posts = "/:slug/"
```

调整后文章 URL 变为简短英文 slug（如 `/k8s-rbac/`）。**注意**：全站既有文章链接随之变化，如已发布过旧 URL，需自行在部署侧配置重定向。无 slug 的页面（about、演示文章）自动回退为原标题/文件名路由。

## 五、校验结果

- `hugo` 构建通过（exit 0，无警告、无 Front Matter 解析错误）；
- 全站 slug 无重复（115 个）；
- 所有 `featuredImage` 引用的图片文件均已存在；
- 所有处理过的文章均包含 `summary`、`toc.auto: true` 与 `<!--more-->`。

## 六、slug 明细

| 文章 | slug | 封面 |
|---|---|---|
| Git/Git 2.x 入门与日常使用教程 | git-tutorial | /images/git/cover.jpg |
| Git/提交代码后发起Merge全流程示例 | git-merge-workflow | /images/scenery/cover-5.jpg |
| ansible/Ansible快速入门 | ansible-quickstart | /images/ansible/cover.jpg |
| ansible/Ansible知识图谱 | ansible-knowledge-map | 无 |
| ansible/ansible权限认证报错问题记录 | ansible-ssh-auth-error | 无 |
| cicd/gitlab/基于 Gitlab 构建 Gin-Vue-Admin 后端 server 服务CICD | gitlab-cicd-gva | /images/cicd/cover.jpg |
| cicd/jenkins/Jenkins Pipeline | jenkins-pipeline | /images/cicd/cover.jpg |
| cicd/jenkins/Jenkins 入门介绍 | jenkins-intro | /images/cicd/cover.jpg |
| cicd/jenkins/使用Blue Ocean创建一个简单的流水线 | jenkins-blue-ocean-pipeline | /images/cicd/cover.jpg |
| devops/10 Must-Know Free DevOps Tools for Professional Success | devops-free-tools | /images/devops/cover.jpg |
| devops/Gotenberg 工程实践：从前端 URL 到可控 PDF 导出的完整落地 | gotenberg-pdf-export | /images/devops/cover.jpg |
| docker/CGroups与Namespaces | cgroups-namespaces-hands-on | /images/docker/cover.jpg |
| docker/CRI-Dockerd 完全指南 | cri-dockerd-guide | /images/docker/cover.jpg |
| docker/Docker Engine 与 Docker Compose 版本兼容性全解析 | docker-compose-version-compatibility | /images/docker/cover.jpg |
| docker/Dockerfile从入门到生产实践 | dockerfile-guide | /images/docker/cover.jpg |
| docker/Docker不是魔法：从"会敲命令"到"真正理解容器"的认知跃迁 | docker-internals-explained | /images/docker/cover.jpg |
| docker/Docker入门指南 | docker-guide | /images/docker/cover.jpg |
| docker/从Docker至Containerd | docker-to-containerd | 无 |
| docker/基于 Docker Compose 部署 MySQL 8.0 生产实践指南 | docker-compose-mysql | /images/docker/cover.jpg |
| docker/容器没有魔法：一堵墙、一块水表，和一场认知跃迁 | container-no-magic | /images/docker/cover.jpg |
| gpu/Linux系统Nvidia驱动部署 | nvidia-driver-install | /images/gpu/cover.jpg |
| grafana/Grafana | grafana-intro | /images/scenery/cover-1.jpg |
| helm/Helm_Charts 介绍 | helm-charts-intro | /images/helm/cover.jpg |
| helm/Helm修改configmap后应用自动重启生效 | helm-configmap-restart | 无 |
| helm/Helm生产实践记录 | helm-production-practice | /images/helm/cover.jpg |
| helm/helm_charts编写模板说明 | helm-chart-template-guide | /images/helm/cover.jpg |
| hugo/Hugo LoveIt 主题迁移 GitHub Actions 实战 | hugo-loveit-actions | /images/scenery/cover-2.jpg |
| influxdb/Influxdb 配置鉴权 | influxdb-auth | 无 |
| k3s/K3s快速入门及安装部署 | k3s-setup | /images/scenery/cover-6.jpg |
| kafka/kafka介绍 | kafka-intro | /images/scenery/cover-3.jpg |
| kubeedge/KubeEdge v1.17.4 amd64结合arm64云边架构环境搭建 | kubeedge-setup | /images/kubeedge/cover.jpg |
| kubernetes/cni/calico三层namesapce架构浅析 | calico-three-namespace-design | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes_api访问控制/一文了解Kubernetes RBAC | k8s-rbac | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes原理/当 Kubernetes 创建 Pod 时发生了什么 | what-happens-when-k8s-creates-pod | 无 |
| kubernetes/kubernetes最佳实践/k8s容器运行时containerd升级 | containerd-offline-upgrade | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes最佳实践/kube-dns切换为coredns实践 | kube-dns-to-coredns | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes最佳实践/kubelet目录满问题记录 | kubelet-dir-disk-full | 无 |
| kubernetes/kubernetes最佳实践/kubernetes常用命令整理 | kubectl-commands-cheatsheet | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes最佳实践/ubuntu-server部署k8s1.28(containerd版本)集群 | k8s-128-containerd-ubuntu | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes最佳实践/调整节点pod驱逐时间 | tune-pod-eviction-timeout | 无 |
| kubernetes/kubernetes理论与故障排查/K8s_v1.20.X创建PVC报错 | pvc-pending-selflink-v120 | 无 |
| kubernetes/kubernetes理论与故障排查/K8s删除Namesapce一直处于Terminating状态处理 | namespace-terminating-stuck | 无 |
| kubernetes/kubernetes理论与故障排查/Kubelet 系统污点引发的 CNI 部署死锁排查实录 | cni-taint-deadlock-calico | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes理论与故障排查/Kubernetes 网络排错 | k8s-network-debug | 无 |
| kubernetes/kubernetes理论与故障排查/k8s 常见故障和解决思路及方法 | k8s-common-failures-troubleshooting | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes理论与故障排查/kubeadm安装k8s集群报错提示kubelet未运行 | kubeadm-kubelet-cgroup-driver | 无 |
| kubernetes/kubernetes理论与故障排查/kubernetes pod排障指南 | pod-troubleshooting-guide | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes理论与故障排查/kubernetes通过nfs动态创建pv异常 | nfs-dynamic-pv-selflink | 无 |
| kubernetes/kubernetes理论与故障排查/网络模式变更为什么会洗残k8s集群 | vm-network-change-k8s-repair | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes理论与故障排查/记录一次apiserver和etcd连接的随机端口占用冲突问题 | apiserver-etcd-port-conflict | 无 |
| kubernetes/kubernetes理论与故障排查/记录一次生产环境故障分析 | prod-outage-coredns-taint | 无 |
| kubernetes/kubernetes理论与故障排查/通用生产级故障复盘（Post-Mortem）博客模板 | post-mortem-blog-template | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes知识笔记/K8s默认Service和Headless Service的区别 | service-vs-headless-service | 无 |
| kubernetes/kubernetes知识笔记/Kubernetes Addons 全景图：大白话版 | k8s-addons-overview | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes知识笔记/Kubernetes ConfigMap多文件挂载至同一个pod内目录实践 | configmap-multi-file-mount | /images/kubernetes/cover.jpg |
| kubernetes/kubernetes调度器/Pod调度 | pod-scheduling-affinity | /images/kubernetes/cover.jpg |
| kubernetes/kuberntes入门/Kubernetes 技能图谱 | k8s-skill-roadmap | 无 |
| kubernetes/kuberntes入门/Kubernetes的基本概念和术语 | k8s-basic-concepts | /images/kubernetes/cover.jpg |
| linux/work-with-shell/ch0_shell分类 | shell-ch0-types | /images/scenery/cover-7.jpg |
| linux/work-with-shell/ch1_入参和默认变量 | shell-ch1-args | 无 |
| linux/work-with-shell/ch2_变量 | shell-ch2-variables | /images/scenery/cover-15.jpg |
| linux/work-with-shell/ch3_命令执行 | shell-ch3-command-execution | 无 |
| linux/work-with-shell/ch4_条件分支 | shell-ch4-conditions | /images/scenery/cover-16.jpg |
| linux/work-with-shell/ch5_循环 | shell-ch5-loops | /images/scenery/cover-17.jpg |
| linux/work-with-shell/ch6_函数 | shell-ch6-functions | 无 |
| linux/work-with-shell/ch7_返回值 | shell-ch7-return-values | 无 |
| linux/work-with-shell/ch8_注释 | shell-ch8-comments | 无 |
| linux/work-with-shell/ch9_日志保存 | shell-ch9-logging | 无 |
| linux/work-with-shell/ch10_脚本执行 | shell-ch10-script-execution | 无 |
| linux/work-with-shell/ch11_脚本退出码 | shell-ch11-exit-codes | 无 |
| linux/work-with-shell/ch12_数组 | shell-ch12-arrays | /images/scenery/cover-8.jpg |
| linux/work-with-shell/ch13_字典 | shell-ch13-dictionaries | /images/scenery/cover-9.jpg |
| linux/work-with-shell/ch14_test条件判断 | shell-ch14-test | /images/scenery/cover-10.jpg |
| linux/work-with-shell/ch15_参数处理 | shell-ch15-argument-handling | /images/scenery/cover-11.jpg |
| linux/work-with-shell/ch16_表达式和算术 | shell-ch16-arithmetic | 无 |
| linux/work-with-shell/ch17_文件和重定向 | shell-ch17-redirection | /images/scenery/cover-12.jpg |
| linux/work-with-shell/ch18_常用脚本 | shell-ch18-common-scripts | /images/scenery/cover-13.jpg |
| linux/work-with-shell/ch19_字符串操作详解 | shell-ch19-string-operations | /images/scenery/cover-14.jpg |
| linux/排障瑞士军刀/Linux系统性能问题定位 | linux-performance-tuning | /images/linux/cover.jpg |
| linux/排障瑞士军刀/Linux网络排障 | linux-network-troubleshooting | /images/linux/cover.jpg |
| linux/文本三剑客/awk | awk-guide | /images/linux/cover.jpg |
| linux/文本三剑客/grep | grep-guide | /images/linux/cover.jpg |
| linux/文本三剑客/文本处理三剑客之SED | sed-guide | /images/linux/cover.jpg |
| linux/知识库/LVM逻辑卷组管理 | lvm-management | /images/scenery/cover-19.jpg |
| linux/知识库/Linux 后台启动命令setsid和nohup的区别介绍 | setsid-vs-nohup | /images/linux/cover.jpg |
| linux/知识库/Linux时间校正 | linux-time-sync | /images/linux/cover.jpg |
| linux/知识库/Linux防火墙Iptables详解 | iptables-detail | /images/linux/cover.jpg |
| linux/知识库/Shell 脚本中最反直觉的陷阱 | shell-exit-code-trap | /images/linux/cover.jpg |
| linux/知识库/fdisk磁盘管理详解 | fdisk-guide | /images/scenery/cover-18.jpg |
| linux/知识库/getent 命令详解：为什么部署脚本都爱用它做幂等检查 | getent-guide | /images/linux/cover.jpg |
| linux/知识库/rsync结合inotify实现静态资源实时同步 | rsync-inotify-sync | /images/linux/cover.jpg |
| linux/知识库/xargs原理及使用 | xargs-guide | /images/scenery/cover-20.jpg |
| linux/知识库/用了Linux这么多年，开机那几秒到底发生了什么？ | linux-boot-process | /images/linux/cover.jpg |
| mysql/MySQL 8.0 字符存储机制和字段长度设计规范参考 | mysql-charset-design | /images/mysql/cover.jpg |
| mysql/MySQL 8.0 表设计字段类型与长度选型标准化模板 | mysql-schema-template | /images/mysql/cover.jpg |
| mysql/MySQL集群之主从部署 | mysql-replication | /images/mysql/cover.jpg |
| mysql/MySql数据库备份和还原 | mysql-backup-restore | /images/mysql/cover.jpg |
| mysql/XtraBackup | mysql-xtrabackup | /images/mysql/cover.jpg |
| nginx/Nginx 架构原理 | nginx-arch | /images/nginx/cover.jpg |
| nginx/Nginx介绍-快速了解 | nginx-intro | /images/nginx/cover.jpg |
| nginx/Nginx安装部署 | nginx-install | /images/nginx/cover.jpg |
| nginx/Nginx常用配置 | nginx-common-config | /images/nginx/cover.jpg |
| nginx/Nginx日常工作常用脚本 | nginx-daily-scripts | 无 |
| nginx/基于 Nginx 构建高性能静态镜像下载站：从配置到生产实践 | nginx-static-mirror-site | /images/nginx/cover.jpg |
| openssh/OpenSSH_v9.9p2源码离线一键升级 | openssh-offline-upgrade | /images/openssh/cover.jpg |
| prometheus/PromQL | promql-guide | /images/scenery/cover-2.jpg |
| prometheus/Prometheus监控k8s指标和监控组件策略 | prometheus-k8s-metrics | /images/scenery/cover-2.jpg |
| python/Python 快速上手指南_项目目录 | python-project-layout | 无 |
| python/Python3 快速上手指南 | python-quickstart | /images/python/cover.jpg |
| python/django/Django 应用容器化构建 | django-docker-build | /images/python/cover.jpg |
| python/django/Django 模板语法 | django-templates | /images/python/cover.jpg |
| redis/Redis知识体系 | redis-overview | /images/redis/cover.jpg |
| windows/BAT 批处理系统梳理从基础到工程脚本落地 | bat-scripting | /images/windows/cover.jpg |
| 学习方法/欲罢不能的学习方法策略 | learning-methods | 无 |
| 安全/web安全/跨站请求伪造CSRF攻击的原理以及防范措施 | csrf-defense | /images/security/cover.jpg |

## 七、第二轮调整：封面相关性复检与 h2 序号

### 7.1 封面相关性复检

逐张目检 24 个主题封面后，按"封面须与目录/标签主题关联"的原则调整：

- **保留**（画面含主题 logo 或强关联视觉）：ansible、cicd、docker、git、gpu、kubernetes、methodology、nginx、python、security。
- **替换为主题相关图**（原地覆盖 `static/images/<topic>/cover.jpg`）：devops（DevOps 无限环）、helm（船舵）、influxdb（数据库 ER 图）、k3s（Kubernetes logo，按用户要求移除树莓派图）、kubeedge（K8s 分布式节点）、linux（Tux 企鹅，符合 Linux/红帽相关要求）、mysql（海豚 logo）、openssh（SSH 命令速查表）、redis（服务器机柜）、windows（Windows 旗帜 logo）。
- **改用风景图**（找不到关联图的主题，引用 `static/images/scenery/`）：grafana → cover-1、hugo → cover-2、kafka → cover-3、prometheus → cover-2。

### 7.2 二级标题序号补全

对 h2（`## `）无序号的文章按出现顺序补充中文序号（一、二、三……），已有序号（中文或阿拉伯数字）或无 h2 的文章跳过；代码块内的 `##` 行不受影响。共 **80 篇**添加序号（linux 26、kubernetes 20、其余目录 34），**34 篇**跳过（20 篇已有序号、14 篇无 h2）。

## 八、第三轮调整：用户标记封面更换为风景图

用户对 linux/shell 等标签下不满意的封面在 Front Matter 中以 `featuredImage-:` 形式标记（共 16 篇：linux/work-with-shell 12 篇、linux/知识库 3 篇、Git 合并流程 1 篇、k3s 1 篇）。处理方式：

- 新下载 16 张互不重复的平静风景图 `static/images/scenery/cover-5.jpg` ~ `cover-20.jpg`（此前 cover-1 ~ cover-4 已用于 grafana/hugo/kafka/prometheus）；
- 16 篇标记文章逐篇指向不同风景图（明细见第六节表格），键名恢复为 `featuredImage`；
- 顺带修复了过程中发现的 2 处被注释的 `#featuredImage:` 行。

## 九、后续建议

- 新增文章时按本文第一节的规则书写 Front Matter（summary、toc.auto、短 slug，教程类文章引用主题封面）。
- 封面如需替换，直接覆盖对应 `static/images/<topic>/cover.jpg` 即可，无需改动文章。
- 若旧 URL 已被搜索引擎收录，建议在 GitHub Pages/CDN 侧为高频旧链接配置 301 重定向。
