---
title: "Nginx 日常工作常用脚本"
subtitle: ""
date: 2022-01-11T12:06:37+08:00
lastmod: 2022-06-27T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Nginx"]
tags: ["Nginx"]
---
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
