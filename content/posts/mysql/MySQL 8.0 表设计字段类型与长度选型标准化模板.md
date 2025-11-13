---
title: "MySQL 8.0 字符存储机制和字段长度设计规范参考"
subtitle: ""
date: 2024-05-14T12:06:37+08:00
lastmod: 2024-05-16T12:06:37+08:00
draft: false
toc:
enable: true
weight: false
categories: ["MySQL"]
tags: ["MySQL"]
---


## 🧩 一、基础配置建议

| 项目     | 推荐设置                             | 说明                         |
| ------ | -------------------------------- | -------------------------- |
| 字符集    | `utf8mb4`                        | 支持中日韩文字及 emoji             |
| 排序规则   | `utf8mb4_general_ci`             | 性能与兼容性均衡                   |
| 存储引擎   | `InnoDB`                         | 默认事务型引擎                    |
| 时间精度   | `DATETIME(3)` / `TIMESTAMP(3)`   | 毫秒精度时间戳                    |
| 主键类型   | `BIGINT UNSIGNED AUTO_INCREMENT` | 高并发安全、自增ID                 |
| 默认命名规范 | 全小写+下划线                          | 例：`user_info`、`created_at` |

---

## 🧱 二、通用字段模板（推荐统一使用）

| 字段名          | 类型                               | 默认值                                                     | 说明         |
| ------------ | -------------------------------- | ------------------------------------------------------- | ---------- |
| `id`         | `BIGINT UNSIGNED AUTO_INCREMENT` | -                                                       | 主键，自增      |
| `created_at` | `DATETIME(3)`                    | `CURRENT_TIMESTAMP(3)`                                  | 创建时间       |
| `updated_at` | `DATETIME(3)`                    | `CURRENT_TIMESTAMP(3)` ON UPDATE `CURRENT_TIMESTAMP(3)` | 更新时间       |
| `deleted_at` | `DATETIME(3)`                    | `NULL`                                                  | 删除时间（逻辑删除） |
| `created_by` | `BIGINT UNSIGNED`                | `NULL`                                                  | 创建人ID      |
| `updated_by` | `BIGINT UNSIGNED`                | `NULL`                                                  | 更新人ID      |
| `deleted_by` | `BIGINT UNSIGNED`                | `NULL`                                                  | 删除人ID      |
| `remark`     | `VARCHAR(255)`                   | `NULL`                                                  | 备注信息       |

---

## 🧠 三、常见字段类型与长度选型

| 场景           | 字段名                           | 类型                                           | 推荐长度      | 说明               |
| ------------ | ----------------------------- | -------------------------------------------- | --------- | ---------------- |
| **主键**       | id                            | BIGINT UNSIGNED                              | -         | 自增主键             |
| **名称类**      | name / title                  | VARCHAR(60)                                  | 60        | 一般名称、标题、标识       |
| **描述类**      | description / remark          | VARCHAR(255) / TEXT                          | 255 / 长文本 | 可写说明内容           |
| **邮箱**       | email                         | VARCHAR(100)                                 | 100       | 常见邮箱最大约 70       |
| **电话**       | phone                         | CHAR(11)                                     | 11        | 固定长度手机号          |
| **用户名**      | username                      | VARCHAR(50)                                  | 50        | 英文或拼音用户名         |
| **密码**       | password                      | CHAR(60)                                     | 60        | bcrypt 哈希值长度     |
| **性别**       | gender                        | ENUM('male','female','unknown')              | -         | 性别枚举             |
| **状态**       | status                        | ENUM('enabled','disabled','pending','error') | -         | 业务状态             |
| **IP 地址**    | ip / cluster_ip / external_ip | VARCHAR(45)                                  | 45        | 兼容 IPv4 / IPv6   |
| **URL / 链接** | url                           | VARCHAR(2083)                                | 2083      | 浏览器最大 URL 长度     |
| **UUID**     | uuid                          | CHAR(36)                                     | 36        | 36位 UUID 格式      |
| **金额**       | amount / price                | DECIMAL(10,2)                                | -         | 最大 99999999.99   |
| **数量**       | count / total                 | INT UNSIGNED                                 | -         | 整数统计值            |
| **布尔状态**     | is_active / enabled           | TINYINT(1)                                   | -         | 0 / 1 表示真假       |
| **JSON 配置**  | config / meta                 | JSON                                         | -         | 存储动态数据结构         |
| **时间戳**      | create_time / update_time     | DATETIME(3)                                  | -         | 带毫秒时间字段          |
| **地区代码**     | country_code                  | CHAR(2)                                      | -         | ISO 3166-1，例如 CN |
| **版本号**      | version                       | VARCHAR(20)                                  | 20        | 软件或接口版本号         |
| **语言**       | language                      | VARCHAR(10)                                  | 10        | 例如 zh-CN、en-US   |

---

## 🧮 四、设计规范建议

### 1️⃣ 字符类型选择

| 内容                | 建议类型         |
| ----------------- | ------------ |
| 可变长度字符串           | `VARCHAR(n)` |
| 固定长度字符串（手机号、UUID） | `CHAR(n)`    |
| 超长文本              | `TEXT`       |
| 动态结构化数据           | `JSON`       |

---

### 2️⃣ 数字类型选择

| 场景       | 类型                | 说明         |
| -------- | ----------------- | ---------- |
| 主键、自增 ID | `BIGINT UNSIGNED` | 0～2^64     |
| 普通整数     | `INT`             | 常规计数       |
| 小整数      | `SMALLINT`        | 0～65535    |
| 状态码、标志位  | `TINYINT`         | 0～255      |
| 金额、比率    | `DECIMAL(10,2)`   | 精确存储，不丢失小数 |

---

### 3️⃣ 时间类型选择

| 类型             | 推荐用途   | 说明             |
| -------------- | ------ | -------------- |
| `DATETIME(3)`  | 通用时间字段 | 可存 1000-9999 年 |
| `TIMESTAMP(3)` | 记录修改时间 | 自动受时区影响        |
| `DATE`         | 仅日期    | 例如生日、到期日       |
| `TIME`         | 仅时间    | 例如打卡时间         |

---

### 4️⃣ 约束与默认值

* 必须定义主键；
* 时间字段默认自动填充；
* 所有 `VARCHAR` 字段必须指定长度；
* 使用 `ENUM` 时务必定义默认值；
* JSON 字段不建议建立索引；
* 所有金额、统计类字段应定义为非负数（`UNSIGNED`）。

---

## 🧱 五、模板示例：标准业务表

```sql
CREATE TABLE `user_account` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `username` VARCHAR(50) NOT NULL COMMENT '用户名',
  `email` VARCHAR(100) DEFAULT NULL COMMENT '邮箱地址',
  `phone` CHAR(11) DEFAULT NULL COMMENT '手机号',
  `password` CHAR(60) NOT NULL COMMENT '加密密码',
  `status` ENUM('enabled','disabled') DEFAULT 'enabled' COMMENT '账户状态',
  `role` ENUM('admin','user','guest') DEFAULT 'user' COMMENT '角色',
  `last_login_ip` VARCHAR(45) DEFAULT NULL COMMENT '最近登录IP',
  `last_login_time` DATETIME(3) DEFAULT NULL COMMENT '最近登录时间',
  `remark` VARCHAR(255) DEFAULT NULL COMMENT '备注',
  `created_at` DATETIME(3) DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `updated_at` DATETIME(3) DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户账户信息表';
```

---

## 🧾 六、附录：长度选择经验法则

| 类型             | 推荐长度          | 可存内容   |
| -------------- | ------------- | ------ |
| `VARCHAR(10)`  | 10 字符 ≈ 10 汉字 | 简短标签   |
| `VARCHAR(50)`  | 50 字符         | 姓名、短标题 |
| `VARCHAR(100)` | 邮箱、域名         |        |
| `VARCHAR(255)` | 描述、路径、配置项     |        |
| `VARCHAR(512)` | 中等长度内容        |        |
| `TEXT`         | 长文本、日志、配置文件   |        |
| `JSON`         | 动态结构数据（如端口映射） |        |


