---
title: "Nginx 常用配置"
subtitle: ""
date: 2022-01-13T12:06:37+08:00
lastmod: 2022-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Nginx"]
tags: ["Nginx"]
---
## 背景
整理在使用 Nginx 过程中频繁使用的Nginx 常用配置，另外补充一些常用命令和解决方案

## Nginx 命令行常用命令
- nginx -s reload # 向主进程发送信号，重新加载配置文件，热重启
- nginx -s reopen # 重启 Nginx
- nginx -s stop # 快速关闭
- nginx -s quit # 等待工作进程处理完成后关闭
- nginx -t # 查看当前 Nginx 配置是否有错误
- nginx -t -c <配置路径> # 检查配置是否有问题，如果已经在配置目录，则不需要 - c

## 常用模块
nginx模块分为两种，官方和第三方，我们通过命令 nginx -V 查看 nginx已经安装的模块
```
[root@www ~]# nginx -V
nginx version: nginx/1.20.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
built with OpenSSL 1.1.1k  FIPS 25 Mar 2021
TLS SNI support enabled
configure arguments: --prefix=/usr/share/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/var/lib/nginx/tmp/client_body --http-proxy-temp-path=/var/lib/nginx/tmp/proxy --http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi --http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi --http-scgi-temp-path=/var/lib/nginx/tmp/scgi --pid-path=/run/nginx.pid --lock-path=/run/lock/subsys/nginx --user=nginx --group=nginx --with-compat --with-debug --with-file-aio --with-google_perftools_module --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_degradation_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_image_filter_module=dynamic --with-http_mp4_module --with-http_perl_module=dynamic --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_xslt_module=dynamic --with-mail=dynamic --with-mail_ssl_module --with-pcre --with-pcre-jit --with-stream=dynamic --with-stream_ssl_module --with-stream_ssl_preread_module --with-threads --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -m64 -mtune=generic' --with-ld-opt='-Wl,-z,relro -specs=/usr/lib/rpm/redhat/redhat-hardened-ld -Wl,-E'

```

| Nginx模块名称               | 模块作用                                                     |
| --------------------------- | ------------------------------------------------------------ |
| ngx_http_access_module      | 四层基于IP的访问控制，可以通过匹配客户端源IP地址进行限制     |
| ngx_http_auth_basic_module  | 状态页，使用basic机制进行用户认证，在编译安装nginx的时候需要添加编译参数--withhttp_stub_status_module，否则配置完成之后监测会是提示语法错误 |
| ngx_http_stub_status_module | 状态统计模块                                                 |
| ngx_http_gzip_module        | 文件的压缩功能                                               |
| ngx_http_gzip_static_module | 静态压缩模块                                                 |
| ngx_http_ssl_module         | nginx 的https 功能                                           |
| ngx_http_rewrite_module     | 重定向模块，解析和处理rewrite请求                            |
| ngx_http_referer_module     | 防盗链功能，基于访问安全考虑                                 |
| ngx_http_proxy_module       | 将客户端的请求以http协议转发至指定服务器进行处理             |
| ngx_stream_proxy_module     | tcp负载，将客户端的请求以tcp协议转发至指定服务器处理         |
| ngx_http_fastcgi_module     | 将客户端对php的请求以fastcgi协议转发至指定服务器助理         |
| ngx_http_uwsgi_module       | 将客户端对Python的请求以uwsgi协议转发至指定服务器处理        |
| ngx_http_headers_module     | 可以实现对头部报文添加指定的key与值                          |
| ngx_http_upstream_module    | 负载均衡模块，提供服务器分组转发、权重分配、状态监测、调度算法等高级功能 |
| ngx_stream_upstream_module  | 后端服务器分组转发、权重分配、状态监测、调度算法等高级功能   |
| ngx_http_fastcgi_module     | 实现通过fastcgi协议将指定的客户端请求转发至php-fpm处理       |
| ngx_http_flv_module         | 为flv伪流媒体服务端提供支持                                  |

## Nginx 核心配置
### 配置块嵌套
![nginx-configuration-inner](/images/nginx/nginx-configuration-inner.png "配置块嵌套")


### Nginx 默认配置文件
```
[root@www nginx]# cat nginx.conf|grep -v \#

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
	# 数据传输性能相关的参数;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
    	# 端口号
        listen       80;  
        listen       [::]:80;
        # server_name 指令后可以跟多个指令，第一个为主域名
        server_name  _; #不做域名匹配，只根据虚拟主机内的port去匹配
        root         /usr/share/nginx/html;
		# 模块化引入外部配置
        include /etc/nginx/default.d/*.conf;
		
		# nginx强大的基于url处理用户请求吗，就是基于location来的
        location / {
            # 定义网站根目录
            root  /usr/share/nginx/html;
            # 定义首页文件
            index   index.html;
        }
  	
        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }


}

```
- main: 全局设置
- events: 配置影响 Nginx 服务器或与用户的网络连接
- http:http 模块设置
- upstream: 负载均衡设置
- server:http 服务器配置，一个 http 模块中可以有多个 server 模块
- location:url 匹配配置，一个 server 模块中可以包含多个 location 模块

一个 nginx 配置文件的结构就像 nginx.conf 显示的那样，配置文件的语法规则：

1. 配置文件由模块组成
2. 使用#添加注释
3. 使用 $ 使用变量
4. 使用 include 引用多个配置文件
5. 每一条语句结尾必须是分号结束
6. 以区段形式的配置参数，需要有闭合的花括号 {}
7. 不同作用域的配置参数，不能瞎嵌套
    - server{}是用于定义nginx的 http核心模块功能的子配置，必须防止在http{}括号外中
    - 写在http{}外层，与其同级，语法报错
8. include配置参数：`include /etc/nginx/default.d/*.conf;`
   - 导入外部的配置文件，优化，简化主配置文件的格式
   - http{}利用include导入外部的 server{}配置
   - include得写在http{}花括号内，才表示给这个区域导入外部的配置文件
   - 只针对http{}区域生效

### Http Location 指令模块

location 是 `ngx_http_core_module` 核心模块下的指令，并且是最常用的指令，这里做一下整理记录


![nginx-location-match-uri](/images/nginx/nginx-location-match-uri.png "location 匹配规则")
指令的Location
```
Syntax:	location [ = | ~ | ~* | ^~ ] uri { ... }
location @name { ... }
Default:	—
Context:	server, location
```
![nginx-http-location-match-sx](/images/nginx/nginx-http-location-match-sx.png "location 匹配顺序")

举例
```
# = 精确匹配
location = / {
    [ configuration A ]
}

location / {
    [ configuration B ]
}

location /documents/ {
    [ configuration C ]
}

location ^~ /images/ {
    [ configuration D ]
}

location ~* \.(gif|jpg|jpeg)$ {
    [ configuration E ]
}
```
{{< admonition type=example >}}
" / "请求将匹配配置A， " /index.html "请求将匹配配置B， " /documents/document.html "请求将匹配配置C， " /images/1.gif "请求将匹配配置D， " /documents/1.jpg "请求将匹配配置E。
{{< /admonition >}}

## Nginx 场景方案
### nginx 配置动静分离
#### 什么是动静分离
在 Web 开发中，通常来说，动态资源其实就是指那些后台资源，而静态资源就是指 HTML，JavaScript，CSS，img 等文件。
在使用前后端分离之后，可以很大程度的提升静态资源的访问速度，同时在开发过程中也可以让前后端开发并行可以有效的提高开发时间，也可以有效的减少联调时间 。
#### 动静分离方案
- 直接使用不同的域名，把静态资源放在独立的云服务器上，这个种方案也是目前比较推崇的。
- 动态请求和静态文件放在一起，通过 nginx 配置分开
```
server {
  location /www/ {
      root /www/;
    index index.html index.htm;
  }

  location /image/ {
      root /image/;
  }
}
```
### nginx 配置反向代理
反向代理常用于不想把端口暴露出去，直接访问域名处理请求。
```
server {
    listen    80;
    server_name www.dazhongma.top;
    location /swoole/ {
        proxy_pass http://127.0.0.1:9501;
    }
    location /node/ {
        proxy_pass http://127.0.0.1:9502;
    }

}
```
### nginx 配置负载均衡
```
# 定义一个名为phpServer的上游服务器组，用于负载均衡
upstream phpServer{
    # 指定上游服务器组中的服务器列表和端口
    server 127.0.0.1:9501;  # 服务器1的地址和端口
    server 127.0.0.1:9502;  # 服务器2的地址和端口
    server 127.0.0.1:9503;  # 服务器3的地址和端口
}

# 定义一个服务器块，用于处理HTTP请求
server {
    # 监听80端口，即HTTP默认端口
    listen 80;
    # 设置服务器的域名
    server_name www.dazhongma.top;

    # 定义对根目录的请求的处理方式
    location / {
        # 将请求代理到上游服务器组phpServer
        proxy_pass http://phpServer;
        # 关闭代理中的重定向
        proxy_redirect off;
        # 设置代理请求中的Host头，使用原始请求的Host头
        proxy_set_header Host $host;
        # 设置代理请求中的X-Real-IP头，使用原始请求的IP地址
        proxy_set_header X-Real-IP $remote_addr;
        # 设置当代理请求失败时，如何进行下一步操作
        proxy_next_upstream error timeout invalid_header;
        # 设置代理请求时，临时文件的最大大小为0，即不使用临时文件
        proxy_max_temp_file_size 0;
        # 设置代理连接的超时时间
        proxy_connect_timeout 90;
        # 设置代理发送请求的超时时间
        proxy_send_timeout 90;
        # 设置代理读取响应的超时时间
        proxy_read_timeout 90;
        # 设置代理使用的缓冲区大小
        proxy_buffer_size 4k;
        # 设置代理使用的缓冲区数量和大小
        proxy_buffers 4 32k;
        # 设置代理忙碌时使用的缓冲区大小
        proxy_busy_buffers_size 64k;
        # 设置代理临时文件写入的最大大小
        proxy_temp_file_write_size 64k;
    }
}
```
#### 常用负载均衡策略
**round-robin / 轮询**： 到应用服务器的请求以 round-robin / 轮询的方式被分发
```
upstream phpServer{
    server 127.0.0.1:9501 weight=3;
    server 127.0.0.1:9502;
    server 127.0.0.1:9503;
}
```
在这个配置中，每 5 个新请求将会如下的在应用实例中分派： 3 个请求分派去 9501, 一个去 9502, 另外一个去 9503.

**least-connected / 最少连接**：下一个请求将被分派到活动连接数量最少的服务器
```
upstream phpServer{
    least_conn;
    server 127.0.0.1:9501;
    server 127.0.0.1:9502;
    server 127.0.0.1:9503;
}
```
当某些请求需要更长时间来完成时，最少连接可以更公平的控制应用实例上的负载。

**ip-hash/IP 散列**： 使用 hash 算法来决定下一个请求要选择哪个服务器 (基于客户端 IP 地址)
```
upstream phpServer{
    ip_hash;
    server 127.0.0.1:9501;
    server 127.0.0.1:9502;
    server 127.0.0.1:9503;
}
```
将一个客户端绑定给某个特定的应用服务器；

### nginx 配置跨域
由于浏览器同源策略的存在使得一个源中加载来自其它源中资源的行为受到了限制。即会出现跨域请求禁止。
所谓同源是指：域名、协议、端口相同。

| URL                                           | 结果 | 原因                    |
| --------------------------------------------- | ---- | ----------------------- |
| http://www.dazhongma.top/other/index.html     | 成功 | 域名、协议、端口均 相同 |
| https://www.dazhongma.top/home/index.html     | 失败 | 协议不同                |
| http://www.dazhongma.top:8080/home/index.html | 失败 | 端口不同                |
| http://www.dazhongma.com/home/index.html      | 失败 | 域名不同                |

```
server {
        listen       80;
        server_name  www.dazhongma.top;
        root   /Users/shiwenyuan/blog/public;
        index  index.html index.htm index.php;
        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }
        add_header 'Access-Control-Allow-Origin' "$http_origin";
        add_header 'Access-Control-Allow-Credentials' 'true';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, DELETE, PUT, PATCH';
        add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,X-XSRF-TOKEN';

        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
        error_page  404              /404.html;
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
}
```
- Access-Control-Allow-Origin：允许的域名，只能填 *（通配符）或者单域名。
- Access-Control-Allow-Methods: 允许的方法，多个方法以逗号分隔。
- Access-Control-Allow-Headers: 允许的头部，多个方法以逗号分隔。
- Access-Control-Allow-Credentials: 是否允许发送 Cookie。

## 日常工作中的奇淫技巧
### 日志切割脚本
```
#!/bin/bash
#设置你的日志存放的目录
log_files_path="/mnt/usr/logs/"
#日志以年/月的目录形式存放
log_files_dir=${log_files_path}"backup/"
#设置需要进行日志分割的日志文件名称，多个以空格隔开
log_files_name=(access.log error.log)
#设置nginx的安装路径
nginx_sbin="/mnt/usr/sbin/nginx -c /mnt/usr/conf/nginx.conf"
#Set how long you want to save
save_days=10

############################################
#Please do not modify the following script #
############################################
mkdir -p $log_files_dir

log_files_num=${#log_files_name[@]}
#cut nginx log files
for((i=0;i<$log_files_num;i++));do
    mv ${log_files_path}${log_files_name[i]} ${log_files_dir}${log_files_name[i]}_$(date -d "yesterday" +"%Y%m%d")
done
$nginx_sbin -s reload
```
### 图片放盗链
```
server {
  listen       80;        
  server_name  *.phpblog.com.cn;

  # 图片防盗链
  location ~* \.(gif|jpg|jpeg|png|bmp|swf)$ {
    valid_referers none blocked server_names ~\.google\. ~\.baidu\. *.qq.com;
    if ($invalid_referer){
      return 403;
    }
  }
}

```
### nginx 访问控制
```
location ~ \.php$ {
    allow 127.0.0.1;  #只允许127.0.0.1的访问，其他均拒绝
    deny all;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include        fastcgi_params;
}
```
### 丢弃不受支持的文件扩展名的请求
```
location ~ \.(js|css|sql)$ {
    deny all;
}
```


## 参考
- https://nginx.org/en/docs/http/ngx_http_core_module.html#http