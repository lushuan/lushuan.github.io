---
title: "Git提交代码后发起Merge全流程示例"
subtitle: ""
date: 2025-05-17T12:06:37+08:00
lastmod: 2025-05-17T12:06:37+08:00
draft: false
toc:
enable: true
weight: false
categories: ["Git"]
tags: ["Git"]
---
## 背景

* 场景：一个团队中，**员工负责开发功能并提交合并请求 (Merge Request)**
* **技术经理** 负责 **Code Review → Merge → 验收 → 回滚（如有问题）**
* **全程用命令行 git 操作**，没有 GUI。
---
## 🧭 场景说明
* **主分支**：`master`
* **开发分支命名规则**：`feature/<功能名>`，例如：`feature/add-login-api`
* **远程仓库**：`origin`
* **角色**：

    * 👩‍💻 员工：负责创建分支、开发、提交代码、推送到远程、发起 merge request
    * 👨‍💼 技术经理：负责 Code Review、合并、验证、必要时回滚

---

# 🌱 第一阶段：员工开发流程

## 1️⃣ 克隆项目

```bash
git clone git@github.com:company/project.git
cd project
```

> 克隆远程仓库到本地。之后所有操作都在该项目目录下进行。

---

## 2️⃣ 从 master 拉取最新代码

```bash
git checkout master
git pull origin master
```

> 确保自己基于最新的主干开发，避免冲突。

---

## 3️⃣ 创建自己的功能分支

```bash
git checkout -b feature/add-login-api
```

> 创建并切换到自己的分支，命名要有意义（`feature/` 开头表示新功能）。

---

## 4️⃣ 编写代码、提交更改

```bash
git add .
git commit -m "feat: add login API with JWT authentication"
```

> * `git add .` 表示把修改的文件加入暂存区
> * `git commit -m` 提交说明必须简洁明确（推荐遵守 [Conventional Commit](https://www.conventionalcommits.org/) 规范）

---

## 5️⃣ 推送到远程分支

```bash
git push origin feature/add-login-api
```

> 把分支上传到远程仓库，供技术经理审核。

---

## 6️⃣ 发起 Merge Request

登录 GitLab/GitHub/Gitee →
发起一个从 `feature/add-login-api` → `master` 的 **Merge Request (MR)**。
写清楚：

* 变更内容
* 测试说明
* 可能的影响范围

![Git Merge Request](/images/git/git-1.png "Git Merge Request")

> 💡 这步通常是网页上操作（不是 git 命令）。

---

# 🔍 第二阶段：技术经理 Code Review + 合并流程

## 1️⃣ 拉取最新远程分支

```bash
git fetch origin
```

> 获取所有远程更新（包括员工新建的 feature 分支）。

---

## 2️⃣ 检查员工的分支

```bash
git checkout feature/add-login-api
```

> 切换到该功能分支。

然后查看变更：

```bash
git log --oneline --graph --decorate
git diff master
```

> * `git diff master`：查看和主分支的代码差异
> * `git log`：确认提交历史是否干净，有没有多余的提交

---

## 3️⃣ 本地测试 / 验收

> 在本地编译、运行单元测试，确认功能正常、无冲突。

---

## 4️⃣ 合并到 master

切回 master 分支：

```bash
git checkout master
git pull origin master
```

合并 feature 分支：

```bash
git merge --no-ff feature/add-login-api -m "merge: add login API feature"
```

> `--no-ff` 表示保留合并节点，方便以后追踪是谁合并的。

---

## 5️⃣ 推送合并结果到远程 master

```bash
git push origin master
```

> 把合并后的主干更新到远程仓库。
> 此时 Merge Request 可在平台上关闭或自动关闭。

---

## 6️⃣ 清理无用分支

合并完成后，可以删除本地和远程的 feature 分支：

```bash
git branch -d feature/add-login-api          # 删除本地分支
git push origin --delete feature/add-login-api  # 删除远程分支
```

---

# 🚨 第三阶段：合并后发现问题 → 回滚方案

假设技术经理合并后发现 master 出了问题。

有两种情况：

---

### 🧩 情况 1：只想撤销最近一次合并提交

1️⃣ 查看提交历史：

```bash
git log --oneline
```

找到合并提交的 ID（假设是 `abc1234`）

2️⃣ 执行回滚：

```bash
git revert -m 1 abc1234
```

> `-m 1` 表示保留主分支（master）为主线，撤销 merge 分支的改动。
> 这会生成一个新的提交，用来反向取消那次合并。

3️⃣ 推送修改：

```bash
git push origin master
```

---

### 🧨 情况 2：回滚到某个稳定版本（比如上一个成功的 commit）

查看提交历史：

```bash
git log --oneline
```

找到一个稳定版本（假设是 `789abcd`），然后执行：

```bash
git reset --hard 789abcd
git push origin master --force
```

> **⚠️ 警告：**
> 这种方法会重写远程历史（危险操作），必须 **确保团队成员知道**，否则他们本地的 master 会混乱。

---

# 🧹 第四阶段：日常维护建议

* 每次合并前，务必：

  ```bash
  git pull origin master
  ```

  保证自己的分支基于最新的主干。
* 命名规范化：

  ```
  feature/*     → 新功能
  fix/*         → 修复
  hotfix/*      → 紧急修复
  refactor/*    → 重构
  ```
* 提交说明规范化（示例）：

  ```
  feat: add login api
  fix: correct user token expiry
  refactor: simplify session handling
  ```

---

# ✅ 总结：完整命令流程复盘

| 角色   | 操作步骤     | 命令                                                                                         |
| ---- | -------- | ------------------------------------------------------------------------------------------ |
| 员工   | 同步主干     | `git pull origin master`                                                                   |
| 员工   | 创建分支     | `git checkout -b feature/add-login-api`                                                    |
| 员工   | 开发提交     | `git add . && git commit -m "feat: add login api"`                                         |
| 员工   | 推送分支     | `git push origin feature/add-login-api`                                                    |
| 员工   | 发起 MR    | （网页操作）                                                                                     |
| 技术经理 | 拉取更新     | `git fetch origin`                                                                         |
| 技术经理 | 检查分支     | `git checkout feature/add-login-api`                                                       |
| 技术经理 | 查看差异     | `git diff master`                                                                          |
| 技术经理 | 测试通过后合并  | `git checkout master && git merge --no-ff feature/add-login-api -m "merge: add login api"` |
| 技术经理 | 推送主干     | `git push origin master`                                                                   |
| 技术经理 | 清理分支     | `git branch -d feature/add-login-api && git push origin --delete feature/add-login-api`    |
| 技术经理 | 回滚合并（如需） | `git revert -m 1 <merge_commit_id>`                                                        |

