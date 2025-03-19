---
title: "rsync结合inotify实现静态资源实时同步"
subtitle: ""
date: 2024-08-26T12:06:37+08:00
lastmod: 2024-08-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["linux"]
tags: ["rsync"]
---
rsync(remote synchronize)是一个远程数据同步工具，可通过 LAN/WAN 快速同步多台主机之间的文件，也可以使用 rsync 同步本地硬盘中的不同目录。rsync 是用于替代rcp的一个工具，rsync 使用所谓的 rsync算法 进行数据同步，这种算法只传送两个文件的不同部分，而不是每次都整份传送，因此速度相当快。

rsync 基于inotify开发,inotify 主要的功能是监测目标目录是否有变化。rsync 用作同步，inotify 用作监测，两者结合使用实现静态资源方案同步的实现。

Rsync有三种模式
- 本地模式(类似于cp命令)
- 远程模式(类似于scp命令)
- 守护进程(socket进程：是rsync的重要功能)
## 安装
- 测试机器节点`192.168.143.101`作为源主机 和 `192.168.143.102`作为目标主机
- rsync 结合inotify 监控源主机的指定目录实时推送到目标主机的指定目录下，实现静态资源的实时同步

安装配置(Red Hat)
```shell
$ yum install rsync -y
# 修改配置文件
$ vim /etc/rsyncd.conf
···
 [ftp]
        path = /data/rsync_test

# 启动并设置开机自启
$ systemctl enable rsyncd --now
# or
$ rsync --daemon
# 查看目标主机同步源主机指定目录下的文件
$ rsync --list-only lushuan@192.168.143.101::ftp
drwxr-xr-x             19 2025/02/10 10:20:40 .
-rw-r--r--              0 2025/02/10 10:20:40 1.txt
# 远程同步1：将本地内容同步到远程主机 rsync -av source/ username@remote_host:destination
$ rsync -avz  /data/rsync_test/  lushuan@192.168.143.102::ftp/ 
# 远程同步2: 将远程主机内容同步到本地
$ rsync -avz 192.168.143.101::ftp/ /data/rsync_test/
receiving incremental file list
./
1.txt
```

注意
- 传输的双方都必须安装 rsync。
- 目标主机需要防火墙需要对源主机开放873 端口权限
- 目标主机的同步目录权限应为755

## rsyncd 增加安全认证及免密登录
```shell
echo "lushuan:111" >> /etc/rsyncd.pwd
chmod 600 /etc/rsyncd.pwd
vim /etc/rsyncd.conf
...
auth users = lushuan
secrets file=/etc/rsyncd.pwd
 [ftp]
        path = /data/rsync_test
$ rsync --list-only lushuan@192.168.143.101::ftp
Password: (111)
drwxr-xr-x             19 2025/02/10 10:20:40 .
-rw-r--r--              0 2025/02/10 10:20:40 1.txt
# 目标主机免交互输入密码方式同步文件
$ echo "111" >> /etc/rsyncd.pwd.client
$ chmod 600 /etc/rsyncd.pwd.client
$ rsync --list-only --password-file=/etc/rsyncd.pwd.client lushuan@192.168.143.101::ftp
drwxr-xr-x             19 2025/02/10 10:20:40 .
-rw-r--r--              0 2025/02/10 10:20:40 1.txt
# 执行增量同步操作到目标主机的目标目录 /data/rsync_test
$ rsync -avz --password-file=/etc/rsyncd.pwd.client lushuan@192.168.143.101::ftp /data/rsync_test
# 目标主机操作，源主机有文件删除时执行同步操作 --delete
$ rsync -avz --delete --password-file=/etc/rsyncd.pwd.client lushuan@192.168.143.101::ftp /data/rsync_test
```

**rsync 常用选项**


| 选项          | 含义                                                         |
|---------------|--------------------------------------------------------------|
| `-a, --archive` | 归档模式，表示以递归方式传输文件，并保持所有文件属性（相当于 `-rlptgoD` 的组合）。 |
| `-r, --recursive` | 对子目录以递归模式处理，同步目录时包括子目录及其中的文件。       |
| `-v, --verbose` | 详细模式输出，显示更多的关于传输过程的信息。                    |
| `-z, --compress` | 在传输过程中压缩文件数据。                                     |
| `-P`           | 相当于 `--partial --progress`，显示传输进度并保留部分传输的文件（如果传输中断）。 |
| `-u, --update` | 只更新目标端较旧的文件或不存在的文件，跳过比源文件新的目标文件。 |
| `-l, --links`  | 保留符号链接，将源目录中的符号链接作为链接直接复制而不是它们指向的实际文件。 |
| `-p, --perms`  | 保留文件权限。                                                |
| `-t, --times`  | 保留修改时间。                                                |
| `-g, --group`  | 保留文件所属组信息。                                          |
| `-o, --owner`  | 保留文件所有者信息（仅适用于超级用户）。                       |
| `-D`           | 相当于 `--devices --specials`，保留设备文件和特殊文件。        |
| `--delete`     | 删除目标位置上存在但在源位置不存在的文件，使目标位置的内容完全与源位置一致。 |
| `--exclude=PATTERN` | 指定排除规则，不传输匹配该模式的文件或目录。              |

## rsync 推送方案
- 近时推送(目标主机定时轮询拉取源主机，会有一定的网络资源的浪费)
- 实时推送

### 实时推送源服务器和目标服务器配置
源主机`192.168.143.101`和目标主机`192.168.143.102` rsyncd 配置修改
```shell
# 1. 192.168.143.102 目标主机配置
$ echo "lushuan:111" >> /etc/rsyncd.pwd
$ chmod 600 /etc/rsyncd.pwd
$ vim /etc/rsyncd.conf
...
uid = root # 创建/data/rsync_test目录的用户拥有的权限
gid = root # 创建/data/rsync_test目录的用户拥有的权限
auth users = lushuan
secrets file=/etc/rsyncd.pwd
read only =  no
 [ftp]
        path = /data/rsync_test
# 重启服务
$ systemctl restart rsyncd

# 2.192.168.143.101 源主机配置
$ echo "111" >> /etc/rsyncd.pwd.client
$ chmod 600 /etc/rsyncd.pwd.client
$ rsync --list-only --password-file=/etc/rsyncd.pwd.client lushuan@192.168.143.102::ftp
# 重启服务
$ systemctl restart rsyncd
# 实时推送(手动)
$ rsync -avz --delete --password-file=/etc/rsyncd.pwd.client /data/rsync_test/ lushuan@192.168.143.102::ftp 
```
## inotify

### inotify 介绍
`inotify` 是一种用于 Linux 系统的文件系统事件监控机制，它允许应用程序监听文件或目录的变化。通过 `inotify`，程序可以接收到诸如文件被读取、写入、创建、删除、权限更改等操作的通知。这使得它非常适合需要实时响应文件系统变化的应用场景，比如自动同步文件、日志监控或者作为某些开发工具（如热重载功能）的基础。

以下是 `inotify` 的一些关键特性：

1. **高效性**：与之前的方案（例如 dnotify）相比，`inotify` 使用了更高效的机制来跟踪文件系统事件，减少了资源消耗。
2. **灵活性**：可以监视单个文件或整个目录树，并且可以选择感兴趣的事件类型。
3. **非递归监控**：默认情况下，对目录的监控不是递归的；如果要监控子目录，则需要为每个子目录单独设置监控，或者使用第三方工具来实现递归监控。
4. **丰富的事件类型**：支持多种类型的事件，包括但不限于访问文件（access）、修改文件（modify）、文件属性或内容的改变（attrib）、打开文件（open）、关闭文件（close）、移动（move）、创建（create）、删除（delete）等。

在实际应用中，可以通过编程接口（如 C 语言提供的 API 或者各种脚本语言的扩展库）直接使用 `inotify`，也可以利用一些基于 `inotify` 开发的工具，如 `inotify-tools`，它提供了命令行工具 `inotifywait` 和 `inotifywatch` 来简化监控任务。

由于其强大的功能和灵活性，`inotify` 已经成为现代 Linux 系统上监控文件系统活动的标准方式之一。不过需要注意的是，`inotify` 主要适用于 Linux 系统，在其他类 Unix 系统上可能不直接可用。

### 安装
推送端`192.168.143.101`安装`inotify`
```shell
# 依赖
yum install -y automake
# 安装
yum install -y inotify-tools
```
源主机监控目录
```shell
inotifywait -m -r  --timefmt '%Y-%m-%d %H:%M:%S' --format '%T %w%f %e' -e close_write,modify,delete,create,attrib,move /data/rsync_test/
```
简单的自动化脚本
```shell
#!/bin/bash

SOURCE="/data/rsync_test/"
DEST_USER="lushuan"
DEST_HOST="192.168.143.102"
DEST_DIR="ftp"
# 如果没有设置加密认证可以移除掉
PASSWORD_FILE="/etc/rsyncd.pwd.client"

inotifywait -m -r  --timefmt '%Y-%m-%d %H:%M:%S' -e close_write,modify,delete,create,attrib,move --format '%w%f' ${SOURCE} | while read NEWFILE
do
    rsync -avz --delete --password-file=${PASSWORD_FILE} ${SOURCE} ${DEST_USER}@${DEST_HOST}::${DEST_DIR}
done
```

`inotifywait` 常用参数及其含义

| 选项          | 含义                                                         |
|---------------|--------------------------------------------------------------|
| `-m, --monitor` | 持续监控而不退出，即使在首次事件发生后也继续监听。               |
| `-r, --recursive` | 递归监听所有子目录中的事件。                                  |
| `-e, --event` | 明确指定要监听的事件类型，如 `access`, `modify`, `attrib`, `move`, `create`, `delete`, `close_write` 等。可以同时指定多个事件，使用逗号分隔。 |
| `--timefmt`   | 当与 `--format` 一起使用时，设置时间输出格式，例如 `'%Y-%m-%d %H:%M:%S'`。 |
| `--format`    | 定制输出信息的格式，如 `'[%T] %w %f %#Xe'`，其中 `%T` 表示时间，`%w` 监听的目录，`%f` 发生事件的文件名，`%#Xe` 表示事件列表。 |
| `-q, --quiet` | 减少或消除非错误消息的输出，用于减少命令行输出的信息量。         |
| `-d, --daemon` | 以守护进程模式运行，通常配合 `--outfile` 使用来指定日志文件。     |
| `--outfile`   | 指定输出重定向到的文件路径，常与 `-d` 选项一起使用。             |

## 最佳实践
在部署rsyncd 时，可能会遇到权限相关的问题，或者是配置相关的问题，下面给出对应的参考，若果有其它的异常报错可以参考[rsync故障排除解答](https://blog.51cto.com/53cto/1771826),下面是配置参考，根据自己需求来修改，这里是加密认证的配置。 另外如果是在生产环境可能遇到防火墙没有开放的问题，注意放开一下 873 端口，最后就是源主机和目标主机如果是非root 用户注意查看目录的权限，推荐目录权限为766
```shell
...
uid = root
gid = root
auth users = lushuan
secrets file=/etc/rsyncd.pwd
read only =  no
 [ftp]
        path = /data/rsync_test
```


## 参考
1. [rsync官网](https://rsync.samba.org/)
2. [rsync 用法教程](https://www.ruanyifeng.com/blog/2020/08/rsync.html)
3. [rsync故障排除解答](https://blog.51cto.com/53cto/1771826)
