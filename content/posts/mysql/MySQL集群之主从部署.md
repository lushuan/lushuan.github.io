---
title: "MySQL 集群之主从部署"
subtitle: ""
date: 2024-10-16T12:06:37+08:00
lastmod: 2024-10-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["MySQL"]
tags: ["MySQL"]
---

## 背景信息
为了增加数据的冗余备份，增强服务的高可用性，从传统的单服务应用升级到集群服务应用是一个合适方案，这里介绍一下MySQL 5.7 版本的部署，5.7 版本也是目前大多数企业当前在用的版本，虽然也有很多企业升级到了8.0 版本，不过从安装部署的角度来看其实是大同小异的
## 环境准备

| 操作系统版本 | cpu  | 内存 | 磁盘 | IP              | 部署方式 | MySQL版本 |主从节点 |
| ------------ | ---- | ---- | ---- | --------------- | -------- | --------- |---------  |
| centos 7.9   | 2c   | 4G   | 40G  | 192.168.143.101 | rpm      | 5.7.20    | Master|
| centos 7.9   | 2c   | 4G   | 40G  | 192.168.143.102 | rpm      | 5.7.20    | Slave|

## 安装MySQL 5.7
1、下载安装包
```
$ cd /path/to/directory
$ wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.20-1.el7.x86_64.rpm-bundle.tar
# 删除不需要的安装包
$ ls |grep -vE 'client|libs|common|server-5.7'|xargs rm -f
```
2、安装依赖
```
# mysql-community-common-5.7.20-1.el7.x86_64.rpm 依赖
$ sudo yum -y install gcc vim wget net-tools lrzsz libaio
# mysql-community-client-5.7.20-1.el7.x86_64.rpm 依赖
$ sudo yum install  ncurses-compat-libs -y
# mysql-community-server-5.7.20-1.el7.x86_64.rpm 依赖
$ sudo yum install libaio numactl-libs -y
```
3、[服务器时间校正](https://blog.dazhongma.top/linux%E6%97%B6%E9%97%B4%E6%A0%A1%E6%AD%A3/)  
4、关闭防火墙和安全(主从)
```
$ setenforce 0
$ systemctl stop firewalld
$ systemctl disable firewalld
```
4.1 (可选)如果你的仓库中没有 ncurses-compat-libs，你可能需要添加一个额外的仓库，例如 EPEL (Extra Packages for Enterprise Linux)：
```
$ sudo yum install epel-release  -y
$ sudo yum install ncurses-compat-libs  -y
```
4.2 (可选) 检验系统是否包含mariadb-lib
检验系统是否包含mariadb-lib,如果有需要卸载，否则mysql 无法安装
```shell
rpm -qa | grep mariadb
rpm -e --nodeps mariadb-libs-5.5.68-1.el7.x86_64
```
5、安装
```bash
$ sudo rpm -ivh mysql-community-common-5.7.20-1.el7.x86_64.rpm
$ sudo rpm -ivh mysql-community-libs-5.7.20-1.el7.x86_64.rpm
$ sudo rpm -ivh mysql-community-libs-compat-5.7.20-1.el7.x86_64.rpm
$ sudo rpm -ivh mysql-community-client-5.7.20-1.el7.x86_64.rpm
$ sudo rpm -ivh mysql-community-server-5.7.20-1.el7.x86_64.rpm

$ rpm -qa|grep mysql
mysql-community-common-5.7.20-1.el7.x86_64
mysql-community-libs-5.7.20-1.el7.x86_64
mysql-community-libs-compat-5.7.20-1.el7.x86_64
mysql-community-client-5.7.20-1.el7.x86_64
mysql-community-server-5.7.20-1.el7.x86_64
# 检查生成的mysql 目录和文件
$ ls  /var/lib/mysql /etc/my.cnf 
/etc/my.cnf

/var/lib/mysql:
```
6、安装过程摘取
```
# rpm -ivh mysql-community-server-5.7.20-1.el7.x86_64.rpm
warning: mysql-community-server-5.7.20-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-server-5.7.20-1.e################################# [100%]
/usr/lib/tmpfiles.d/mysql.conf:16: Line references path below legacy directory /var/run/, updating /var/run/mysqld → /run/mysqld; please update the tmpfiles.d/ drop-in file accordingly.
```
{{< admonition type=note >}}
这个警告提示说明该 RPM 包未被 GPG 签名验证。如果你信任这个包的来源，可以忽略这个警告
{{< /admonition >}}

或者一次性安装会自动进行依赖顺序安装调整：
```bas
sudo rpm -ivh *.rpm
```


### 安装包功能和作用
部署 MySQL 5.7.20 版本的 RPM 包时，需要确保按照正确的顺序安装这些包，因为它们之间存在依赖关系。以下是各个包的功能作用和推荐的安装顺序：

1. **mysql-community-common-5.7.20-1.el7.x86_64.rpm**
   - 功能：提供所有 MySQL 组件共享的文件。
   - 作用：这个包是基础，其他组件依赖它提供的文件。

2. **mysql-community-libs-5.7.20-1.el7.x86_64.rpm**
   - 功能：包含客户端库文件。
   - 作用：为使用 MySQL 客户端连接到 MySQL 数据库服务器提供必要的库文件。

3. **mysql-community-libs-compat-5.7.20-1.el7.x86_64.rpm**
   - 功能：提供兼容性库文件。
   - 作用：为旧版本的应用程序提供与旧版 MySQL 库的向后兼容性。

4. **mysql-community-client-5.7.20-1.el7.x86_64.rpm**
   - 功能：包含 MySQL 客户端命令行工具。
   - 作用：允许用户通过命令行界面与 MySQL 数据库交互。

5. **mysql-community-server-5.7.20-1.el7.x86_64.rpm**
   - 功能：提供 MySQL 数据库服务器。
   - 作用：这是核心包，负责处理 SQL 查询并管理数据库。


## 配置主服务器
MySQL 的主配置文件通常位于 `/etc/my.cnf` 或 `/etc/mysql/my.cnf`，具体取决于你的操作系统。对于某些发行版或自定义安装，配置文件可能位于其他位置，如 `/etc/my.cnf.d/` 目录下的某个文件中。你可以使用以下命令来查找配置文件的位置：

```bash
$ mysql --help | grep "Default options" -A 1
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf 
```
调整需要将数据放到`/data/mysql` 目录下，该目录为外挂磁盘，磁盘空间较大，而不是安装在默认的系统磁盘目录下

创建安装目录`/data/mysql`
```bash
mkdir -p /data/mysql/binlog
chown -R mysql:mysql /data/mysql
chomod -R 755 /data/mysql
```

修改`/etc/my.cnf`配置文件
```
[mysqld]
user = mysql
port = 3306
sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER
lower_case_table_names = 1
group_concat_max_len = 40960
max_allowed_packet = 1G

datadir = /data/mysql
socket = /tmp/mysql.sock
log_error = /data/mysql/mysqld.log

# 设置server_id,注意要唯一
server_id = 1
# 忽略的数据，指不需要同步的数据库
#binlog-ignore-db=mysql
# 指定同步的数据库
#binlog-do-db=test

###binlog###
binlog_format = row
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log_bin = /data/mysql/binlog/binlog.log
expire_logs_days = 7
master_info_repository = table
relay_log_info_repository = table

###slave###
slave_parallel_type = LOGICAL_CLOCK
slave_parallel_workers = 4
log_bin_trust_function_creators = 1
skip_slave_start = ON
relay_log_recovery = ON

###buffer_pool###
join_buffer_size = 32M
sort_buffer_size = 16M
query_cache_size = 512M
myisam_sort_buffer_size = 128M
thread_cache_size = 128
tmp_table_size = 512M
max_heap_table_size = 128M
read_rnd_buffer_size = 2M

#InnoDB#
default_storage_engine = InnoDB
innodb_file_per_table = ON
innodb_flush_log_at_trx_commit = 2
innodb_buffer_pool_size = 4G
innodb_log_file_size = 1024M
innodb_flush_method = O_DIRECT

##### MyISAM#####
key_buffer_size = 512M

### character_set#####
character_set_server = utf8
collation_server = utf8_general_ci
explicit_defaults_for_timestamp = ON

####name_resolves####
host_cache_size = 0
skip_name_resolve = ON

###connection###
max_connections = 25000
max_connect_errors = 10000

### Enable slow_query_log permanent ###
slow_query_log = 1
long_query_time = 0.5
slow_query_log_file = /data/mysql/mysql_slow.log

### MASTER_MASTER ###
auto_increment_increment = 1
auto_increment_offset = 1

[mysql]
#prompt = 'mysql[\u@\h]:[\d]:'

[client]
socket = /tmp/mysql.sock
port = 3306
```
启动mysqld服务，并设置开机自启
```shell
$ sudo systemctl enable mysqld --now
$ sudo systemctl status mysqld
```

## 配置从服务器
创建安装目录`/data/mysql`
```bash
mkdir -p /data/mysql/binlog
chown -R mysql:mysql /data/mysql
chomod -R 755 /data/mysql
```

修改`/etc/my.cnf`配置文件
```
[mysqld]
# 设置server_id,注意要唯一
server_id = 1
...
### MASTER_MASTER ###
auto_increment_increment = 1
auto_increment_offset = 2   # 可根据需要调整，确保从节点与主节点的 AUTO_INCREMENT 值不冲突
```
启动mysqld服务，并设置开机自启
```shell
$ sudo systemctl enable mysqld --now
$ sudo systemctl status mysqld
```
## 修改登录密码
1、获取临时密码
默认是在`/var/log/mysqld.log` 文件下，这里修改了安装目录为`/data/mysql`
```bash
sudo grep 'temporary password' /data/mysq/mysqld.log
# grep 'temporary password' /var/log/mysqld.log
2024-11-25T06:34:37.637723Z 1 [Note] A temporary password is generated for root@localhost: hvqki:Gp>0EI
```
2、临时密码登录
```bash
mysql -u root -p
```
3、登录后修改默认密码并进行安全配置：
首次登录后，建议立即更改 root 密码并执行 `mysql_secure_installation` 脚本来提高安全性。
登录后修改root密码
```
ALTER USER 'root'@'%' IDENTIFIED BY 'your_password';


mysql>  ALTER USER 'root'@'%' IDENTIFIED BY 'your_password';
Query OK, 0 rows affected (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

# 赋远程访问权限
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'your_password' WITH GRANT OPTION;
Query OK, 0 rows affected, 1 warning (0.02 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
mysql> select host, user, authentication_string, plugin from user;
+-----------+---------------+-------------------------------------------+-----------------------+
| host      | user          | authentication_string                     | plugin                |
+-----------+---------------+-------------------------------------------+-----------------------+
| localhost | root          | *4E02A7566A743B1042608F7ED03213DE937E7D0E | mysql_native_password |
| localhost | mysql.session | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | mysql_native_password |
| localhost | mysql.sys     | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | mysql_native_password |
| %         | root          | *4E02A7566A743B1042608F7ED03213DE937E7D0E | mysql_native_password |
+-----------+---------------+-------------------------------------------+-----------------------+
4 rows in set (0.00 sec)

```
4、mysql_secure_installation MySQL 安全设置(可选)  
mysql_secure_installation 是一个用于提高 MySQL 安装安全性的脚本。它可以帮助你执行一系列的安全设置，例如设置 root 密码、移除匿名用户、禁用远程 root 登录、移除测试数据库等。以下是执行 mysql_secure_installation 脚本的详细步骤：
```
[root@localhost mysql]<20241225 14:42:33># sudo mysql_secure_installation

Securing the MySQL server deployment.

Enter password for user root: 
```
**跟随提示完成配置**

运行脚本后，你会被问到一系列问题。下面是每个问题的大致内容和建议的回答方式：

- **Enter password for user root:** 
  输入你当前的 root 用户密码（即安装时生成的临时密码或你刚刚设置的新密码）。

- **Switch to unix_socket authentication [Y/n]**
  这个选项是关于是否切换到 Unix Socket 认证插件，默认情况下选择 `N` 不启用这个选项，这样可以继续使用基于密码的身份验证。

- **Change the root password? [Y/n]**
  如果你已经在登录时修改了 root 密码，可以选择 `n`；如果还没有修改，可以选择 `y` 并按照提示设置新密码。

- **Remove anonymous users? [Y/n]**
  移除匿名用户可以提高安全性，推荐选择 `Y`。

- **Disallow root login remotely? [Y/n]**
  禁止 root 用户从远程登录通常是个好主意，除非你需要远程管理 MySQL，推荐选择 `Y`。测试环境也可以选择`n`

- **Remove test database and access to it? [Y/n]**
  移除测试数据库及其访问权限也是提高安全性的措施之一，推荐选择 `Y`。测试环境也可以选择`n`

- **Reload privilege tables now? [Y/n]**
  重新加载权限表以使更改立即生效，推荐选择 `Y`。

**对话参考**
```
Securing the MySQL server deployment.

Enter password for user root: ********

Estimated strength of the password: 100 
Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) : Y

By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.

Remove anonymous users? (Press y|Y for Yes, any other key for No) : Y
 ... Success!

Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network.

Disallow root login remotely? (Press y|Y for Yes, any other key for No) : Y
 ... Success!

By default, MySQL comes with a database named 'test' that
anyone can access. This is also intended only for testing,
and should be removed before moving into a production
environment.

Remove test database and access to it? (Press y|Y for Yes, any other key for No) : Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : Y
 ... Success!

Cleaning up...

All done! If you've completed all of the above steps, your MySQL
installation should now be secure.

Thanks for using MySQL!
```
{{< admonition type=note >}}
安装过程有问题查看服务日志，`/data/mysql/mysqld.log`
{{< /admonition >}}
## 安装后验证
- 可以尝试通过 MySQL 客户端登录到服务器。
- 使用 `mysqladmin` 工具验证服务器是否正在运行：
```bash
$ mysqladmin -u root -p version
Enter password: 
mysqladmin  Ver 8.42 Distrib 5.7.20, for Linux on x86_64
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Server version          5.7.20
Protocol version        10
Connection              Localhost via UNIX socket
UNIX socket             /var/lib/mysql/mysql.sock
Uptime:                 14 min 46 sec

Threads: 1  Questions: 12  Slow queries: 0  Opens: 106  Flush tables: 1  Open tables: 99  Queries per second avg: 0.013
```
## 开启主从同步
### Master 节点
```shell
# 登录到 MySQL 主节点
$ mysql -u root -p

# 创建复制用户（例如：`replica_user`）
CREATE USER 'replica_user'@'192.168.143.102' IDENTIFIED BY 'your_password';

#授予复制权限
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'192.168.143.102';

#  刷新权限
FLUSH PRIVILEGES;
#  获取主节点的二进制日志位置
SHOW MASTER STATUS;

+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 | 154       | your_database_name |                |
+------------------+----------+--------------+------------------+
```
记下 `File`二进制文件（例如：`mysql-bin.000001`）和 `Position`偏移量（例如：`154`）的值。

### Slave 节点
```shell
# 登录到 MySQL 备节点
$ mysql -u root -p

# 配置主节点信息
# 使用 `CHANGE MASTER TO` 命令配置从库指向主库。你需要提供主库的 IP 地址、用户名、密码以及上面提到的 `File` 和 `Position` 值：
CHANGE MASTER TO
  MASTER_HOST='192.168.143.101',
  MASTER_USER='replica_user',
  MASTER_PASSWORD='your_password',
  MASTER_LOG_FILE='mysql-bin.000001',    -- 主节点的日志文件
  MASTER_LOG_POS=154;                    -- 主节点的日志位置

#启动复制进程 开始从库的 I/O 和 SQL 线程以开始复制过程：
START SLAVE;

# 查看复制状态 检查从库的状态，确认复制是否正常工作：
SHOW SLAVE STATUS\G;
```
关注以下字段：
- `Slave_IO_Running`: 应该是 `Yes`
- `Slave_SQL_Running`: 应该是 `Yes`
- `Last_Error`: 应为空或无错误信息

## 测试主从同步
在Master主服务器上创建数据库和表，并插入数据：
```
mysql> create database test;
mysql> use test;
mysql> create table test(age int);
mysql> insert into test values(1);
mysql> select * from test;
+------+
| age  |
+------+
|    1 |
+------+
```
在从服务器上检查数据是否同步：
```
mysql> select * from test.test;
+------+
| age  |
+------+
|    1 |
+------+
1 row in set (0.00 sec)
```
以上操作确保了主服务器的数据成功复制到从服务器，实现了主从同步。
## 卸载 MySQL
卸载或者卸载重新安装
```shell
systemctl stop mysqld
systemctl status mysqld

rpm -evh rpm包名
# 依次卸载rpm包，按照下面顺序卸载不然会报错
sudo rpm -evh mysql-community-server-5.7.20-1.el7.x86_64
sudo rpm -evh mysql-community-client-5.7.20-1.el7.x86_64
sudo rpm -evh mysql-community-libs-compat-5.7.20-1.el7.x86_64
sudo rpm -evh mysql-community-libs-5.7.20-1.el7.x86_64
sudo rpm -evh mysql-community-common-5.7.20-1.el7.x86_64
# 查询是否还存在遗漏文件
rpm -qa|grep mysql
# 删除 mysql 数据库内容

sudo rm -rf  /data/mysql
sudo rm -rf  /var/lib/mysql
sudo rm -rf /usr/local/mysql
sudo rm -rf /var/run/mysqld
sudo rm -rf /etc/my.cnf
sudo rm -f /var/log/mysqld.log
```

## 参考
1. [MySQL v5.7 rpm 安装包]( https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.29-1.el7.x86_64.rpm-bundle.tar)
3. [MySQL 官网各个版本安装包](https://downloads.mysql.com/archives/community/)








