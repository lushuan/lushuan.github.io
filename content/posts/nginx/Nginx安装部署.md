---
title: "Nginx 安装部署"
subtitle: ""
date: 2022-01-15T12:06:37+08:00
lastmod: 2022-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Nginx"]
tags: ["Nginx"]
---
- 方式一：yum 安装(Redhat)
- 方式二：二进制预编译安装
- 指定安装版本
- 安装操作系统版本CentOs7.x


## 方式一： yum 包管理器安装
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
# 查看安装提供的版本
$ sudo yum list nginx --showduplicates | sort -r
已加载插件：fastestmirror
已安装的软件包
可安装的软件包
 * updates: mirrors.aliyun.com
nginx.x86_64                        1:1.20.1-10.el7                        epel 
nginx.x86_64                        1:1.20.1-10.el7                        @epel
Loading mirror speeds from cached hostfile
 * extras: mirrors.aliyun.com
 * epel: pubmirror1.math.uh.edu
 * base: mirrors.aliyun.com

# 安装指定版本
yum install nginx-1.20.1 -y
# or 添加 Nginx 源
rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm

# 查看nginx 版本
$ nginx -v
nginx version: nginx/1.20.1

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

## 方式二：二进制预编译安装
在Linux系统上安装Nginx的二进制版本通常指的是从Nginx官方网站下载预编译的软件包，而不是通过包管理器（如`apt`或`yum`）进行安装。以下是针对Nginx 1.20.1版本的二进制安装步骤：

### 步骤 1: 下载 Nginx

首先，你需要访问[Nginx官方网站](https://nginx.org/en/download.html)找到1.20.1版本的下载链接。对于稳定版，你可以使用wget命令直接下载：

```bash
wget https://nginx.org/download/nginx-1.20.1.tar.gz
```

### 步骤 2: 解压下载的文件

下载完成后，解压缩文件到当前目录：

```bash
tar -zxvf nginx-1.20.1.tar.gz
```

这将创建一个名为`nginx-1.20.1`的文件夹。

### 步骤 3: 配置和编译

进入解压后的目录并准备编译Nginx。首先运行配置脚本以设置编译环境：

```bash
cd nginx-1.20.1
./configure
```

默认情况下，上述命令会使用默认配置进行编译。如果你想自定义安装路径或其他选项，可以添加相应的参数。例如，指定不同的安装前缀路径：

```bash
./configure --prefix=/data/nginx --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_addition_module --with-http_sub_module --with-http_stub_status_module --with-http_gzip_static_module --with-http_addition_module --with-pcre --with-stream
```

配置完成后，编译源代码：

```bash
make
```

### 步骤 4: 安装

编译成功后，执行以下命令安装Nginx：

```bash
sudo make install
```

这将会把Nginx安装到你之前配置时指定的位置，默认情况下是`/data/nginx`。

### 步骤 5: 设置软链接（可选）

如果你希望简化Nginx命令的调用，可以为Nginx创建一个软链接到系统的`sbin`目录中：

```bash
sudo ln -s /data/nginx/sbin/nginx /usr/local/sbin/nginx
```

这样，你就可以直接使用`nginx`命令来管理和操作Nginx服务器了。


### 步骤 6: 启动 Nginx

安装完成后，可以通过以下命令启动Nginx：

```bash
nginx
# 指定配置文件
nginx -c /data/nginx/conf/nginx.conf
```

如果你想检查Nginx配置是否正确，可以在启动前使用如下命令测试配置文件：

```bash
nginx -t
```
指定配置文件检查配置文件语法是否正常
```shell
sudo nginx -t -c /path/to/your/nginx.conf
```
重新加载Nginx配置
```shell
sudo nginx -s reload -c /path/to/your/nginx.conf
```


### 步骤7 添加到shell 配置文件中
添加到你的shell配置文件（如`.bashrc`或`.profile`）中
```shell
echo 'export PATH=$PATH:/usr/local/sbin' >> ~/.bashrc
source ~/.bashrc
```

### 步骤 8: 配置防火墙和SELinux（如果适用）

根据你的服务器安全设置，可能需要调整防火墙规则允许HTTP(S)流量，并且如果启用了SELinux，则可能还需要对其进行适当的配置以便让Nginx正常工作。

### 注意事项

- **依赖库**：确保系统已安装必要的依赖库，如PCRE、Zlib和OpenSSL等，这些对于Nginx的一些功能是必需的。如果你在`./configure`阶段遇到缺少依赖的问题，你可能需要先安装这些依赖。
- **更新与维护**：虽然可以从二进制安装开始，但后续的更新可能会比通过包管理器更加复杂。建议定期检查Nginx官网获取最新的安全补丁和版本更新。

通过以上步骤，你应该能够成功地在Linux系统上安装Nginx 1.20.1的二进制版本。



![image-20240722133344836](/images/nginx/nginx-access.png "Nginx 访问首页")
## Nginx 管理
### 1. 通过`systemctl`或者`service`管理
```
nginx -t
systemctl start nginx
systemctl reload nginx
systemctl restart nginx
systemctl stop nginx
```

### 2. 通过nginx原生命令管理
#### 设置nginx 软连接
```shell
sudo ln -s /PATH/nginx/sbin/nginx /usr/local/sbin/nginx
```
#### 查找Nginx二进制文件的位置
通常，Nginx的二进制文件位于`/usr/sbin/nginx`，但这也可能根据你的安装位置有所不同。如果不确定，可以使用`which nginx`或`whereis nginx`来查找。

```bash
which nginx
```

或者

```bash
whereis nginx
```

#### 检查Nginx配置文件语法是否正确
在启动或重新加载Nginx之前，建议先检查配置文件的语法是否有误。

```bash
$ nginx -t
```

这将测试配置文件的有效性，并告知你是否存在任何语法错误。

#### 启动Nginx
要启动Nginx，请使用以下命令：

```bash
$ nginx
```

如果没有指定配置文件路径，Nginx会默认寻找位于`/etc/nginx/nginx.conf`的主配置文件。如果你想指定一个不同的配置文件，可以使用`-c`选项：

```bash
$ nginx -c /path/to/nginx.conf
```

#### 停止Nginx
有几种方式可以停止Nginx：

1. **快速停止**：立即停止Nginx，不等待当前处理的请求完成。
   ```bash
   $ nginx -s stop
   ```

2. **优雅地停止**（推荐）：允许Nginx完成当前正在处理的所有请求后再停止。
   ```bash
   $ nginx -s quit
   ```

#### 重新加载Nginx配置
当你修改了Nginx配置文件并希望重新加载而不中断现有连接时，可以使用以下命令：

```bash
$ nginx -s reload
```

这个命令会告诉Nginx重新加载其配置文件。如果有任何语法错误，Nginx将忽略此命令并不会重新加载配置。

#### 重启Nginx
虽然没有直接的“重启”命令，但你可以组合使用上述命令实现类似的效果。例如，先发送停止信号再启动Nginx：

```bash
$ nginx -s quit && /usr/sbin/nginx
```

不过，更常用的是简单地使用`reload`命令，它可以在大多数情况下满足需求而不需要完全重启服务。

#### 注意事项

- 在执行这些操作前，确保有足够的权限。通常需要以root用户或使用`sudo`执行这些命令。
- 直接使用Nginx命令行工具进行管理时，注意不要与`systemctl`或`service`同时使用，以免造成冲突或意外的服务状态变化。
- 总是先检查配置文件的正确性（使用`nginx -t`），然后再尝试重新加载或重启服务，以避免因配置错误导致服务无法正常启动。


