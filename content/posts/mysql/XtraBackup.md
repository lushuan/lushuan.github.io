---
title: "XtraDB Backup（Percona XtraBackup）备份MySQL详细介绍"
subtitle: ""
date: 2024-12-17T12:06:37+08:00
lastmod: 2025-01-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["MySQL"]
tags: ["MySQL","XtraBackup"]
---
## 一、XtraDB Backup（Percona XtraBackup）介绍

MySQL冷备、mysqldump、MySQL热拷贝都无法实现对数据库进行增量备份。在实际生产环境中增量备份是非常实用的，如果数据大于50G或100G，存储空间足够的情况下，可以每天进行完整备份，如果每天产生的数据量较大，需要定制数据备份策略。例如每周实用完整备份，周一到周六实用增量备份。而Percona-Xtrabackup就是为了实现增量备份而出现的一款主流备份工具，xtrabakackup有2个工具，分别是xtrabakup、innobakupe。

**XtraBackup** 是由 Percona 开发的开源工具，专门用于 MySQL 及兼容数据库（如 MariaDB、Percona Server）的**在线热备份**。它支持 InnoDB/XtraDB 存储引擎的高效备份，具有以下核心特性：

1. **非阻塞备份**：备份过程中不阻塞数据库的读写操作。
2. **增量备份**：仅备份自上次备份以来更改的数据，节省时间和存储。
3. **快速恢复**：支持通过全量+增量备份链快速恢复数据。
4. **备份压缩与加密**：可选压缩（如 `--compress`）和加密功能，优化存储安全。
5. **自动实现备份检验**
6. **开源，免费**

### XtraBackup 工具介绍
xtrabackup工具文件组成:
- Xtrabackup2.2 版之前包括4个可执行文件:
- innobackupex: Perl 脚本
- xtrabackup: C/C++，编译的二进制程序
- xbcrypt: 加解密
- xbstream: 支持并发写的流文件格式
 
xtrabackup 是用来备份 InnoDB 表的，不能备份非InnoDB 表，和 MySQLServer 没有交互

innobackupex 脚本用来备份非InnoDB 表，同时会调用 xtrabackup 命令来备份 InnoDB 表，还会和 MySQLServer 发送命令进行交互，如加全局读锁(FTWRL)、获取位点(SHOW SLAVESTATUS)等。即innobackupex是在 xtrabackup 之上做了一层封装实现的

**xtrabackup的新版变化**

xtrabackup版本升级到2.4后，相比之前的2.1有了比较大的变化:innobackupex 功能全部集成到 xtrabackup 里面，只有一个 binary程序，另外为了兼容考虑，innobackupex作为xtrabackup的软链接，即xtrabackup现在支持非Innodb表备份，并且Innobackupex在下一版本中移除，建议通过xtrabackup替换innobackupex

### xtrabackup备份流程图

![xtrabackup-backup-process](/images/mysql/xtrabackup-backup-process.png "xtrabackup备份流程图")


---

## 二、XtraBackup 与 MySQL 的版本对应规则
### 1. 匹配规则
| **XtraBackup 版本** | **兼容的 MySQL 版本**       | **说明**                                                                 |
|----------------------|-----------------------------|-------------------------------------------------------------------------|
| **8.0.x**            | MySQL 8.0、Percona Server 8.0 | 支持 MySQL 8.0 的新特性（如 REDO/UNDO 日志格式变更、数据字典优化）          |
| **2.4.x**            | MySQL 5.6、5.7、Percona 5.7 | 支持 MySQL 5.6/5.7，**不兼容 MySQL 8.0**                                 |
| **2.3.x 及以下**     | MySQL 5.5 或更早版本         | 已逐渐淘汰                                                              |

#### 关键注意事项：
1. **MySQL 8.0 必须使用 XtraBackup 8.0+**，否则备份会直接报错（例如 `Unsupported redo log format`）。
2. **MySQL 5.7 建议使用 XtraBackup 2.4.x**，部分 8.0 版本可能兼容但需验证。
3. **Percona Server 需与 XtraBackup 版本对齐**（如 Percona Server 5.7 → XtraBackup 2.4）。

![xtrabackup-version-match-1](/images/mysql/xtrabackup-version-match-1.png "xtrabackup-version-match-1")
![xtrabackup-version-match-2](/images/mysql/xtrabackup-version-match-2.png "xtrabackup-version-match-2")
### 2. 版本升级与降级策略

#### 1. MySQL 升级（如 5.7 → 8.0）
- **步骤**：
  1. 升级 XtraBackup 到 8.0+。
  2. 创建新备份（旧备份可能无法直接恢复至 MySQL 8.0）。

#### 2. XtraBackup 降级
- **步骤**：
  ```bash
  # 卸载当前版本
  sudo yum remove percona-xtrabackup-80

  # 安装旧版本
  sudo yum install percona-xtrabackup-24
  ```

## 三、安装 XtraBackup 

### 1. 安装 XtraBackup

#### 1. 添加 Percona 官方仓库
避免使用默认仓库（可能版本过旧），优先通过 Percona 仓库安装：
```bash
# 安装 Percona 仓库
sudo yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm

# 启用仓库
sudo percona-release setup ps80  # 若需 MySQL 8.0 对应工具
# 或
sudo percona-release setup ps57  # 若需 MySQL 5.7 对应工具
```

#### 2. 查看可安装的 XtraBackup 版本
```bash
sudo yum list percona-xtrabackup-*

# 示例输出：
# percona-xtrabackup-80.x86_64   8.0.33-28.1.el7   percona-release-x86_64
# percona-xtrabackup-24.x86_64   2.4.28-1.el7      percona-release-x86_64
```

#### 3. 安装指定版本
根据 MySQL 版本选择对应的包：
- **MySQL 8.0**：
  ```bash
  sudo yum install percona-xtrabackup-80
  ```
- **MySQL 5.6/5.7**：
  ```bash
  sudo yum install percona-xtrabackup-24
  ```
- **离线安装指定的版本**
  ```bash
   # In RPM-based distribution (like CentOS 7), you need to:
   sudo yum install cmake gcc gcc-c++ libaio libaio-devel bison ncurses-devel \
    libgcrypt-devel libcurl-devel libev-devel python-sphinx vim-common
   # 或者直接通过 yum 安装来解决依赖的问题
   sudo yum install -y percona-xtrabackup-24-2.4.20-1.el7.x86_64.rpm
   sudo rpm -qa |grep xtrabackup
   percona-xtrabackup-24-2.4.20-1.el7.x86_64
  ```

#### 4. 验证安装版本
```bash
# xtrabackup会自动读取mysql配置
xtrabackup --version

# 输出示例：
# xtrabackup version 2.4.20 based on MySQL server 5.7.26 Linux (x86_64) (revision id: c8b4056)
```

{{< admonition type=note >}}
xtrabackup 和 mysql 数据库版本是有对应关系的
{{< /admonition >}}


## 三、使用 XtraBackup 进行增量备份及还原
### 1. 全量备份
### 2. 执行全量备份
```bash
mkdir -p /path/to/full_backup
xtrabackup --backup --target-dir=/path/to/full_backup \
           --user=<mysql_user> --password=<mysql_password>
```
- `--target-dir`：指定备份文件存储目录，目录不需要提前创建
- 备份完成后，输出提示 `completed OK!` 表示成功。
- 全量备份可以一周或者一天执行一次

查看备份信息
```shell
cat /path/to/full_backup/xtrabackup_checkpoints
cat /path/to/full_backup/xtrabackup_info
cat /path/to/full_backup/backup-my.cnf
```


#### 2. 备份文件结构
- 全量备份：包含所有数据文件。
- 增量备份：仅含变化的数据页（文件后缀 `.delta` 和 `.meta`）。

---

### 2. 还原全量备份
- 还原前需要停掉MySQL 服务
- 确保MySQL 数据目录是清空状态
#### 1. 预整理
预整理，目的是将未完成的事务进行回滚
```bash
xtrabackup --prepare  --target-dir=/path/to/full_backup
```

{{< admonition type=note >}}
适合单纯地进行全量备份的恢复，并且没有打算在其上应用任何增量备份
{{< /admonition >}}


#### 2. 最终准备并恢复数据
```bash
xtrabackup --prepare --target-dir=/path/to/full_backup
```
停止 MySQL 服务，清空数据目录，将备份文件复制回数据目录：

#### 3.复制到数据库目录
例如数据库主机宕机或者挂掉了，将备份的数据拷贝到新的主机或者修复的数据库主机目录下
```bash
xtrabackup --copy-back --target-dir=/path/to/full_backup
```

#### 4.还原属性
修改所属者和所属组
```shell
chown -R mysql:mysql /var/lib/mysql
```
- `/var/lib/mysql` 是默认目录，可通过`/etc/my.cnf`来进行查看，如有修改动态调整

#### 5.启动服务
```shell
systemctl start mysqld
```

---
## MySQL 增量备份
### MySQL 增量备份（基于 xtrabackup）
#### 1. 基于 LSN 的增量备份
XtraBackup 通过 LSN（Log Sequence Number）追踪数据页变化，增量备份需基于上一次全量或增量备份：

```bash
# 第一次修改后增量备份
xtrabackup --backup --target-dir=/path/to/inc_backup_1 \
           --incremental-basedir=/path/to/full_backup \
           --user=<mysql_user> --password=<mysql_password>
# 第二次修改后增量备份   
xtrabackup --backup --target-dir=/path/to/inc_backup_2 \
           --incremental-basedir=/path/to/inc_backup_1 \
           --user=<mysql_user> --password=<mysql_password>
          
```
- `--incremental-basedir`：指向上次备份的目录（全量或增量）。
- 备份生成三个备份目录`/path/to/{full_backup,inc_backup_1,inc_backup_2}`

### MySQL 增量备份（基于 xtrabackup）还原
- 还原前需要停掉MySQL 服务
- 确保MySQL 数据目录是清空状态
#### 1. 准备全量备份
预准备：确保数据一致，阻止回滚未完成的事务
```bash
xtrabackup --prepare --apply-log-only --target-dir=/path/to/full_backup
```
- --apply-log-only阻止回滚未完成的事务，它仅执行redo日志的应用而不执行undo操作。这意味着事务不会被回滚。

**适用场景**：当计划应用增量备份（Incremental Backup）到全量备份时，需要先使用这个命令来准备全量备份。这样做是为了确保后续的增量备份可以正确地应用到基础备份上。 也就是说你的恢复策略包括先做一次全量备份，然后在此基础上应用一个或多个增量备份，则首先需要使用第一个命令（带--apply-log-only选项）准备全量备份，之后按照顺序逐一准备并应用每个增量备份（同样使用--apply-log-only直到最后一个增量备份）。最后，使用（不带--apply-log-only选项）来完成准备过程，使得整个备份集（包括全量和所有增量部分）可以用于恢复。

#### 2. 还原合并增量备份
合并增量备份到全量备份：
```bash
# 合并第一次增量备份到完全备份
xtrabackup --prepare --apply-log-only --target-dir=/path/to/full_backup \
           --incremental-dir=/path/to/inc_backup_1
# 合并第二次增量备份到完全备份:最后一次还原不需要加--apply-log-only
xtrabackup --prepare  --target-dir=/path/to/full_backup \
           --incremental-dir=/path/to/inc_backup_2
```

#### 3.复制到数据库目录
例如数据库主机宕机或者挂掉了，将备份的数据拷贝到新的主机或者修复的数据库主机目录下
```bash
xtrabackup --copy-back --target-dir=/path/to/full_backup
```
- 这里也可以使用 cp 命令进行复制

{{< admonition type=note >}}
数据库目录必须为空，且MySQL服务不能启动
{{< /admonition >}}


#### 4.还原属性
修改所属者和所属组
```shell
chown -R mysql:mysql /var/lib/mysql
```
- `/var/lib/mysql` 是默认目录，如有修改动态调整

#### 5.启动服务
```shell
systemctl start mysqld
```


### MySQL 增量备份（基于 Binlog）

### 1. 启用 Binlog
在 `my.cnf` 中配置：
```ini
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
server_id = 1
```

### 2. 定期备份 Binlog
```bash
# 刷新日志，生成新 Binlog 文件
mysqladmin flush-logs

# 备份旧的 Binlog 文件（如 mysql-bin.000001）
cp /var/lib/mysql/mysql-bin.000001 /backup/
```

### 3. 使用 Binlog 恢复数据
```bash
# 恢复到指定时间点
mysqlbinlog /backup/mysql-bin.000001 | mysql -u root -p
```

---

## 五、方案对比与选择

| **备份方式**       | **XtraBackup 增量备份**      | **Binlog 增量备份**         |
|--------------------|------------------------------|---------------------------|
| **粒度**           | 页面级（物理备份）            | SQL 语句级（逻辑备份）      |
| **恢复速度**       | 快（直接文件替换）            | 慢（需重放 SQL）           |
| **适用场景**       | 大型数据库快速恢复             | 精细恢复（如误删单表）      |
| **存储占用**       | 中等（仅变化页）              | 较大（记录所有变更）        |

---
## 六、 XtraBackup 2.4 常用参数
以下是 **Percona XtraBackup 2.4** 的常用参数解释表格，涵盖备份、恢复、压缩、加密等核心功能：

---

### XtraBackup 2.4 核心参数表

| **参数**                 | **类型** | **说明**                                                                 | **默认值**         | **示例**                                                                 |
|--------------------------|----------|-------------------------------------------------------------------------|--------------------|--------------------------------------------------------------------------|
| **通用参数**             |          |                                                                         |                    |                                                                          |
| `--backup`               | 命令     | 执行全量备份。                                                          | -                  | `xtrabackup --backup --target-dir=/backup`                               |
| `--prepare`              | 命令     | 准备备份文件（应用日志，使其可恢复）。                                  | -                  | `xtrabackup --prepare --target-dir=/backup`                              |
| `--copy-back`            | 命令     | 将备份文件复制到 MySQL 数据目录。                                       | -                  | `xtrabackup --copy-back --target-dir=/backup`                            |
| `--target-dir`           | 选项     | 指定备份文件的存储目录。                                                | -                  | `--target-dir=/backup/full`                                              |
| `--datadir`              | 选项     | 指定 MySQL 数据目录路径。                                               | `/var/lib/mysql`   | `--datadir=/data/mysql`                                                  |
| **增量备份**             |          |                                                                         |                    |                                                                          |
| `--incremental`          | 选项     | 执行增量备份。                                                          | `false`            | `--incremental --target-dir=/backup/inc1`                                |
| `--incremental-basedir`  | 选项     | 指定增量备份的基础目录（上一次全量或增量备份的路径）。                  | -                  | `--incremental-basedir=/backup/full`                                     |
| `--incremental-lsn`      | 选项     | 手动指定增量备份的起始 LSN（替代 `--incremental-basedir`）。            | -                  | `--incremental-lsn=12345678`                                             |
| **压缩与解压**           |          |                                                                         |                    |                                                                          |
| `--compress`             | 选项     | 启用备份文件压缩（使用 QuickLZ 算法）。                                 | `false`            | `--compress`                                                             |
| `--compress-threads`     | 选项     | 指定压缩线程数（并行压缩）。                                            | `1`                | `--compress --compress-threads=4`                                        |
| `--decompress`           | 选项     | 解压已压缩的备份文件（需在 `--prepare` 阶段使用）。                     | `false`            | `xtrabackup --prepare --decompress --target-dir=/backup`                 |
| **加密**                 |          |                                                                         |                    |                                                                          |
| `--encrypt`              | 选项     | 启用备份文件加密（支持 AES128/AES192/AES256）。                         | `false`            | `--encrypt=AES256`                                                       |
| `--encrypt-key`          | 选项     | 直接指定加密密钥（不推荐，存在安全风险）。                              | -                  | `--encrypt-key="MySecretKey123"`                                         |
| `--encrypt-key-file`     | 选项     | 从文件读取加密密钥（推荐方式）。                                        | -                  | `--encrypt-key-file=/path/to/keyfile`                                    |
| `--encrypt-threads`      | 选项     | 指定加密线程数（并行加密）。                                            | `1`                | `--encrypt=AES256 --encrypt-threads=4`                                   |
| **并行处理**             |          |                                                                         |                    |                                                                          |
| `--parallel`             | 选项     | 指定备份或恢复时的并行线程数（加速大文件操作）。                        | `1`                | `--parallel=4`                                                           |
| `--use-memory`           | 选项     | 指定 `--prepare` 阶段可用的内存大小（提高处理速度）。                   | `100MB`            | `--use-memory=2G`                                                        |
| **日志与监控**           |          |                                                                         |                    |                                                                          |
| `--log`                  | 选项     | 指定日志文件路径。                                                      | 输出到标准错误流   | `--log=/var/log/xtrabackup.log`                                          |
| `--stream`               | 选项     | 将备份流式传输到标准输出（通常结合压缩或加密使用）。                    | `false`            | `--stream=xbstream \| gzip > backup.xbstream.gz`                         |
| `--stats`                | 选项     | 输出备份统计信息（需配合 `--backup` 使用）。                            | `false`            | `--stats`                                                                |
| **高级选项**             |          |                                                                         |                    |                                                                          |
| `--slave-info`           | 选项     | 备份从库时记录复制信息（用于构建新从库）。                              | `false`            | `--slave-info`                                                           |
| `--safe-slave-backup`    | 选项     | 停止复制线程以确保备份一致性（从库备份时使用）。                        | `false`            | `--safe-slave-backup`                                                    |
| `--throttle`             | 选项     | 限制备份时的 I/O 操作速率（单位：IOPS）。                               | `0`（无限制）      | `--throttle=100`                                                         |
| `--version-check`        | 选项     | 检查 Percona Server 或 MySQL 版本兼容性。                               | `true`             | `--version-check=false`                                                  |

---

### 关键参数使用示例

#### 1. 全量备份 + 压缩 + 加密
```bash
xtrabackup --backup \
  --target-dir=/backup/full \
  --compress \
  --compress-threads=4 \
  --encrypt=AES256 \
  --encrypt-key-file=/etc/mysql/xtrabackup.key \
  --encrypt-threads=4
```

#### 2. 增量备份
```bash
# 第一次增量（基于全量）
xtrabackup --backup \
  --target-dir=/backup/inc1 \
  --incremental-basedir=/backup/full

# 第二次增量（基于前一次增量）
xtrabackup --backup \
  --target-dir=/backup/inc2 \
  --incremental-basedir=/backup/inc1
```

#### 3. 准备备份并解压
```bash
# 解压并应用日志
xtrabackup --prepare \
  --target-dir=/backup/full \
  --decompress
```

#### 补充 安装 qpress 解压缩
qpress 便携式高速文件归档器

当全量备份进行压缩后，在还原时解压需要解压时需要`qpress`命令
```shell
$ wget -d --user-agent="Mozilla/5.0 (Windows NT x.y; rv:10.0) Gecko/20100101 Firefox/10.0" https://docs-tencentdb-1256569818.cos.ap-guangzhou.myqcloud.com/qpress-11-linux-x64.tar

$ tar -xf qpress-11-linux-x64.tar -C /usr/local/bin
$ source /etc/profile
```

---

### 注意事项
1. **加密密钥管理**：  
   - 优先使用 `--encrypt-key-file` 替代 `--encrypt-key`，避免密钥泄露。  
   - 密钥文件需严格限制权限（如 `chmod 600`）。

2. **并行与资源占用**：  
   - 高并发（`--parallel`、`--compress-threads`）可能显著增加 CPU 和内存消耗。  
   - 生产环境中建议根据硬件资源调整。

3. **版本兼容性**：  
   - XtraBackup 2.4 支持 MySQL 5.1~5.7 和 Percona Server 对应版本。  
   - 确保备份工具与数据库版本匹配。

4. **增量备份依赖链**：  
   - 增量备份必须基于完整的全量备份链，恢复时需按顺序合并所有增量。  
   - 定期清理过时的备份链以节省存储空间。

---

通过合理组合这些参数，可以实现高效、安全的数据库备份与恢复操作。
## 七、生产环境数据库备份策略
### 一、备份频率建议
#### 1. 全量备份频率
- **建议周期：每周 1 次全量备份**  
  - **理由**：  
    - 全量备份是恢复的基础，确保数据的完整性。  
    - 如果数据量较小（如 TB 级以下），建议每周执行一次全量备份。  
    - 如果数据量极大（如数十 TB），可延长至每 2 周一次，但需结合增量备份策略降低风险。  
  - **执行时间**：  
    - 选择业务低峰期（如凌晨）以减少对 I/O 和 CPU 的影响。

#### 2. 增量备份频率
- **建议周期：每日 1 次增量备份**  
  - **理由**：  
    - 增量备份仅捕获自上次全量或增量后的数据变化，资源占用低。  
    - 每日一次可平衡 RPO（恢复点目标）与存储成本。  
  - **特殊情况调整**：  
    - 若数据变更频繁（如电商大促期间），可缩短至每小时一次增量备份。  
    - 若数据变更极少（如静态配置库），可放宽至每 2-3 天一次。

---

### 二、备份策略示例
```plaintext
周一：全量备份（base）
周二~周日：每日增量备份（inc1, inc2, ..., inc6）
下周一：新一轮全量备份 + 清理旧备份链
```

---

### 三、关键注意事项
#### 1. 备份保留策略
- 保留至少 **2-3 个全量备份链**（如最近 2 周的备份），防止备份损坏或误删。  
- 增量备份需与其依赖的全量备份共同保留，确保恢复链完整。

#### 2. 恢复时间优化
- 增量备份链不宜过长（建议不超过 7 次增量），否则恢复时合并时间可能超出 RTO。  
- 若增量链过长，可调整全量备份频率（如每 3 天一次全量）。

#### 3. 结合 Binlog 实现精细化恢复
- 启用 MySQL Binlog，按需（如每 5-15 分钟）归档 Binlog 文件。  
- 恢复时，先通过 XtraBackup 还原到最近备份点，再通过 Binlog 恢复到精确时间点。

#### 4. 监控与验证
- 监控备份任务状态，确保无失败或遗漏。  
- 定期演练恢复流程（如每季度一次），验证备份有效性。

---

### 四、典型场景调整建议
| **场景**               | **全量备份频率** | **增量备份频率**      | **备注**                          |
|------------------------|------------------|-----------------------|-----------------------------------|
| 小型数据库（< 100GB）  | 每周 1 次        | 每日 1 次             | 恢复速度快，资源压力小。          |
| 中型数据库（100GB~1TB）| 每周 1 次        | 每日 1 次（或 12h/次）| 增量频率可根据白天/夜间负载调整。 |
| 大型数据库（> 1TB）    | 每周 1 次        | 每日 2-4 次           | 缩短增量间隔，避免单次增量过大。  |
| 超高写入负载数据库     | 每 3 天 1 次     | 每小时 1 次           | 通过更频繁全量备份减少增量链长度。|

---


### 总结
- **核心原则**：全量备份频率由数据量和恢复时间容忍度决定，增量备份频率由数据变更速度决定。  
- **平衡点**：在存储成本、备份资源开销、RTO/RPO 之间找到最佳平衡。  
- **动态调整**：根据业务周期（如节假日、促销）灵活调整备份策略。

## 八、关键注意事项

1. **定期验证备份**：通过恢复测试确保备份有效性。
2. **监控备份任务**：结合 cron 或工具（如 Rundeck）自动化备份。
3. **备份保留策略**：保留多个全量+增量备份点，避免单点故障。
4. **混合使用策略**：XtraBackup + Binlog 提供双重保障，支持时间点恢复（PITR）。

通过合理选择工具和策略，可确保 MySQL 数据的高效备份与可靠恢复。

## 九、最佳实践
1. **始终检查版本兼容性**：在升级 MySQL 或 XtraBackup 前，查阅 [Percona 官方文档](https://docs.percona.com/percona-xtrabackup/latest/compatibility.html)。
2. **测试备份恢复流程**：在非生产环境验证备份文件的可用性。
3. **自动化版本管理**：使用 Ansible/Puppet 等工具确保环境版本一致。
4. **将被备份数据放到远程数据库主机下**：提升安全性


## 参考
1. [XtraDB Backup 官网](https://www.percona.com/mysql/software/percona-xtrabackup)
2. [Percona Operator for MySQL based on Percona Server for MySQL](https://docs.percona.com/percona-server/5.7/)
3. [XtraDB Backup 下载](https://www.percona.com/downloads)
4. [autoxtrabackup](https://github.com/gstorme/autoxtrabackup/tree/master) 开源的xtrabackup自动备份脚本