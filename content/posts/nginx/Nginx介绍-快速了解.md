---
title: "Nginx 介绍-快速了解"
subtitle: ""
date: 2022-01-19T12:06:37+08:00
lastmod: 2022-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Nginx"]
tags: ["Nginx"]
---
## Nginx 介绍
### Nginx 是什么？
Nginx (engine x) 是一个高性能的HTTP和反向代理web服务器。
Nginx是由伊戈尔·赛索耶夫为俄罗斯访问量第二的Rambler.ru站点（俄文：Рамблер）开发的.
第一个公开版本0.1.0发布于2004年10月4日。

### 为什么选择Nginx?
互联网公司大都选择 Nginx
1.Nginx 技术成熟，具备的功能是企业最常使用而且最需要的
2.适合当前主流架构趋势, 微服务、云架构、中间层
3.统一技术栈, 降低维护成本, 降低技术更新成本。
### Nginx 重要性
1.开源,可以从官网直接获取源代码
2.高性能,Nginx性能非常残暴,支持海量并发
3.高可靠,服务稳定,占用内存底
4.模块化,Nginx具有丰富的模块可以按需使用，并且有开发能力的技术人员还可以二次开发
5.支持热更新配置文件，一般情况下修改配置文件可以平滑生效，不用重新启动服务

### Nginx 应用场景

![image-20240721205413807](/images/nginx/nginx-usage-scenario-detail.png "Nginx 应用场景")

### Nginx 的组成

![image-20240721205840420](/images/nginx/nginx-components.png "Nginx 应用场景")

可以将Nginx 理解为一辆小汽车，nginx.conf 可比作驾驶员，控制这Nginx的的行为，但是当汽车在驾驶的过程的过程中，出现了问题，这个时候就需要一个黑匣子到底是汽车本身出现了问题，还是驾驶员驾驶不当出现了问题。Nginx 二进制可执行文件是由本身的框架和第三方模块，编译的可执行文件，这个文件可以理解为汽车本身。它有完整的系统，所有的功能都有它提供。`access.log` 可以理解为这个汽车经过任何一个地方形成的一个GPS轨迹，`error.log` 可以理解为汽车的黑匣子,当遇到不可预期的问题时，可以查看该日志文件记录信息进行问题定位。如果要对nginx 进行运维分析，可以对`access.log`进行分析。如果遇到一些未知的错误，这时就要对`error.log`进行分析了。

### 安装Nginx
这里选择通过yum 源进行安装，安装操作系统为Linux CentOS 7.9

1. CentOs 配置阿里云 yum 源
```
# 备份
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
# 下载yum 源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
# 清除yum缓存
yum clean all
# 缓存阿里云源
yum makecache
# 测试阿里云源 
yum list

```

2. 安装依赖包， EPEL 仓库中有 Nginx 的安装包
```
yum install openssl-devel pcre-devel epel-release -y
```
3. 安装nginx 服务
```
yum install nginx -y
```
4. 启动服务并配置开机自启动
```
systemctl start nginx
systemctl eable nginx
```
5. 测试
```
curl 192.168.1.20
```
浏览器访问
```
# 关闭防火墙
systemctl stop firewalld

```
![image-20240722133344836](/images/nginx/nginx-access.png "Nginx 访问首页")
#### Nginx 启动说明
1. yum 安装启动说明
```
nginx -t
systemctl start nginx
systemctl reload nginx
systemctl restart nginx
systemctl stop nginx
```
2. 编译安装启动管理方式
```
nginx -t
nginx
nginx -s reload
nginx -s stop
```

### Nginx 重要配置文件说明
####  1. 查看配置文件
```
$ rpm -ql nginx
...................................................
/etc/logrotate.d/nginx 		#nginx日志切割的配置文件
/etc/nginx/nginx.conf 		#nginx主配置文件
/etc/nginx/conf.d 			#子配置文件
/etc/nginx/conf.d/default.conf #默认展示的页面一样
/etc/nginx/mime.types 		#媒体类型 （http协议中的文件类型）
/etc/sysconfig/nginx 		#systemctl 管理 nginx的使用的文件
/usr/lib/systemd/system/nginx.service 		#systemctl 管理nginx（开 关 重启 reload) 
/usr/sbin/nginx 			#nginx命令
/usr/share/nginx/html 		#站点目录 网站的根目录
/var/log/nginx 				#nginx日志 access.log 访问日志
```
####  2. 查看已经编译的模块
```
[root@www html]# nginx -V
nginx version: nginx/1.20.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
built with OpenSSL 1.1.1k  FIPS 25 Mar 2021
TLS SNI support enabled
configure arguments: --prefix=/usr/share/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/var/lib/nginx/tmp/client_body --http-proxy-temp-path=/var/lib/nginx/tmp/proxy --http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi --http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi --http-scgi-temp-path=/var/lib/nginx/tmp/scgi --pid-path=/run/nginx.pid --lock-path=/run/lock/subsys/nginx --user=nginx --group=nginx --with-compat --with-debug --with-file-aio --with-google_perftools_module --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_degradation_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_image_filter_module=dynamic --with-http_mp4_module --with-http_perl_module=dynamic --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_xslt_module=dynamic --with-mail=dynamic --with-mail_ssl_module --with-pcre --with-pcre-jit --with-stream=dynamic --with-stream_ssl_module --with-stream_ssl_preread_module --with-threads --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -m64 -mtune=generic' --with-ld-opt='-Wl,-z,relro -specs=/usr/lib/rpm/redhat/redhat-hardened-ld -Wl,-E'
```
####  配置文件注解
1. 第一部分：配置文件主区域配置
```
user nginx; #定义运行nginx进程的用户
worker_processes auto; #Nginx运行的work进程数量(建议与CPU数量一致或 auto)
error_log /var/log/nginx/error.log warn; #nginx错误日志
pid /var/run/nginx.pid; #nginx运行pid
```
2. 第二部分：配置文件事件区域
```
events {
	worker_connections 1024; #每个 worker 进程支持的最大连接数
}
```
3. 第三部分：配置http区域
```
http {
	include /etc/nginx/mime.types; 					#Nginx支持的媒体类型库文件
	default_type application/octet-stream; 			#默认的媒体类型
	log_format main '$remote_addr - $remote_user [$time_local] "$request" '
					'$status $body_bytes_sent "$http_referer" '
					'"$http_user_agent" "$http_x_forwarded_for"';
	access_log /var/log/nginx/access.log main; 		#访问日志保存路径
	sendfile on; 				#开启高效传输模式
	
	#tcp_nopush on; 			#必须配合tcp_nopush使用，当数据包累计到一定大小后就发送
	keepalive_timeout 65; 		#连接超时时间，单位是秒
	#gzip on; 					#开启文件压缩
	include /etc/nginx/conf.d/*.conf; #包含子配置文件
}
```
4. 第四部分：子配置文件内容
```
server {
	listen 80; 				#指定监听端口
	server_name localhost; 	#指定监听的域名
	location / {
		root /usr/share/nginx/html; #定义站点的目录
		index index.html index.htm; #定义首页文件
	}
}
```
http server location 扩展了解项
- http{}层下允许有多个 Server{}层，一个 Server{}层下又允许有多个 Location
- http{} 标签主要用来解决用户的请求与响应。
- server{} 标签主要用来响应具体的某一个网站。
- location{} 标签主要用于匹配网站具体 URL 路径

### Nginx 配置语法
1. 配置文件由指令和指令块构成
2. 每条指令以；分号结尾，指令与参数间以空格符号分隔
3. 指令块以{} 大括号将多条指令组织在一起
4. include 语句允许组合多个配置文件以提升可维护性
5. 使用#符号添加注释，提高可读性
6. 使用$符号使用变量
7. 部分指令的参数支持正则表达式

Nginx 语法示例

![image-20240722125000339](/images/nginx/nginx-grammar-example.png "Nginx 语法示例")

### 配置参数
时间的单位
1. ms: milliseconds
2. s: seconds
3. m: minutes
4. h: hours
5. d: days
6. w: weeks
7. M: months,30 days
8. y: years,.365 days

空间的单位
1. bytes
2. kilobytes
3. megabytes
4. gigabytes

### http 配置的指令块
1. http
2. server
3. upstream
4. location

![image-20240722125641189](/images/nginx/nginx-http-instruction-block.png "http 配置的指令块")

### Nginx 命令行
1. 格式： nginx- s reload
2. 帮助： -h
3. 使用指定的配置文件： -c
4. 指定配置命令： -g
5. 指定运行目录： -p
6. 发送信号：-s
  - 立刻停止服务： stop
  - 优雅的停止服务：quit
  - 重载配置文件：reload
  - 重新开始记录日志文件：reopen
7. 测试配置文件是否有语法错误：-t -T
8. 打印nginx 的版本信息、编译信息等：-v -V 

## 参考
- http://nginx.org/
- https://www.nginx-cn.net/
- https://matob.web.id/random/nginx/
