---
title: "MySQL 数据库备份和还原"
subtitle: ""
date: 2020-08-17T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["MySQL"]
tags: ["MySQL"]
---
## 背景
数据是企业生产的重中之重，保护数据安全，防止数据丢失很关键

## 使用 mysqldump 进行备份
备份的方案场景，列举部分场景

1. 只备份某一个库，不要里面的表
2. 备份所有的库，和所有的表，但是不要表中的数据，只备份它的结构(创建的SQL)
3. 只要某一个库，以及这个库中的所有表，以及所有数据
4. 只要某一个库，里面的某一张表

{{< admonition title="mysqldump 相关命令" open=false >}}
mysqldump客户端可用来转储数据库或搜集数据库进行备份或将数据转移到另一个SQL服务器(不一定是一个MySQL服务器)。转储包含创建表和/或装载表的SQL语句。

如果你在服务器上进行备份，并且表均为MyISAM表，应考虑使用mysqlhotcopy，因为可以更快地进行备份和恢复。参见8.9节，“mysqlhotcopy：数据库备份程序”。

有3种方式来调用mysqldump：

shell> mysqldump [options] db_name [tables]
shell> mysqldump [options] ---database DB1 [DB2 DB3...]
shell> mysqldump [options] --all--database
如果没有指定任何表或使用了---database或--all--database选项，则转储整个数据库。

要想获得你的版本的mysqldump支持的选项，执行mysqldump ---help。

如果运行mysqldump没有--quick或--opt选项，mysqldump在转储结果前将整个结果集装入内存。如果转储大数据库可能会出现问题。该选项默认启用，但可以用--skip-opt禁用。

如果使用最新版本的mysqldump程序生成一个转储重装到很旧版本的MySQL服务器中，不应使用--opt或-e选项。

mysqldump支持下面的选项：

---help，-？
    显示帮助消息并退出。

--add-drop--database

    在每个CREATE DATABASE语句前添加DROP DATABASE语句。

--add-drop-tables

    在每个CREATE TABLE语句前添加DROP TABLE语句。

--add-locking

    用LOCK TABLES和UNLOCK TABLES语句引用每个表转储。重载转储文件时插入得更快。参见7.2.16节，“INSERT语句的速度”。

--all--database，-A

    转储所有数据库中的所有表。与使用---database选项相同，在命令行中命名所有数据库。

--allow-keywords

    允许创建关键字列名。应在每个列名前面加上表名前缀。

---comments[={0|1}]

    如果设置为 0，禁止转储文件中的其它信息，例如程序版本、服务器版本和主机。--skip—comments与---comments=0的结果相同。 默认值为1，即包括额外信息。

--compact

    产生少量输出。该选项禁用注释并启用--skip-add-drop-tables、--no-set-names、--skip-disable-keys和--skip-add-locking选项。

--compatible=name

    产生与其它数据库系统或旧的MySQL服务器更兼容的输出。值可以为ansi、mysql323、mysql40、postgresql、oracle、mssql、db2、maxdb、no_key_options、no_tables_options或者no_field_options。要使用几个值，用逗号将它们隔开。这些值与设置服务器SQL模式的相应选项有相同的含义。参见5.3.2节，“SQL服务器模式”。

    该选项不能保证同其它服务器之间的兼容性。它只启用那些目前能够使转储输出更兼容的SQL模式值。例如，--compatible=oracle 不映射Oracle类型或使用Oracle注释语法的数据类型。

--complete-insert，-c

    使用包括列名的完整的INSERT语句。

--compress，-C

    压缩在客户端和服务器之间发送的所有信息（如果二者均支持压缩）。

--create-option

    在CREATE TABLE语句中包括所有MySQL表选项。

---database，-B

    转储几个数据库。通常情况，mysqldump将命令行中的第1个名字参量看作数据库名，后面的名看作表名。使用该选项，它将所有名字参量看作数据库名。CREATE DATABASE IF NOT EXISTS db_name和USE db_name语句包含在每个新数据库前的输出中。

---debug[=debug_options]，-# [debug_options]

    写调试日志。debug_options字符串通常为'd:t:o,file_name'。

--default-character-set=charset

    使用charsetas默认字符集。参见5.10.1节，“数据和排序用字符集”。如果没有指定，mysqldump使用utf8。

--delayed-insert

    使用INSERT DELAYED语句插入行。

--delete-master-logs

    在主复制服务器上，完成转储操作后删除二进制日志。该选项自动启用--master-data。

--disable-keys，-K

    对于每个表，用/*!40000 ALTER TABLE tbl_name DISABLE KEYS */;和/*!40000 ALTER TABLE tbl_name ENABLE KEYS */;语句引用INSERT语句。这样可以更快地装载转储文件，因为在插入所有行后创建索引。该选项只适合MyISAM表。

--extended-insert，-e

    使用包括几个VALUES列表的多行INSERT语法。这样使转储文件更小，重载文件时可以加速插入。

--fields-terminated-by=...，--fields-enclosed-by=...，--fields-optionally-enclosed-by=...，--fields-escaped-by=...，--行-terminated-by=...

    这些选项结合-T选项使用，与LOAD DATA INFILE的相应子句有相同的含义。参见13.2.5节，“LOAD DATA INFILE语法”。

--first-slave，-x

    不赞成使用，现在重新命名为--lock-all-tables。

--flush-logs，-F

    开始转储前刷新MySQL服务器日志文件。该选项要求RELOAD权限。请注意如果结合--all--database(或-A)选项使用该选项，根据每个转储的数据库刷新日志。例外情况是当使用--lock-all-tables或--master-data的时候：在这种情况下，日志只刷新一次，在所有 表被锁定后刷新。如果你想要同时转储和刷新日志，应使用--flush-logs连同--lock-all-tables或--master-data。

--force，-f
    在表转储过程中，即使出现SQL错误也继续。

--host=host_name，-h host_name
    从给定主机的MySQL服务器转储数据。默认主机是localhost。

--hex-blob
    使用十六进制符号转储二进制字符串列(例如，'abc' 变为0x616263)。影响到的列有BINARY、VARBINARY、BLOB。

--lock-all-tables，-x
    所有数据库中的所有表加锁。在整体转储过程中通过全局读锁定来实现。该选项自动关闭--single-transaction和--lock-tables。

--lock-tables，-l
    开始转储前锁定所有表。用READ LOCAL锁定表以允许并行插入MyISAM表。对于事务表例如InnoDB和BDB，--single-transaction是一个更好的选项，因为它不根本需要锁定表。
    请注意当转储多个数据库时，--lock-tables分别为每个数据库锁定表。因此，该选项不能保证转储文件中的表在数据库之间的逻辑一致性。不同数据库表的转储状态可以完全不同。

--master-data[=value]
    该选项将二进制日志的位置和文件名写入到输出中。该选项要求有RELOAD权限，并且必须启用二进制日志。如果该选项值等于1，位置和文件名被写入CHANGE MASTER语句形式的转储输出，如果你使用该SQL转储主服务器以设置从服务器，从服务器从主服务器二进制日志的正确位置开始。如果选项值等于2，CHANGE MASTER语句被写成SQL注释。如果value被省略，这是默认动作。

--master-data
    选项启用--lock-all-tables，除非还指定--single-transaction(在这种情况下，只在刚开始转储时短时间获得全局读锁定。又见--single-transaction。在任何一种情况下，日志相关动作发生在转储时。该选项自动关闭--lock-tables。

--no-create-db，-n

    该选项禁用CREATE DATABASE /*!32312 IF NOT EXISTS*/ db_name语句，如果给出---database或--all--database选项，则包含到输出中。

--no-create-info，-t

    不写重新创建每个转储表的CREATE TABLE语句。

--no-data，-d

    不写表的任何行信息。如果你只想转储表的结构这很有用。

--opt

    该选项是速记；等同于指定 --add-drop-tables--add-locking --create-option --disable-keys--extended-insert --lock-tables --quick --set-charset。它可以给出很快的转储操作并产生一个可以很快装入MySQL服务器的转储文件。该选项默认开启，但可以用--skip-opt禁用。要想只禁用确信用-opt启用的选项，使用--skip形式；例如，--skip-add-drop-tables或--skip-quick。

--password[=password]，-p[password]

    连接服务器时使用的密码。如果你使用短选项形式(-p)，不能在选项和密码之间有一个空格。如果在命令行中，忽略了--password或-p选项后面的 密码值，将提示你输入一个。

--port=port_num，-P port_num

    用于连接的TCP/IP端口号。

--protocol={TCP | SOCKET | PIPE | MEMORY}

    使用的连接协议。

--quick，-q

    该选项用于转储大的表。它强制mysqldump从服务器一次一行地检索表中的行而不是检索所有行并在输出前将它缓存到内存中。

--quote-names，-Q

    用‘`’字符引用数据库、表和列名。如果服务器SQL模式包括ANSI_QUOTES选项，用‘"’字符引用名。默认启用该选项。可以用--skip-quote-names禁用，但该选项应跟在其它选项后面，例如可以启用--quote-names的--compatible。

--result-file=file，-r file

    将输出转向给定的文件。该选项应用在Windows中，因为它禁止将新行‘\n’字符转换为‘\r\n’回车、返回/新行序列。

--routines，-R

    在转储的数据库中转储存储程序(函数和程序)。使用---routines产生的输出包含CREATE PROCEDURE和CREATE FUNCTION语句以重新创建子程序。但是，这些语句不包括属性，例如子程序定义者或创建和修改时间戳。这说明当重载子程序时，对它们进行创建时定义者应设置为重载用户，时间戳等于重载时间。

    如果你需要创建的子程序使用原来的定义者和时间戳属性，不使用--routines。相反，使用一个具有mysql数据库相应权限的MySQL账户直接转储和重载mysql.proc表的内容。

    该选项在MySQL 5.1.2中添加进来。在此之前，存储程序不转储。

--set-charset

    将SET NAMES default_character_set加到输出中。该选项默认启用。要想禁用SET NAMES语句，使用--skip-set-charset。

--single-transaction

    该选项从服务器转储数据之前发出一个BEGIN SQL语句。它只适用于事务表，例如InnoDB和BDB，因为然后它将在发出BEGIN而没有阻塞任何应用程序时转储一致的数据库状态。

    当使用该选项时，应记住只有InnoDB表能以一致的状态被转储。例如，使用该选项时任何转储的MyISAM或HEAP表仍然可以更改状态。

--single-transaction选项和--lock-tables选项是互斥的，因为LOCK TABLES会使任何挂起的事务隐含提交。

    要想转储大的表，应结合--quick使用该选项。

--socket=path，-S path

    当连接localhost(为默认主机)时使用的套接字文件。

--skip--comments

    参见---comments选项的描述。

--tab=path，-T path

    产生tab分割的数据文件。对于每个转储的表，mysqldump创建一个包含创建表的CREATE TABLE语句的tbl_name.sql文件，和一个包含其数据的tbl_name.txt文件。选项值为写入文件的目录。

    默认情况，.txt数据文件的格式是在列值和每行后面的新行之间使用tab字符。可以使用--fields-xxx和--行--xxx选项明显指定格式。

    注释：该选项只适用于mysqldump与mysqld服务器在同一台机器上运行时。你必须具有FILE权限，并且服务器必须有在你指定的目录中有写文件的许可。

--tables

    覆盖---database或-B选项。选项后面的所有参量被看作表名。

--triggers

    为每个转储的表转储触发器。该选项默认启用；用--skip-triggers禁用它。

--tz-utc

    在转储文件中加入SET TIME_ZONE='+00:00'以便TIMESTAMP列可以在具有不同时区的服务器之间转储和重载。(不使用该选项，TIMESTAMP列在具有本地时区的源服务器和目的服务器之间转储和重载）。--tz-utc也可以保护由于夏令时带来的更改。--tz-utc默认启用。要想禁用它，使用--skip-tz-utc。该选项在MySQL 5.1.2中加入。

--user=user_name，-u user_name

    连接服务器时使用的MySQL用户名。

--verbose，-v

    冗长模式。打印出程序操作的详细信息。

--version，-V

    显示版本信息并退出。

--where='where-condition', -w 'where-condition'

    只转储给定的WHERE条件选择的记录。请注意如果条件包含命令解释符专用空格或字符，一定要将条件引用起来。

    例如：

    "--where=user='jimf'"
    "-wuserid>1"
    "-wuserid<1"
    --xml，-X

    将转储输出写成XML。

    还可以使用--var_name=value选项设置下面的变量：

    max_allowed_packet

    客户端/服务器之间通信的缓存区的最大大小。最大为1GB。

    net_buffer_length

    客户端/服务器之间通信的缓存区的初始大小。当创建多行插入语句时(如同使用选项--extended-insert或--opt)，mysqldump创建长度达net_buffer_length的行。如果增加该变量，还应确保在MySQL服务器中的net_buffer_length变量至少这么大。

    还可以使用--set-variable=var_name=value或-O var_name=value语法设置变量。然而，现在不赞成使用该语法。
{{< /admonition >}}

### 数据备份的各个场景
#### 场景一：备份所有数据库实例
```sql
mysqldump -u root -p --all-databases> all_databases.sql
```
#### 场景二：导出某个或某几个数据库
数据库为`myblog`
```sql
mysqldump -u root -p --databases test mysql > db_test_and_mysql.sql
```
#### 场景三：导出一张表
数据库为`myblog`，表为`wp_users`
```sql
mysqldump -u dbadmin -p myblog wp_users> blog_users.sql
```
#### 场景四：导出整个数据库结构
导出一个数据库结构  -d 没有数据 --add-drop-table 在每个create语句之前增加一个drop table
```sql
mysqldump -u dbadmin -p -d --add-drop-table myblog > blog_struc.sql
```
#### 场景五：导出数据库中的一张表结构
数据库为`myblog`，表为`wp_users`
```sql
mysqldump -u dbadmin -p -d --add-drop-table myblog  wp_users> blog_users_struc.sql
```
#### 场景六：导出数据库一个表数据,不包括表结构
数据库为`test`，表为`person`
```sql
mysqldump -u root -p -t test person > order.sql
```

### 数据还原
数据导出就有导入还原,库实例为`test`,将备份的数据进行还原
```sql
mysql -h127.0.0.1 -uroot -P3306 -p test < order.sql
```
## 通过 crontab 定时备份指定数据库脚本参考
脚本验证适用环境Centos7.x
```shell
#!/bin/bash
#set -e
# 备份并清理，将mysql 表数据保留21天

dateDir=$(date +%Y%m%d)"_all"
backUpFolder=/zpaasssd/zcm/mysql_171_grafana_dashboard_bkp
host=10.10.168.67
port=20821
username="root"
password="password"

main(){

mkdir ${backUpFolder}/${dateDir}
#rm  -rf ${backUpFolder}/$(date -d "-21 day" +%Y%m%d)"_all".tar.gz

if [[ -f ${backUpFolder}/$(date -d "-21 day" +%Y%m%d)"_all".tar.gz ]];then
  rm  -rf ${backUpFolder}/$(date -d "-21 day" +%Y%m%d)"_all".tar.gz
fi

mysqldump -h $host -P $port -u ${username} -p${password}  --databases zcm_grafana --default-character-set utf8  > ${backUpFolder}/${dateDir}/zcm_grafana.sql
mysqldump -h $host -P $port -u ${username} -p${password}  --databases zcm_grafpub --default-character-set utf8 > ${backUpFolder}/${dateDir}/zcm_grafpub.sql
#mysqldump -h $host -P $port -u ${username} -p${password}  --databases zcm_nms --default-character-set utf8 > ${backUpFolder}/${dateDir}/zcm_nms.sql
mysqldump -h $host -P $port -u ${username} -p${password}  --databases zcm_task --default-character-set utf8 > ${backUpFolder}/${dateDir}/zcm_task.sql
mysqldump -h $host -P $port -u ${username} -p${password}  --databases zcm_dialing --default-character-set utf8 > ${backUpFolder}/${dateDir}/zcm_dialing.sql
#mysqldump -h $host -P $port -u ${username} -p${password}  --databases zcm_idocs --default-character-set utf8 > ${backUpFolder}/${dateDir}/zcm_idocs.sql
mysqldump -h $host -P $port -u ${username} -p${password}  --databases nms_sla --default-character-set utf8 > ${backUpFolder}/${dateDir}/nms_sla.sql

#压缩备份文件,清理原始文件
cd ${backUpFolder}
tar zcvf ${dateDir}.tar.gz ${dateDir}
rm -rf ${backUpFolder}/${dateDir}

# 解压
#tar zxvf /zpaasssd/zcm/mysql_171_grafana_dashboard_bkp.tar.gz
echo success
}

main
```



