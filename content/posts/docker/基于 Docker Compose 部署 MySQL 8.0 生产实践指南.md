---
title: "基于 Docker Compose 部署 MySQL 8.0 生产实践指南（Kylin V10 / RHEL 系列）| 附完整 yml 与运维命令"
subtitle: ""
date: 2025-12-17T16:25:58+08:00
lastmod: 2025-12-17T16:25:58+08:00
draft: true
toc:
  enable: true
weight: false
categories: ["Docker"]
tags: ["Docker Compose","MySQL 8.0"]
---

> **环境声明**：本文基于以下已完成的环境进行部署操作
> - 操作系统：Kylin V10 / CentOS / RHEL 系列
> - Docker：25.0.0（已安装，systemd 托管）
> - Docker Compose：v2.24.1（插件模式）
> - MySQL 8.0 镜像：已通过 `docker load` 导入本地

```bash
[root@master ~]# docker --version
Docker version 25.0.0, build e758fe5

[root@master ~]# docker compose version
Docker Compose version v2.24.1

[root@master ~]# docker images | grep mysql
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
mysql        8.0       6c55ddbef969   22 months ago   591MB
```

---

## 一、安装 MySQL 客户端

在 Kylin V10 及 RedHat 系列（CentOS、RHEL、Rocky、AlmaLinux）中，**MySQL 客户端已集成在 MariaDB 软件包中**，无需单独安装 MySQL 官方客户端。

```bash
# Kylin V10 / CentOS / RHEL 系列统一使用以下方式安装
yum install -y mariadb

# 或者（RHEL 8+ / Kylin V10 SP2+）
dnf install -y mariadb
```

安装完成后验证：

```bash
>$ mysql --version
mysql  Ver 8.0.26 for Linux on x86_64 (Source distribution)
# 输出示例：mysql  Ver 15.1 Distrib 10.x.x-MariaDB, for Linux (x86_64)
```

> 💡 **说明**：MariaDB 客户端与 MySQL 8.0 完全兼容，可正常连接和操作 MySQL 8.0 服务。Kylin V10 默认源中已包含该包，离线环境可使用 `rpm -ivh` 安装。

---

## 二、目录规划

```bash
# 创建标准目录结构
mkdir -p /opt/docker/mysql/{data,conf,logs,backup}

# 设置 MySQL 容器运行用户权限（UID 999）
chown -R 999:999 /opt/docker/mysql/data
chown -R 999:999 /opt/docker/mysql/logs

# 最终目录结构
/opt/docker/mysql/
├── docker-compose.yml    # 编排文件
├── backup.sh             # 自动备份脚本
├── conf/
│   └── my.cnf            # MySQL 主配置文件
├── data/                 # 数据文件持久化（自动生成）
├── logs/                 # 错误日志 / 慢查询日志
└── backup/               # 备份文件存放
```
{{< admonition type=info >}}
MySQL 官方镜像内部运行用户 UID/GID 为 `999`。
{{< /admonition >}}

---

## 三、MySQL 配置文件

```bash
cat > /opt/docker/mysql/conf/my.cnf << 'EOF'
[mysqld]
# ===== 基础 =====
server-id                = 1
port                     = 3306
bind-address             = 0.0.0.0
default-authentication-plugin = mysql_native_password

# ===== 字符集与时区 =====
character-set-server     = utf8mb4
collation-server         = utf8mb4_general_ci
default-time-zone        = '+08:00'

# ===== 连接 =====
max_connections          = 500
max_connect_errors       = 100000
wait_timeout             = 28800
interactive_timeout      = 28800

# ===== InnoDB =====
innodb_buffer_pool_size  = 1G
innodb_log_file_size     = 256M
innodb_flush_log_at_trx_commit = 1
innodb_file_per_table    = 1

# ===== 慢查询 =====
slow_query_log           = 1
slow_query_log_file      = /var/log/mysql/slow.log
long_query_time          = 2

# ===== Binlog（备份 / 主从） =====
log-bin                  = mysql-bin
binlog_format            = ROW
binlog_expire_logs_seconds = 604800
max_binlog_size          = 256M

# ===== 其他 =====
skip-name-resolve
lower_case_table_names   = 1
sql_mode                 = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
explicit_defaults_for_timestamp = 1

[client]
default-character-set    = utf8mb4

[mysql]
default-character-set    = utf8mb4
EOF
```

> ⚠️ `innodb_buffer_pool_size` 调优参考：数据库专用机设为物理内存的 60%~70%，混部场景设为 30%~40%。

---

## 四、docker-compose.yml

```bash
cat > /opt/docker/mysql/docker-compose.yml << 'EOF'
services:
  mysql:
    image: mysql:8.0
    container_name: mysql8
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: "YourStr0ng!Pass@2026"
      MYSQL_DATABASE: "app_db"
      MYSQL_USER: "app_user"
      MYSQL_PASSWORD: "AppUs3r!Pass@2026"
      TZ: "Asia/Shanghai"
    volumes:
      - ./data:/var/lib/mysql
      - ./conf/my.cnf:/etc/mysql/my.cnf:ro
      - ./logs:/var/log/mysql
      - ./backup:/backup
    command:
      - --default-authentication-plugin=mysql_native_password
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p$$MYSQL_ROOT_PASSWORD"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "2.0"
        reservations:
          memory: 512M
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"
EOF
```

---

## 五、启动服务

```bash
cd /opt/docker/mysql

# 启动
docker compose up -d

# 查看状态（首次初始化约 10~30 秒）
docker compose ps

# 查看初始化日志
docker compose logs -f mysql
```

等待 `docker compose ps` 中 STATUS 显示 **`(healthy)`** 即表示 MySQL 已就绪：

```
NAME      IMAGE       STATUS                   PORTS
mysql8    mysql:8.0   Up 30 seconds (healthy)  0.0.0.0:3306->3306/tcp
```

---

## 六、验证部署

### 6.1 第一步：进入容器内部验证（无需客户端）

部署完成后，首先通过 `docker exec` 进入容器确认 MySQL 正常运行：

```bash
# 进入 MySQL 命令行
docker exec -it mysql8 mysql -uroot -p'YourStr0ng!Pass@2026'
```

在 MySQL 终端中执行验证：

```sql
-- 确认版本
SELECT VERSION();

-- 确认时区
SELECT NOW();

-- 确认字符集
SHOW VARIABLES LIKE 'character_set%';

-- 确认数据库已创建
SHOW DATABASES;

-- 确认用户
SELECT user, host FROM mysql.user;

-- 退出
exit;
```

### 6.2 第二步：使用宿主机 MySQL 客户端连接（日常运维）

验证容器内部正常后，后续统一使用宿主机安装的 `mysql` 客户端进行操作：

```bash
# 注意：必须使用 -h 127.0.0.1，不能省略（否则走 socket 会报错）
mysql -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' -P3306 -e "SELECT VERSION();"
```

输出：
```
+-----------+
| VERSION() |
+-----------+
| 8.0.40    |
+-----------+
```

> ⚠️ **重要**：Docker 部署的 MySQL，宿主机连接时**必须指定 `-h 127.0.0.1`**。  
> 直接执行 `mysql -uroot -p` 会尝试通过 Unix Socket 连接，而 socket 文件在容器内部，宿主机上不存在，将报 `ERROR 2002`。

### 6.3 设置连接别名（提升效率）

```bash
cat >> /etc/profile.d/mysql-alias.sh << 'EOF'
alias dmysql='mysql -h 127.0.0.1 -P 3306'
EOF

source /etc/profile.d/mysql-alias.sh

# 之后直接使用
mysql -uroot -p'YourStr0ng!Pass@2026' -e "SHOW DATABASES;"
```

---

## 七、运维常用命令

### 7.1 容器管理

```bash
cd /opt/docker/mysql

docker compose up -d        # 启动
docker compose stop         # 停止
docker compose restart      # 重启
docker compose down         # 停止并删除容器（数据保留）
docker compose logs -f --tail=100 mysql   # 查看日志
docker stats mysql8 --no-stream            # 资源占用
```

### 7.2 日常 SQL 操作

```bash
# 执行单条 SQL
mysql -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' -e "SHOW DATABASES;"

# 查看连接数
mysql -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' -e "SHOW STATUS LIKE 'Threads_connected';"

# 查看当前进程
mysql -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' -e "SHOW FULL PROCESSLIST;"

# 查看慢查询数
mysql -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' -e "SHOW GLOBAL STATUS LIKE 'Slow_queries';"
```

### 7.3 日志查看

```bash
# 容器日志
docker compose logs --tail=50 mysql

# MySQL 错误日志
tail -f /opt/docker/mysql/logs/error.log

# 慢查询日志
tail -f /opt/docker/mysql/logs/slow.log
```

### 7.4 备份与恢复

```bash
# 全库备份
mysqldump -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' \
  --all-databases --single-transaction --routines --triggers --events \
  | gzip > /opt/docker/mysql/backup/all_db_$(date +%Y%m%d_%H%M%S).sql.gz

# 单库备份
mysqldump -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' \
  --single-transaction app_db \
  | gzip > /opt/docker/mysql/backup/app_db_$(date +%Y%m%d_%H%M%S).sql.gz

# 恢复
gunzip < /opt/docker/mysql/backup/app_db_20260811.sql.gz \
  | mysql -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' app_db
```

### 7.5 自动备份脚本

```bash
cat > /opt/docker/mysql/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/docker/mysql/backup"
DATE=$(date +%Y%m%d_%H%M%S)
KEEP_DAYS=7

mysqldump -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' \
  --all-databases --single-transaction --routines --triggers --events \
  | gzip > ${BACKUP_DIR}/all_db_${DATE}.sql.gz

# 清理过期备份
find ${BACKUP_DIR} -name "*.sql.gz" -mtime +${KEEP_DAYS} -delete

echo "[$(date)] Backup completed: all_db_${DATE}.sql.gz" >> ${BACKUP_DIR}/backup.log
EOF

chmod +x /opt/docker/mysql/backup.sh

# 添加定时任务：每天凌晨 2 点执行
echo "0 2 * * * /opt/docker/mysql/backup.sh" | crontab -
```

### 7.6 配置变更

```bash
# 修改配置
vim /opt/docker/mysql/conf/my.cnf

# 重启生效
docker compose restart mysql

# 部分参数可动态修改（无需重启）
mysql -h 127.0.0.1 -uroot -p'YourStr0ng!Pass@2026' \
  -e "SET GLOBAL max_connections = 1000;"
```

---

## 八、生产环境使用建议

### 8.1 资源与性能

| 建议项 | 说明 |
|:---|:---|
| `innodb_buffer_pool_size` | 根据实际内存调整，专用机 60%~70%，混部 30%~40% |
| `deploy.resources.limits` | 必须设置内存上限，防止 OOM 影响宿主机其他服务 |
| 数据目录磁盘 | 建议使用独立磁盘或 SSD 挂载到 `/opt/docker/mysql/data` |
| 日志轮转 | `logging.max-size` + `max-file` 已配置，防止磁盘打满 |

### 8.2 数据安全

| 建议项 | 说明 |
|:---|:---|
| 备份策略 | 全量备份（每日）+ Binlog 增量（实时），保留 7 天 |
| 备份验证 | 每周至少恢复一次备份到测试环境验证完整性 |
| 密码管理 | 不要明文写在脚本中，使用 `.env` 文件或 Vault |
| Binlog | 已开启 ROW 模式，支持基于时间点恢复（PITR） |

### 8.3 高可用与容灾

| 建议项 | 说明 |
|:---|:---|
| `restart: always` | 已配置，宿主机重启后自动拉起 |
| 主从复制 | 生产环境建议至少一主一从，Binlog 已就绪 |
| 监控告警 | 接入 Prometheus + mysqld_exporter，监控连接数、慢查询、复制延迟 |
| 异地备份 | 备份文件定期同步至对象存储或异地服务器 |

### 8.4 运维规范

| 建议项 | 说明 |
|:---|:---|
| 禁止 `docker compose down -v` | 会删除数据卷，导致数据永久丢失 |
| 变更走流程 | 修改 `my.cnf` 或 `docker-compose.yml` 前先备份原文件 |
| 版本锁定 | 镜像使用 `mysql:8.0` 固定大版本，避免意外升级到 8.4 |
| 定期巡检 | 每周检查磁盘用量、备份完整性、慢查询 TOP 10 |

### 8.5 密码管理（.env 方式）

```bash
# 创建环境变量文件
cat > /opt/docker/mysql/.env << 'EOF'
MYSQL_ROOT_PASSWORD=YourStr0ng!Pass@2026
MYSQL_DATABASE=app_db
MYSQL_USER=app_user
MYSQL_PASSWORD=AppUs3r!Pass@2026
EOF

chmod 600 /opt/docker/mysql/.env
```

`docker-compose.yml` 中引用：

```yaml
environment:
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
  MYSQL_DATABASE: ${MYSQL_DATABASE}
  MYSQL_USER: ${MYSQL_USER}
  MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

> 好处：密码不进入 Git 仓库，`.env` 文件加入 `.gitignore`。

---

## 九、故障排查速查

| 症状 | 排查命令 | 常见原因 |
|:---|:---|:---|
| 容器启动失败 | `docker compose logs mysql` | 配置语法错误 / 端口占用 |
| ERROR 2002 | 确认使用 `-h 127.0.0.1` | 未指定 host，走了 socket |
| Permission denied | `ls -la /opt/docker/mysql/data` | 目录属主非 999:999 |
| 端口冲突 | `ss -tlnp \| grep 3306` | 宿主机已有服务占用 |
| 容器反复重启 | `docker inspect mysql8` 查看 State | OOM / 配置错误 |
| 初始化超时 | 检查 `start_period` 是否 ≥ 30s | 首次初始化 data 目录慢 |

---

## 十、最终文件清单

```
/opt/docker/mysql/
├── docker-compose.yml      # 编排定义
├── .env                    # 密码等敏感信息
├── backup.sh               # 自动备份脚本
├── conf/
│   └── my.cnf              # MySQL 配置
├── data/                   # 数据（自动产生）
├── logs/                   # 日志（自动产生）
└── backup/                 # 备份文件
```

---

*部署完成后，您即拥有一套数据持久化、自动拉起、定时备份、日志可追溯的生产级 MySQL 8.0 容器化实例。*