# Dockerfile介绍及优化

# 介绍 

Dockerfile 是一个文本文件，其内包含了一条条的 指令(Instruction)，每一条指令
构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

一个简单Dockerfile文件
```Dockerfile
# VERSION 0.0.1
FROM ubuntu
MAINTAINER James Turnbull "james@example.com"
RUN echo "deb http://archive.ubuntu.com/ubuntu precise main ↩
universe" > /etc/apt/sources.list
RUN apt-get update
RUN apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo "root:password" | chpasswd
EXPOSE 22
```
1. 可以看的出来Dockerfile 包含一系列命令并附上参数
2. 每个命令都是大写开头，后面是跟着参数

## Dockerfile 命令
|命令|解释|
|---|:---|
|FROM	| 指定基础镜像	| 
|RUN	| Dockerfile中的每个指令都会创建一个新的镜像层	| 
|CMD	| 在容器中执行的命令	| 
|EXPOSE	| 暴露端口	| 
|ENV	| 设置环境变量	| 
|COPY	| 拷贝本地文件和目录至镜像	| 
|ADD	| 功能更丰富的添加拷贝指令，COPY优先于ADD	| 
|ENTRYPOINT		| ENTRYPOINT指令并不是必须的，ENTRYPOINT是一个脚本	| 
|VOLUME		| 定义镜像中的某个目录为容器卷，会随机生成一个容器卷名	| 
|WORKDIR		| 指定工作目录	| 
|ARG		| 定义了可以通过docker build --build-arg命令传递并在Dockerfile中使用的变量。	| 


## Dockerfile 小建议
- 要使用 tag，但不要 latest。
- Debian 挺好的，不要总 Ubuntu。
- apt-get update 在前，rm -rf /var/lib/apt/lists/* 在后。
- yum install，不忘 yum clean。
- 多 RUN 要合并，来减少层数。
- 无用的软件，不要乱安装。
- COPY 放最后，缓存很开心。
- 善用 dockerignore，不浪费传输。
- 不忘 MAINTAINER，这都是我的。
- 容器只运行一个应用

小结：（FROM）选择合适的基础镜像(alpine版本最好)，(RUN)在安装更新时勿忘删除缓存和安装包，
多个命令聚合在一起减少构建时的层数，用不到的软件不要安装，(ADD和COPY)选COPY,（LABEL）添加镜像元数据，
(MAINTAINER)让我知道是谁维护我，(ENV)设置默认的环境变量，（EXPOSE）说一下暴露的映射端口

## 示例
### nginx
```Dockerfile
FROM centos 
MAINTAINER xianchao 
RUN yum install wget -y 
RUN yum install nginx -y 
# COPY index.html /usr/share/nginx/html/ 
EXPOSE 80 
ENTRYPOINT ["/usr/sbin/nginx","-g","daemon off;"] 
```

### tomcat
```Dockerfile
FROM centos 
MAINTAINER xianchao 
RUN yum install wget -y 
ADD jdk-8u45-linux-x64.rpm /usr/local/ 
ADD apache-tomcat-8.0.26.tar.gz /usr/local/ 
RUN cd /usr/local && rpm -ivh jdk-8u45-linux-x64.rpm 
RUN mv /usr/local/apache-tomcat-8.0.26 /usr/local/tomcat8 
ENTRYPOINT /usr/local/tomcat8/bin/startup.sh && tail -F 
/usr/local/tomcat8/logs/catalina.out 
EXPOSE 8080 
```


