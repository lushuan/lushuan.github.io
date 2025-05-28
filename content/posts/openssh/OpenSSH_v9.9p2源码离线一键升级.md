---
title: "OpenSSH v9.9p2源码离线一键升级"
subtitle: ""
date: 2025-02-21T12:06:37+08:00
lastmod: 2025-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["OpenSSH"]
tags: ["OpenSSH"]
---

## 背景
- Linux操作系统环境：CentOS 7.x

{{< admonition type=info title="前言" >}}
由于安全扫描不同主机的 OpenSSH 版本不一，有些版本存在安全漏洞问题，现将 OpenSSH 统一升级至v9.9p2 版本，这里记录下整个操作的过程。

如果 OpenSSL 相对版本较低，这里建议先将OpenSS版本升级至OpenSSL 1.1.1w  11 Sep 2023，然后再升级 OpenSSH v9.9p2,如果你的主机当前的OpenSSH 版本为v9.9p1，则同样建议升级至v9.9p2，因为v9.9p1 含有两个中危漏洞，OpenSSH 安全漏洞(CVE-2025-26465 ) 和 OpenSSH 资源管理错误漏洞(CVE-2025-26466 )，此漏洞同样在 v9.9p2 版本进行了修复。
{{< /admonition >}}

## 相关安装包
{{< admonition type=quote title="地址" >}}
openssh下载地址：http://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/  
openssl下载地址：https://www.openssl.org/source/  
zlib下载地址：http://www.zlib.net/  
openssl-1.1.1w: https://www.openssl.org/source/openssl-1.1.1w.tar.gz  
{{< /admonition >}}

## 升级 OpenSSL
查找openssl相关目录，然后备份  
```shell
$ whereis openssl
#openssl: /usr/bin/openssl /usr/lib64/openssl /usr/include/openssl /usr/share/man/man1/openssl.1ssl.gz
$ cp -rp /usr/bin/openssl  /usr/bin/openssl`date '+%Y%m%d'`
$ cp -rp /usr/lib64/openssl /usr/lib64/openssl`date '+%Y%m%d'`
$ cp -rp /usr/include/openssl /usr/include/openssl`date '+%Y%m%d'`
```

### 卸载旧版本 OpenSSL(可选)
```shell
$ yum -y remove openssl
```
### 安装 OpenSSL v1.1.1w  
```
$ wget --inet4-only  https://www.openssl.org/source/openssl-1.1.1w.tar.gz
$ tar -xzvf openssl-1.1.1w.tar.gz
$ cd openssl-1.1.1w/
#目录选择/usr，因为系统最初始的openssl的目录就是/usr，这样可以省去软连接、更新链接库的问题
$ ./config --prefix=/usr
#如果不加prefix ,openssl的默认路径如下
#Bin：  /usr/local/bin/openssl
#include库  ：/usr/local/include/openssl
#lib库：/usr/local/lib64/
#engine库：/usr/lib64/openssl/engines

#编译安装,这两步相对会耗费点时间
$ make
$ make install
```

### 验证
```
$ whereis openssl
openssl: /usr/bin/openssl /usr/include/openssl /usr/local/openssl /usr/share/man/man1/openssl.1ssl.gz /usr/share/man/man1/openssl.1ossl3.gz /usr/share/man/man1/openssl.1
$ openssl version
OpenSSL 1.1.1w  11 Sep 2023
```

## 升级 OpenSSH 至 v9.9p2
### 查看当前版本
不同环境的ssh 版本会有所不同
```shell
$ ssh -V
OpenSSH_8.0p1, OpenSSL 1.1.1k  FIPS 25 Mar 2021
```

### 升级前先备份
```
#通过whereis ssh sshd找出bin文件、源文件，然后备份(man手册不需要备份)
$ whereis ssh sshd
#ssh: /usr/bin/ssh /etc/ssh /usr/share/man/man1/ssh.1.gz
#sshd: /usr/sbin/sshd /usr/share/man/man8/sshd.8.gz
#cp -rp /etc/ssh /etc/ssh`date '+%Y%m%d%H%M%S'`
$ cp -rp /etc/ssh /etc/ssh`date '+%Y%m%d'`
$ cp -p /usr/bin/ssh /usr/bin/ssh`date '+%Y%m%d'`
$ cp -p /usr/sbin/sshd /usr/sbin/sshd`date '+%Y%m%d%'`
$ cp -p /etc/ssh/sshd_config /etc/ssh/sshd_config`date '+%Y%m%d'`
#备份pam验证文件
$ cp -p /etc/pam.d/sshd /etc/pam.d/sshd`date '+%Y%m%d'`
```

### 卸载旧版OpenSSH 
```
yum -y remove openssh
```

### OpenSSH 离线升级一键脚本（v9.9p2）

`vi upgrade_openssh.sh`

```bash
#!/bin/bash
# OpenSSH 离线升级一键脚本（v9.9p2）



# 检查执行权限
if [[ "$(whoami)" != "root" ]]; then
    echo -e "\033[31m错误：必须使用 root 用户执行此脚本！\033[0m" >&2
    exit 1
fi

# 环境检查,可自行调整，并非CentOS Linux 7.x 皆可
check_environment() {
    echo -e "\n\033[34m[1/7] 正在检查系统环境...\033[0m"
    if ! grep -q "CentOS Linux release 7.9" /etc/redhat-release; then
        echo -e "\033[31m错误：仅支持 CentOS 7 操作系统！\033[0m"
       # exit 1
    fi
    
    if [ "$(uname -m)" != "x86_64" ]; then
        echo -e "\033[31m错误：仅支持 64 位系统！\033[0m"
        exit 1
    fi
    echo -e "\033[32m环境检查通过\033[0m"
}

# 安装依赖包
install_dependencies() {
    echo -e "\n\033[34m[3/7] 安装基础依赖...\033[0m"
    mkdir  /opt/yilai
    cd /opt/yilai
    tar -xvf yilai.tar.gz
    cd yilai
    rpm -ivh *.rpm --nodeps --force
    echo -e "\033[32m依赖包安装完成\033[0m"
}

# 编译安装 zlib
build_zlib() {
    echo -e "\n\033[34m[4/7] 编译安装 zlib...\033[0m"
    cd /opt
    tar -xvf zlib-1.3.1.tar.gz
    cd zlib-1.3.1
    ./configure --prefix=/usr/local/zlib
    make && make install
    echo '/usr/local/zlib/lib' >> /etc/ld.so.conf
    ldconfig -v
}

# 编译安装 OpenSSL
build_openssl() {
    echo -e "\n\033[34m[5/7] 编译安装 OpenSSL...\033[0m"
    cd /opt
    tar -xvf openssl-1.1.1o.tar.gz
    cd openssl-1.1.1o
    ./config --prefix=/usr/local/ssl -d shared
    make && make install
    echo'/usr/local/ssl/lib' >> /etc/ld.so.conf
    ldconfig -v
}

# 安装 OpenSSH
install_openssh() {
    echo -e "\n\033[34m[6/7] 升级 OpenSSH 到 v9.9p2...\033[0m"
    # 卸载旧版本
    # rpm -e --nodeps openssh-server openssh openssh-clients 2>/dev/null
    yum -y remove openssh
    
    # 编译安装
    cd /opt
    wget --inet4-only  https://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-9.9p2.tar.gz
    tar -xvf openssh-9.9p2.tar.gz
    cd openssh-9.9p2
    ./configure --prefix=/usr/local/openssh \
        --with-zlib=/usr/local/zlib \
        --with-ssl-dir=/usr/local/ssl \
        --without-openssl-header-check
    make && make install

    # 配置文件
    #检查是否有PasswordAuthentication yes，若没有则需进行添加
	cat /etc/ssh/sshd_config | grep PasswordAuthentication
    echo 'PermitRootLogin yes' >> /usr/local/openssh/etc/sshd_config
    echo 'PubkeyAuthentication yes' >> /usr/local/openssh/etc/sshd_config
    echo 'PasswordAuthentication yes' >> /usr/local/openssh/etc/sshd_config
    echo 'UsePAM yes' >> /usr/local/openssh/etc/sshd_config
   
    echo 'HostKeyAlgorithms ssh-rsa,ssh-dss ' >> /etc/ssh/sshd_config
    cp /usr/local/openssh/etc/sshd_config /etc/ssh/sshd_config
    # 还原备份的sshd_config(二选一)
	# /usr/bin/cp -fp  /etc/ssh/sshd_config`date '+%Y%m%d'` /etc/ssh/sshd_config
    
    # 替换系统命令（修复关键点）
    if [ -f /usr/sbin/sshd ]; then
        mv /usr/sbin/sshd /usr/sbin/sshd.bak
    fi
    cp -f /usr/local/openssh/sbin/sshd /usr/sbin/sshd  # 使用新编译的二进制文件

    # 修复权限
    chmod 755 /usr/sbin/sshd
    cp /usr/local/openssh/bin/ssh-keygen /usr/bin/ssh-keygen

    # 复制 ssh 命令
    cp /usr/local/openssh/bin/ssh /usr/bin/ssh
    chmod 755 /usr/bin/ssh

    # 启动脚本
    cp -p contrib/redhat/sshd.init /etc/init.d/sshd
    chmod +x /etc/init.d/sshd
    chkconfig --add sshd
    chkconfig sshd on
}

# 最终验证
final_check() {
    echo -e "\n\033[34m[7/7] 执行最终检查...\033[0m"
    systemctl daemon-reload
    systemctl restart sshd
    ssh -V 2>&1 | grep -q "OpenSSH_9.9p2"
    
    if [ $? -eq 0 ]; then
        echo -e "\033[32m升级成功！当前SSH版本：$(ssh -V 2>&1)\033[0m"
        echo -e "\033[33m警告：请通过新SSH端口连接确认无误后，再关闭Telnet服务！\033[0m"
    else
        echo -e "\033[31m错误：升级失败，请检查日志！\033[0m"
        exit 1
    fi
}

# 主执行流程,建议分步执行
main() {
    check_environment
    install_dependencies
    build_zlib
    build_openssl
    install_openssh
    final_check
}

# 执行主函数
main
```

###  执行
```
$ chmod +x upgrade_openssh.sh
$ ./upgrade_openssh.sh
```

### 验证
```
$ ssh -V  # 应显示 "OpenSSH_9.9p2"
OpenSSH_9.9p2, OpenSSL 1.1.1j  16 Feb 2021
$ systemctl status sshd
```

