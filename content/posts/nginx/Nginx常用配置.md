---
title: "Nginx 常用配置"
subtitle: ""
date: 2022-04-13T12:06:37+08:00
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
		
		# nginx强大的基于url处理用户请求，就是基于location来的
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
### Nginx 配置动静分离
#### 什么是动静分离
- 在 Web 开发中，通常来说，动态资源其实就是指那些后台资源，而静态资源就是指 HTML，JavaScript，CSS，img 等文件。
- 在使用前后端分离之后，可以很大程度的提升静态资源的访问速度，同时在开发过程中也可以让前后端开发并行可以有效的提高开发时间，也可以有效的减少联调时间 。
- 适合中小型网址

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
    server_name blog.dazhongma.top;
    location /swoole/ {
        proxy_pass http://127.0.0.1:9501;
    }
    location /node/ {
        proxy_pass http://127.0.0.1:9502;
    }

}
```
### Nginx 负载均衡配置
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
    server_name blog.dazhongma.top;

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

### Nginx 配置跨域
由于浏览器同源策略的存在使得一个源中加载来自其它源中资源的行为受到了限制。即会出现跨域请求禁止。
所谓同源是指：域名、协议、端口相同。

| URL                                           | 结果 | 原因                    |
| --------------------------------------------- | ---- | ----------------------- |
| http://blog.dazhongma.top/other/index.html     | 成功 | 域名、协议、端口均 相同 |
| https://blog.dazhongma.top/home/index.html     | 失败 | 协议不同                |
| http://blog.dazhongma.top:8080/home/index.html | 失败 | 端口不同                |
| http://www.dazhongma.com/home/index.html      | 失败 | 域名不同                |

```
server {
        listen       80;
        server_name  blog.dazhongma.top;
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

### URLRewrite 伪静态配置
能隐藏后端服务器的地址
- URLRewrite的使用场景
- 配置方式

```
http://192.168.143.101/index.jsp?pageNum=2
伪静态配置后，隐藏请求参数
http://192.168.143.101/2.html

rewrite <regex> <replacement> [flag];
关键字   正则       替代内容      flag标记

rewrite 参数的标签段位置：server,location,if

flag标记说明：
last # 本条规则匹配完成后，继续向下匹配新的location URI规则
break # 本条规则匹配完成即终止，不再匹配后面的任何规则
redirect # 返回302 临时重定向，浏览器地址会显示跳转后的URL地址
permant # 301 永久重定向，浏览器地址会显示跳转后的URL地址
```
配置
```
location / {
	# ([0-9]+) 入参，$1 是 ([0-9]+)的占位符
	rewrite ^/([0-9]+).html$  /index.jsp?pageNum=$1 break;
	proxy_pass http://192.168.143.104:8080;
}
```

### 配置防盗链
 防盗链本质上是存在自己服务器上的资源，只能由自己的服务器来访问，其实更多的时候网站还是希望自己的能够有更多的流量进来，而不是添加防盗链防止别人引用，只是在个别的情况，例如微信公众号文章中的图片不允许别人引用时会添加防盗链

- http协议中的referer
- nginx 防盗链配置
- 使用浏览器或curl检测
- 返回错误码
- 返回错误页面
- 整合rewrite 返回报错图片

防盗链配置
```
valid_referes none | blocked |server_names | strings ...;
```
-  none，检测referer 头域不存在的情况
- blocked,检测referer头域的值被防火墙或者代理服务器删除或伪装的情况。这种情况头域的值不以"http://" 或 "https://" 开头
- server_names,设置一个或多个URL，检测Referer头域的值是否是这些URL中的某一个

配置示例
```
location ~*/(js|img|css) {
	# 检测来源的网址，而不是ip
	valid_referes 192.168.44.101;
	if($valid_referer){
		return 403
	}
	root html;
	index index.html index.htm;
}
```
防盗链返回页面配置
```
# 防盗链返回页面配置
error_page 401 /401.html;
    location = /401.html {
}

error_page 404 /404.html;
    location = /404.html {
}

error_page 500 502 503 504 /50x.html;
    location = /50x.html {
}
```

### Nginx 高可用配置
- 高可用场景及解决方案
- 安装keepalived
- 选举方式(优先级配置)
- 示例节点`192.168.143.101`和`192.168.143.102`,vip `192.168.143.100`
安装
```
# 安装依赖
$ yum install openssl-devel
$ yum install keepalived
$ vim /etc/keepalived/keepalived.conf
```
`192.168.143.101` master 节点keepalived配置
```
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL-101
}

vrrp_instance demo_keepalived {
    state MASTER
    # 注意修改网卡信息
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.143.100
    }
}
```
`192.168.143.102` backup 节点keepalived配置
```
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL-102
}

vrrp_instance demo_keepalived {
    state BACKUP
    # 注意修改网卡信息
    interface ens32
    virtual_router_id 51
    priority 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.143.100
    }
}

```

### Nginx https 加密认证访问
#### https 证书配置
- https原理
 - CA 机构
 - 证书
 - 客户端(浏览器)
 - 服务器端
- 证书自签名
- 在线证书申请 

#### 证书自签名
一般是用在内网中，通过openssl进行创建，对于公网还是需要进行申请购买的

生产nginx 自签名证书
```
# 命名时作区分使用(个人习惯)
$ host_ip=192.168.143.101

# 1、模拟创建CA server_${host_ip}.key 私钥 ： 生成key：（生成rsa私钥，des3对称加密算法，openssl格式，2048位强度）
# You must type in 4 to 1023 characters 使用 host_ip 地址即可
$ openssl genrsa -des3 -out server_${host_ip}.key 2048

# 2、通过以下方法生成没有密码的key
$ openssl rsa -in server_${host_ip}.key -out server_${host_ip}.key

# 3、[私钥加密]通过 CA server_${host_ip}.key 私钥钥生成自签名的 CA ca.crt 证书，该证书包含CA机构的公钥：（用来签署下面的server.csr文件,一路回车
$ openssl req -new -x509 -key server_${host_ip}.key -out ca.crt -days 3650

# 4、生成证书签名csr请求 certificate sign request，包含1）公钥 2）申请者信息 3）域名, ：一路回车
$ openssl req -new -key server_${host_ip}.key -out server.csr

# 5、[公钥验签(签名)] 将csr证书签名请求发送给CA机构，CA机构中对 ca.crt 证书进行签名处理并生成crt server_${host_ip}.crt：
# 也就是说公钥 server_${host_ip}.crt 相对于 公钥 ca.crt多了签名处理的
$ openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey server_${host_ip}.key -CAcreateserial -out server_${host_ip}.crt
```
总结：
1. 生成私钥`server_${host_ip}.key`
2. 生成没有密码的私钥key(可选)
3. 自签名的 CA ca.crt 证书，该证书包含CA机构的公钥
4. 生成证书签名csr请求(CSR)
5. 公钥验签(签名)
6. 验证证书

- [图形化工具XCA](https://hohnstaedt.de/xca/)

### Nginx 开启http并重定向到https
#### 开启http
开启http很简单，直接把listen 80;加到listen 443 ssl;上面去就可以了。或者新加一个server配置，如下：
```
server {
    listen 443 ssl;
    server_name localhost;
    
    ssl_certificate /key-path/localhost.pem;
    ssl_certificate_key /key-path/localhost.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;  
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on; 

    location / {
        proxy_set_header HOST $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_pass http://127.0.0.1:8000/;
    }
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_set_header HOST $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_pass http://127.0.0.1:8000/;
    }
}

```
#### 重定向到https的两种方式
要把http重定向到https也很简单，具体可以使用两种配置来实现。

第一种方式使用return 301如下：
```
server {
    listen 80;
    server_name localhost;
    return 301 https://127.0.0.1$request_uri;
}

```
第二种方式使用rewrite如下：
```
server {
    listen 80;
    server_name localhost;
    rewrite ^(.*)$ https://$host$1 permanent;
}

```
对于return和rewrite的区别，可以阅读这篇文章：[Creating NGINX Rewrite Rules](https://blog.nginx.org/blog/creating-nginx-rewrite-rules)

### nginx 对客户端和上游服务器使用keepalive 配置详解
Nginx 中的 keepalive 是一种机制，用于在客户端和服务器之间保持连接的活跃状态，从而避免每次请求都需要重新建立 TCP 连接所带来的开销。这可以显著提高性能，尤其是在高延迟网络环境中。以下是对 Nginx 配置 keepalive 的详细介绍。

#### 1. HTTP Keepalive

对于 HTTP 协议，可以通过以下指令来配置 keepalive：

- **keepalive_timeout**：设置 keep-alive 客户端连接在服务器端保持打开状态的时间长度。例如：
  ```nginx
  keepalive_timeout 65;
  ```
  上面的例子表示如果一个连接在65秒内没有活动，那么它将被关闭。

- **keepalive_requests**：指定一个 keep-alive 连接上可以服务的最大请求数量。默认情况下这个值是100。
  ```nginx
  keepalive_requests 100;
  ```

- **keepalive_disable**：允许禁用对某些浏览器的支持。一些旧版浏览器可能无法正确处理 keep-alive 请求。
  ```nginx
  keepalive_disable msie6;
  ```
### 2. Upstream Keepalive
nginx 对上游服务器使用keepalive 配置详解

当与上游服务器（如应用服务器）通信时，也可以使用 keepalive 来优化连接。这需要通过 `upstream` 块进行配置，并且需要使用 `keepalive` 指令来指定要缓存的空闲 keepalive 连接数。

```nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    keepalive 32; # 设置保持连接的空闲连接数为32
    keepalive_requests 1000;
    keepalive_timeout 65; # 可以不配，直接使用默认
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```
这里需要注意的是，为了使 upstream keepalive 起作用，必须将 `proxy_http_version` 设置为 `1.1` 并清除 `Connection` 头部字段。

#### 注意事项

- 使用 keepalive 可能会增加服务器资源的消耗，因为它需要维护更多的连接状态。因此，合理设置 keepalive 相关参数是非常重要的。
- 对于 HTTPS 站点，除了上述配置外，还需要考虑 SSL/TLS 握手的成本。尽管 keepalive 可以重用已建立的安全连接，减少重复握手带来的延迟，但是首次连接仍需完成完整的 SSL/TLS 握手过程。

通过适当的配置，Nginx 的 keepalive 功能可以帮助提升网站或应用的响应速度和整体性能。

### Nginx 内存与文件缓冲区
Nginx 作为一款高性能的HTTP和反向代理服务器，其处理请求的能力很大程度上依赖于其对内存和文件缓冲区的有效管理。下面介绍Nginx反向代理的内存与文件缓冲区的核心流程。

#### Header 缓冲区

- **proxy_buffer_size**：这个指令用于设置读取来自后端服务器响应的第一部分（通常包括响应头）的缓冲区大小。由于响应头可能包含一些较大的Cookie或其它元数据，适当调整这个值可以确保头部信息能够被完整地缓存而不会导致额外的磁盘I/O操作。

#### Body 缓冲区

- **proxy_buffering on;**：启用或禁用缓冲功能。当开启时，Nginx会尝试先将从上游服务器接收到的数据存储到由`proxy_buffers`指定的内存缓冲区中。如果响应数据量超过了内存缓冲区容量，则超出部分会被写入由`proxy_temp_path`定义的临时文件中。若关闭(`off`)，则所有响应内容都将直接发送给客户端，而不经过任何缓冲，这可能会增加客户端等待时间并减少服务器性能。

- **proxy_buffers 32 64k;**：此指令定义了用于存储响应体的缓冲区数量及其大小。在这个例子中，它表示分配了32个每个大小为64KB的缓冲块来保存响应体数据。合理的配置可以根据预期的响应体大小及系统资源状况进行调整，以优化性能。

- **proxy_max_temp_file_size 1G;**：当响应体过大以至于超过内存缓冲区容量时，Nginx会将多余的数据写入临时文件。该参数限制了这些临时文件的最大总尺寸，默认情况下是1GB。如果达到这个限制，Nginx将停止向磁盘写入新的数据，并开始直接将剩余的响应流式传输给客户端。

- **proxy_temp_file_write_size 8k;**：控制一次写入临时文件的数据量。这里的例子表明每次写操作的大小被设定为8KB。合理设置这个值有助于平衡磁盘I/O负载，尤其是在处理大量并发请求时尤为重要。

通过上述配置，Nginx能有效地管理反向代理过程中的数据流动，既保证了快速响应又能有效利用服务器资源。需要注意的是，实际应用中应根据具体情况（如服务器硬件条件、预计的流量规模等）调整这些参数，以实现最佳性能。

**示例**
```
http {
    # 其他配置项...

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend_server; # 假设后端服务器地址

            # 设置Header缓冲区大小
            proxy_buffer_size 128k;

            # 启用响应体数据的缓冲功能
            proxy_buffering on;

            # 设置用于读取响应体的缓冲区数量和大小
            proxy_buffers 32 64k;

            # 当响应体过大时，限制写入磁盘的临时文件的最大总尺寸为1G
            proxy_max_temp_file_size 1G;

            # 控制一次写入临时文件的数据量为8KB
            proxy_temp_file_write_size 8k;
        }
    }

    # 定义上游服务器组
    upstream backend_server {
        server 192.168.1.1:8080; # 后端服务器的实际IP和端口
        server 192.168.1.2:8080; # 可以添加多个后端服务器进行负载均衡
    }
}
```

配置解释：

- `proxy_buffer_size 128k;`：增大了header部分的缓冲区大小到128KB，适用于包含较大头部信息的情况。
- `proxy_buffering on;`：启用缓冲功能。这意味着Nginx会尝试先将从后端服务器接收到的数据存储在内存中，而不是直接传递给客户端。
- `proxy_buffers 32 64k;`：设置了32块每块64KB大小的缓冲区来存储响应体内容。这样总共提供了2MB（32*64KB）的内存空间来缓存响应体数据。
- `proxy_max_temp_file_size 1G;`：当响应体太大以至于超过内存缓冲区容量时，Nginx会将超出的数据写入磁盘上的临时文件，此设置限制了这些临时文件的最大总尺寸为1GB。
- `proxy_temp_file_write_size 8k;`：控制了一次写入磁盘临时文件的数据量为8KB，有助于平衡磁盘I/O操作。

这个配置实例展示了如何使用上述参数来优化Nginx作为反向代理时的性能表现。根据实际情况，比如后端服务响应的数据量、服务器硬件资源等，你可以适当调整这些数值以达到最佳效果。

### Nginx 对客户端的限制
Nginx 提供了多种配置选项来限制客户端请求，包括对请求体大小、头部大小、超时时间等方面的控制。以下是对你提到的关键词进行补充并给出完整释义，以及一个配置示例。

#### 关键词解释

- **client_body_buffer_size**：设置用于读取客户端请求体（body）的缓冲区大小。如果请求体超过了这个缓冲区大小，数据将被写入磁盘上的临时文件。这对于处理大文件上传等场景非常重要。
  
- **client_header_buffer_size**：设置用于读取客户端请求头（header）的缓冲区大小。大多数情况下，默认值足以满足需求，但如果客户端发送非常大的请求头（如包含大量Cookie或长URL），则可能需要增加此值。
  
- **client_body_temp_path**：指定当请求体超过`client_body_buffer_size`时，Nginx将请求体数据写入磁盘的临时文件存储路径。合理设置可以避免系统默认临时目录空间不足的问题。
  
- **client_max_body_size 1000M;**：定义允许客户端请求的最大主体大小。如果请求体超过这个值，Nginx会返回413 (Request Entity Too Large)错误。设置为0可禁用检查。
  
- **client_body_timeout**：设置等待客户端发送请求体的超时时间。如果在设定的时间内没有收到完整的请求体，Nginx将关闭连接。
  
- **client_header_timeout**：设置等待客户端发送请求头的超时时间。如果在这个时间内没有收到完整的请求头，Nginx也将关闭连接。

#### 配置示例

```nginx
http {
    # 设置客户端请求体的缓冲区大小
    client_body_buffer_size 16k;

    # 设置客户端请求头的缓冲区大小
    client_header_buffer_size 2k;

    # 设置当请求体过大时，Nginx写入磁盘的临时文件存储位置
    client_body_temp_path /var/lib/nginx/body;

    # 允许客户端请求的最大主体大小为1000MB
    client_max_body_size 1000M;

    # 如果在60秒内没有读取到请求体，就报超时错误
    client_body_timeout 60s;

    # 如果在30秒内没有读取到请求头，就报超时错误
    client_header_timeout 30s;

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend_server;
        }
    }

    upstream backend_server {
        server 192.168.1.1:8080;
        server 192.168.1.2:8080;
    }
}
```

#### 补充说明

- 对于处理大文件上传的场景，除了调整`client_body_buffer_size`和`client_max_body_size`外，还需要确保服务器有足够的磁盘空间来存放临时文件，并且正确设置了`client_body_temp_path`。
- `client_body_timeout`和`client_header_timeout`可以根据实际网络状况和应用需求进行调整，以平衡性能和用户体验。
- 考虑安全性，建议根据实际情况合理设置`client_max_body_size`，防止因过大的请求导致服务器资源耗尽。

### Nginx Gzip压缩
使用示例
```
http {
	...
	# nginx 动态压缩 和 静态压缩结合使用会更好
    # 开启gzip 静态压缩
    gzip_static  on;
    # 作为反向代理时，针对上游服务器返回的头信息进行压缩
    gzip_proxied expired no-cache no-store private auth;

    # 开启gzip 动态压缩
    gzip on;
    # 启用gzip压缩的最小文件，小于设置值的文件将不会压缩
    gzip_min_length 1k;
    # gzip 压缩级别，1-9，数字越大压缩的越好，也越占用CPU时间，后面会有详细说明
    gzip_comp_level 6;
    # 进行压缩的文件类型。javascript有多种形式。其中的值可以在 mime.types 文件中找到。
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png application/vnd.ms-fontobject font/ttf font/opentype font/x-woff image/svg+xml;
    # 是否在http header中添加Vary: Accept-Encoding，建议开启，不配置也就可以
    gzip_vary on;
    # 禁用IE 1-6版本的 gzip压缩，大型网站建议不要配置，建议尽量不要在配置文件中出现正则表达式，少用正则，因为这是影响性能的一个选项
    # gzip_disable "MSIE [1-6]\.";
    # 设置压缩所需要的缓冲区大小
    gzip_buffers 32 4k;
    # 设置gzip压缩针对的HTTP协议版本
    gzip_http_version 1.0;
    ...
    server {
    	listen       80;
        listen       [::]:80;
        ...
    }
}
```


#### Gzip 动态压缩及缺点
缺点：开启gzip 压缩后，sendfile 零拷贝则会失效
#### Gzip 静态压缩
静态压缩开始不会导致sendfile 零拷贝失效

### Nginx 配置正向代理与反向代理缓存
正向代理配置参考
```
$ proxy_pass $scheme://$host$request_uri;
$ resolver 8.8.8.8;
```

反向代理缓存proxy_cache配置示例
```
http {
...
  upstream backend {
	      keepalive 1000; 	# 设置最大空闲连接数为32
        keepalive_requests 1000;
        server 192.168.143.102;
    }

proxy_cache_path /ngx_tmp levels=1:2 keys_zone=test_cache:100m inactive=1d max_size=10g;
...

    server {
          listen       80;
          listen       [::]:80;
          ....

         location / {
           proxy_pass http://backend;
           proxy_http_version 1.1;
           proxy_set_header Connection "";
		  # proxy 反向代理缓存
           add_header Nginx-Cache "$upstream_cache_status";
           proxy_cache test_cache;
           proxy_cache_valid 1h; # 到上游服务器取数据多久过期
         }
    }
...
}
```

nginx 反向代理缓存proxy_cache 配置的优点

Nginx的`proxy_cache`配置主要用于实现反向代理缓存，它有多个显著的好处，可以极大地提升Web应用的性能和可靠性：

#### 提高响应速度

- **减少后端服务器负载**：通过缓存静态内容或频繁请求的数据，可以大幅减少后端服务器处理请求的次数，从而提高响应速度。
- **快速响应用户请求**：对于已经缓存的内容，Nginx可以直接从本地磁盘提供服务，而不需要转发到后端服务器进行处理，这大大加快了响应时间。

#### 增强系统的可扩展性和稳定性

- **平滑流量高峰**：在流量高峰期，缓存可以减轻后端服务器的压力，避免因过载导致的服务中断或响应迟缓。
- **提高可用性**：即使后端服务器暂时不可用（例如由于维护或故障），Nginx也可以继续为用户提供缓存的内容，确保服务的连续性。

#### 优化资源利用

- **节省带宽**：缓存机制减少了重复数据传输的需求，有助于节省网络带宽。
- **降低数据库压力**：对于动态内容，如果部分数据可以被缓存（如查询结果），则可以减少对数据库的直接访问次数，降低数据库的压力。

#### 灵活的配置选项

- **细粒度控制**：Nginx提供了丰富的指令来控制缓存行为，比如设置缓存的有效期、如何更新缓存、哪些响应应该被缓存等。
- **支持条件缓存**：可以根据请求方法、响应头等多种条件决定是否缓存响应，使得缓存策略更加灵活和精确。

#### 实现简单且高效

- **易于部署和管理**：与应用程序级别的缓存相比，Nginx的反向代理缓存配置相对简单，容易集成到现有架构中。
- **高效的存储机制**：Nginx采用高效的文件系统存储方式，并支持内存缓存以加速频繁访问的内容。

合理地使用Nginx的`proxy_cache`不仅可以显著提升用户体验，还能增强整个系统的稳定性和效率。不过需要注意的是，缓存策略的设计应当考虑到数据的一致性和时效性要求，以免影响到用户的体验或业务逻辑的正确执行。

### Nginx 配置带宽限制
Nginx 提供了多种方式来限制带宽，以满足不同的需求场景。这通常用于防止服务器过载、保障服务质量或控制内容分发速度等。下面是一些常见的使用场景以及相应的配置示例。

#### 场景一：限制客户端下载速度

在某些情况下，你可能希望限制客户端从你的网站下载文件的最大速度，以确保所有用户都能获得良好的服务体验，或者控制资源的使用情况。

示例配置：

```nginx
server {
    listen 80;
    server_name example.com;

    location /downloads/ {
        # 使用的是令牌桶算法，限制下载速率为每秒100KB
        limit_rate 100k;
        
        # 可选：设置初始爆发速率，这里为500KB
        limit_rate_after 500k;
        
        root /var/www/downloads;
    }
}
```

在这个例子中，`limit_rate` 指令设置了每个响应的最大传输速率（这里是100KB/s），而 `limit_rate_after` 则指定了在开始限速之前允许传输的数据量（这里是500KB）。这意味着对于前500KB的数据，不会有任何速率限制，之后的速度将被限制在100KB/s。

#### 场景二：基于连接数和请求频率的流量控制

有时，除了限制带宽外，还需要对来自单个IP地址的并发连接数或请求频率进行限制，以防止恶意攻击或滥用资源。

示例配置：

```nginx
http {
    # 定义一个名为 addr 的限流区，限制每秒请求数不超过20次
    limit_req_zone $binary_remote_addr zone=addr:10m rate=20r/s;

    server {
        listen 80;
        server_name example.com;

        location / {
            # 应用前面定义的限流区，并允许突发最多5个额外请求
            limit_req zone=addr burst=5 nodelay;
            
            # 限制每个连接的带宽为50KB/s
            limit_rate 50k;

            proxy_pass http://backend;
        }
    }
}
```

这里使用了 `limit_req_zone` 和 `limit_req` 来限制每个IP地址的请求频率，同时通过 `limit_rate` 来限制带宽。这样可以有效地保护服务器免受过多请求的影响，同时也控制了数据传输速度。

#### 场景三：针对不同用户组应用不同的带宽限制

有时候需要根据用户的类型（如免费用户与付费用户）提供不同的服务级别，包括不同的下载速度。

示例配置：

```nginx
map $http_x_user_type $download_rate {
    default "50k";  # 默认下载速率为50KB/s
    premium "200k"; # 高级用户的下载速率为200KB/s
}

server {
    listen 80;
    server_name example.com;

    location /downloads/ {
        # 根据用户类型动态设置下载速率
        limit_rate $download_rate;
        
        root /var/www/downloads;
    }
}
```

在这个配置中，我们使用了 `map` 指令根据自定义的HTTP头（本例中的 `X-User-Type`）来决定使用哪个下载速率。这使得可以根据用户的类别灵活地调整服务等级。

这些只是Nginx带宽限制的一些基础应用场景和配置示例。实际部署时，你可能需要根据具体的需求和环境进行适当的调整。

### Nginx 并发数限制
Nginx 的并发限制功能主要用于控制来自单一客户端或一组客户端的连接数，以防止服务器过载或被滥用。通过使用 `limit_conn_zone` 和 `limit_conn` 指令，可以有效地管理资源并确保服务的稳定性。以下是几个常见的使用场景及相应的配置示例。

#### 场景一：限制单个IP地址的最大并发连接数

在Web服务器上，可能会遇到某些用户同时发起大量请求的情况，这可能导致服务器资源耗尽，影响其他用户的访问体验。通过限制每个IP地址的最大并发连接数，可以有效缓解这种问题。

示例配置：

```nginx
http {
    # 定义一个名为 addr 的限流区，基于客户端的 IP 地址进行限制
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    server {
        listen 80;
        server_name example.com;

        location / {
            # 允许每个IP地址最多有10个并发连接
            limit_conn addr 10;
            
            proxy_pass http://backend;
        }
    }
}
```

在这个例子中，`limit_conn_zone` 使用 `$binary_remote_addr` 变量作为键（即客户端的IP地址），并在共享内存区域 `addr` 中保存状态信息，大小为10MB。然后，在具体的 `location` 块中使用 `limit_conn` 来指定每个IP地址允许的最大并发连接数为10。

#### 场景二：根据不同的URL路径应用不同的并发限制

有时需要对不同类型的资源应用不同的并发限制策略。例如，对于静态资源和动态内容可能希望设置不同的最大并发连接数。

示例配置：

```nginx
http {
    # 定义两个限流区，一个基于IP地址，另一个基于URI
    limit_conn_zone $binary_remote_addr zone=per_ip:10m;
    limit_conn_zone $request_uri zone=per_uri:10m;

    server {
        listen 80;
        server_name example.com;

        location /static/ {
            # 对于静态资源，限制每个IP地址最多有20个并发连接
            limit_conn per_ip 20;
        }

        location /api/ {
            # 对于API请求，限制每个URI最多有5个并发连接
            limit_conn per_uri 5;
            
            proxy_pass http://backend_api;
        }
    }
}
```

此配置为 `/static/` 路径下的请求设置了基于IP地址的并发限制，而为 `/api/` 路径下的请求设置了基于URI的并发限制。

#### 场景三：结合速率限制与并发限制

在一些高流量网站中，不仅需要限制并发连接数，还需要限制请求的速率，以进一步保护后端服务免受突发流量的影响。

示例配置：

```nginx
http {
    # 定义并发限制区
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    
    # 定义速率限制区
    limit_req_zone $binary_remote_addr zone=req_limit_per_ip:10m rate=5r/s;

    server {
        listen 80;
        server_name example.com;

        location / {
            # 设置每个IP地址最多有5个并发连接
            limit_conn addr 5;
            
            # 设置每个IP地址每秒最多处理5个请求，并允许突发最多10个额外请求
            limit_req zone=req_limit_per_ip burst=10 nodelay;
            
            proxy_pass http://backend;
        }
    }
}
```

这个例子展示了如何同时应用并发限制和速率限制来保护你的Web应用。通过这种方式，既可以控制同一时间内的最大连接数，也能平滑请求到达的速率，避免瞬间的大流量冲击。

以上是关于Nginx并发限制的一些典型应用场景及其配置示例。根据实际情况调整这些参数，可以帮助你更好地管理和优化你的Web服务性能。


### Nginx 日志管理
- 应用场景
- 访问日志
- 错误日志
- 日志分隔

[ngx_http_log_module](https://nginx.org/en/docs/http/ngx_http_log_module.html)

#### 日志内存缓冲区
包日志压缩解压缩与json格式输出样例

示例
```
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    # buffer=32k 当日志大小达到32k时会写入至指定的日志文件下
    #  access_log  /var/log/nginx/access.log  main buffer=32k gzip=3 flush=1s;
    access_log  /var/log/nginx/access.log  main buffer=32k;
    
    ...
}
```
#### json格式输出
要在Nginx中配置日志格式为JSON输出，你需要定义一个自定义的日志格式，并使用`log_format`指令将其设置为JSON结构。以下是一个示例配置，展示了如何设置Nginx以JSON格式记录访问日志。

示例配置

首先，在你的Nginx配置文件（通常是`nginx.conf`或位于`sites-available`目录下的某个配置文件）中的`http`块内添加如下定义：

```nginx
http {
    # 定义JSON格式的日志
    log_format json_combined escape=json '{"time_local": "$time_local", '
                                     '"remote_addr": "$remote_addr", '
                                     '"remote_user": "$remote_user", '
                                     '"request": "$request", '
                                     '"status": $status, '
                                     '"body_bytes_sent": $body_bytes_sent, '
                                     '"request_time": $request_time, '
                                     '"http_referer": "$http_referer", '
                                     '"http_user_agent": "$http_user_agent"}';

    server {
        listen 80;
        server_name example.com;

        access_log /var/log/nginx/access.log json_combined;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

**释义**

- **log_format**：这里我们定义了一个名为`json_combined`的日志格式。使用`escape=json`确保所有特殊字符都被正确转义，避免破坏JSON结构。
- **字段说明**：
  - `$time_local`: 请求的本地时间。
  - `$remote_addr`: 客户端IP地址。
  - `$remote_user`: 远程用户（通常用于身份验证；如果没有，则为空）。
  - `$request`: 完整的原始请求行。
  - `$status`: HTTP响应状态码。
  - `$body_bytes_sent`: 发送给客户端的字节数，不包括响应头的大小。
  - `$request_time`: 请求处理时间，单位为秒，精度达到毫秒。
  - `$http_referer`: 来源页面（即用户是从哪个页面链接过来的）。
  - `$http_user_agent`: 用户代理（浏览器类型等信息）。

日志输出样例

基于上述配置，一条典型的Nginx访问日志记录将以如下JSON格式输出到`/var/log/nginx/access.log`文件中：

```json
{
  "time_local": "12/Feb/2025:17:32:00 +0000",
  "remote_addr": "192.168.1.1",
  "remote_user": "-",
  "request": "GET /index.html HTTP/1.1",
  "status": 200,
  "body_bytes_sent": 612,
  "request_time": 0.002,
  "http_referer": "http://example.com/start.html",
  "http_user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.3"
}
```

这个例子展示了一个HTTP GET请求的日志条目，其中包含了请求的各种详细信息。通过将Nginx的日志格式化为JSON，可以更方便地进行日志分析、监控以及与其他系统集成。

#### 日志切割


**日志分隔的必要性**   
- **避免单个文件过大**：长时间运行会导致日志文件占用过多磁盘空间。
- **便于归档和排查**：按时间或大小分隔日志，方便定位问题。
- **结合压缩节省空间**：压缩旧日志减少存储压力。

---

##### 使用 `logrotate` 工具（推荐）
`logrotate` 是Linux系统自带的日志管理工具，支持自动化轮转、压缩和删除旧日志。

###### **步骤：**
1. **编辑Nginx的logrotate配置文件**  
   通常路径为 `/etc/logrotate.d/nginx`，内容如下：
   ```bash
   /var/log/nginx/*.log {  # 匹配Nginx日志路径
       daily               # 按天轮转
       missingok           # 日志不存在时不报错
       rotate 14           # 保留最近14天的日志
       compress            # 压缩旧日志（gzip）
       delaycompress       # 延迟压缩前一个轮转的日志（方便排查最新旧日志）
       notifempty          # 空日志不轮转
       create 0640 www-data adm  # 创建新日志文件并设置权限、所有者
       sharedscripts       # 所有日志处理完成后执行脚本
       postrotate          # 轮转后执行的命令
           [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`  # 通知Nginx重新打开日志
       endscript
   }
   ```

2. **测试配置**  
   手动执行轮转并检查结果：
   ```bash
   logrotate -vf /etc/logrotate.d/nginx  # -v: 详细输出，-f: 强制运行
   ```

3. **验证日志分隔**  
   检查 `/var/log/nginx` 目录是否生成类似 `access.log.1.gz` 的压缩文件，并确认Nginx正常写入新日志。

---


##### **关键注意事项**
1. **权限问题**  
   - 确保新日志文件所有者（如 `www-data`）与Nginx运行用户一致。
   - `logrotate` 配置中使用 `create` 指令设置权限。

2. **信号机制**  
   - `kill -USR1` 通知Nginx重新打开日志文件，无需重启服务。
   - 也可用 `nginx -s reopen` 命令实现相同效果。

3. **轮转周期调整**  
   - 修改 `logrotate` 配置中的 `daily` 为 `weekly`、`monthly` 或 `size`（如 `size 100M`）。

4. **日志处理**  
   - 使用 `compress` 和 `delaycompress` 控制压缩行为。
   - 通过 `rotate` 设置保留的旧日志数量。

##### 配置多个目录的方式
```
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    missingok
    create 0640 www-data adm
    postrotate
        nginx -s reopen
    endscript
}

# /etc/logrotate.d/nginx
/opt/nginx/logs/*.log  /data/logs/nginx/*.log  {
    daily
    rotate 30
    missingok
    dateext
    compress
    delaycompress
    notifempty
    sharedscripts
    postrotate
        [ -f /opt/nginx/logs/nginx.pid ] && kill -USR1 `cat /opt/nginx/logs/nginx.pid`
    endscript
}
```

### Nginx upsream 重试机制
探测上游服务器健康状态
- 重试机制(被动重试机制和主动重试机制)
- 状态检查

Nginx的`upstream`模块用于定义一组服务器，这些服务器可以是提供相同服务的多个后端实例。Nginx能够根据配置将请求分配给这些上游服务器，并且在某些情况下自动进行重试，这有助于提高系统的可用性和可靠性。

#### 被动态式重试机制

当一个请求发送到上游服务器失败时（例如由于超时、连接失败或返回了错误码），Nginx可以根据配置决定是否以及如何重试该请求。这个过程通过`proxy_next_upstream`指令来控制。它允许你指定在什么条件下Nginx应该尝试转发请求到下一个可用的上游服务器。

**可用条件包括**

- `error`: 当与服务器建立连接、传递请求或读取响应头时发生错误。
- `timeout`: 超时发生时（由其他指令如`proxy_connect_timeout`, `proxy_read_timeout`等定义）。
- `invalid_header`: 服务器返回无效响应头。
- `http_500`, `http_502`, `http_503`, `http_504`: 分别对应HTTP状态码500, 502, 503和504。
- `non_idempotent`: 默认情况下，只有幂等请求（如GET, HEAD）会被重试。设置此选项可允许非幂等请求（如POST）也被重试。

示例配置

下面是一个使用`proxy_next_upstream`的例子，展示了如何配置Nginx以实现动态式重试机制：

```nginx
http {
    upstream backend {
        server backend1.example.com max_fails=5;
        server backend2.example.com;
        server backend3.example.com backup; # 备份服务器，仅当前面所有服务器都不可用时使用
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;

            # 配置重试条件
            proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
            
            # 设置尝试下一个上游服务器前等待的时间
            proxy_next_upstream_timeout 60s;
            
            # 设置每个请求最多尝试多少次上游服务器
            proxy_next_upstream_tries 3;
        }
    }
}
```

在这个示例中

- **upstream** 块定义了一个名为`backend`的组，包含三个后端服务器。最后一个标记为`backup`，这意味着只有在所有非备份服务器都无法使用时才会尝试联系它。
- 在`location`块内，`proxy_pass`指令指定了要使用的上游服务器组。
- `proxy_next_upstream`指令设置了在遇到错误、超时或收到特定HTTP状态码（500, 502, 503, 504）时尝试下一个上游服务器。
- `proxy_next_upstream_timeout`设定了尝试下一个上游服务器的最大等待时间为60秒。
- `proxy_next_upstream_tries`限制了对每个请求最多尝试3次上游服务器。

通过这种方式配置，Nginx可以在面对网络波动或其他临时性故障时更加健壮，确保用户请求尽可能得到成功处理。这对于构建高可用性的Web应用至关重要。

#### 主动重试机制
主动健康检查使用tengine模块

tengine版本(已应用到淘宝网上了)  
https://github.com/yaoweibin/nginx_upstream_check_module

nginx 商业版插件  
https://nginx.org/en/docs/http/ngx_http_upstream_hc_module.html



## 参考
- https://nginx.org/en/docs/http/ngx_http_core_module.html#http
- [Nginx Config](https://www.digitalocean.com/community/tools/nginx?global.app.lang=zhCN) 可以快速方便获得nginx的配置