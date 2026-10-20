---
title: "Git 2.x 入门与日常使用教程：从提交、分支到远程协作，一篇真正搞懂 Git"
subtitle: ""
date: 2023-11-07T12:06:37+08:00
lastmod: 2023-11-07T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Git"]
tags: ["Git"]
slug: "git-tutorial"
featuredImage: "/images/git/cover.png"
summary: "本文基于 Git 2.41 系统讲解 Git 的核心概念与日常操作，覆盖工作区/暂存区/仓库模型、提交与撤销、分支管理、stash 暂存、远程协作及现代 switch/restore 命令的完整入门教程。"
---

这是一篇基于 Git 2.41 的入门与日常使用教程，目标不是记住几十条命令，而是建立"代码在哪里、改了什么、如何提交、如何回退、如何协作"的完整思维模型。内容覆盖工作区/暂存区/本地仓库/远程仓库四大核心概念、提交与撤销（restore/reset/revert/reflog）、分支与 stash、远程协作流程，以及现代 Git 推荐命令与老教程的差异。

<!--more-->

> 本文基于 Git 2.41 的使用方式进行整理。
>
> 目标不是记住几十条 Git 命令，而是搞清楚：
>
> **代码在哪里 → 改了什么 → 是否要提交 → 提交到哪里 → 如何回退 → 如何分支开发 → 如何与远程协作。**

---

## 一、Git 到底是什么？

Git 是一个**分布式版本控制系统**。

简单理解：

> **Git 就像给代码建立了一条可以随时查看、比较、回退的时间线。**

例如一个项目经历了：

```
第一次提交
    ↓
增加登录功能
    ↓
修复登录 Bug
    ↓
增加用户管理
    ↓
重构代码
```

Git 会把这些变化保存下来。

以后你可以：

-   查看以前改了什么

-   查看谁提交的

-   回到某个历史版本

-   创建不同开发分支

-   将不同分支合并

-   把代码同步到 GitHub / GitLab / Gitee

-   多人同时开发同一个项目


Git 的一个重要特点是：

> **本地仓库本身就是完整的版本库。**

所以很多 Git 操作即使没有网络也可以完成。

---

## 二、先理解 Git 的四个核心概念

刚开始学习 Git，最容易混乱的是：

```
工作区
暂存区
本地仓库
远程仓库
```

可以把它理解成：

```
                  git add
工作区 ─────────────────────→ 暂存区
  ↑                              │
  │                              │ git commit
  │                              ↓
  │                           本地仓库
  │                              │
  │                              │ git push
  │                              ↓
  │                           远程仓库
  │                              │
  │                              │ git fetch/pull
  └──────────────────────────────┘
```

---

### 1\. 工作区

就是你真正写代码的地方。

例如：

```
my-project/
├── main.go
├── go.mod
├── service/
└── README.md
```

你修改 `main.go`，这就是**工作区发生变化**。

---

### 2\. 暂存区

暂存区可以理解为：

> **准备下一次提交的文件清单。**

例如你修改了：

```
main.go
user.go
README.md
```

但是这次只想提交：

```
main.go
user.go
```

那么：

```
git add main.go
git add user.go
```

这两个文件进入暂存区。

然后：

```
git commit -m "新增用户功能"
```

只会提交已经进入暂存区的内容。

所以：

```
工作区
   │
   │ git add
   ↓
暂存区
   │
   │ git commit
   ↓
本地仓库
```

这是理解 Git 最重要的一条链路。

---

## 三、第一次使用 Git

### 1\. 查看 Git 版本

```
git --version
```

例如：

```
git version 2.41.0
```

---

### 2\. 配置用户名

```
git config --global user.name "你的名字"
```

例如：

```
git config --global user.name "张三"
```

---

### 3\. 配置邮箱

```
git config --global user.email "you@example.com"
```

查看配置：

```
git config --global --list
```

如果想知道配置到底来自哪个文件：

```
git config --list --show-origin
```

这个命令非常有用。

例如你排查 Git 代理问题时，可以看到：

```
file:C:/Users/lushu/.gitconfig
http.proxy=http://127.0.0.1:7897
```

甚至可以发现当前项目 `.git/config` 中存在额外配置。

---

## 四、创建一个 Git 仓库

如果你已经有一个项目：

```
cd my-project
```

执行：

```
git init
```

Git 会创建：

```
.git/
```

这个目录保存 Git 的版本信息。

一般情况下：

> **不要手动修改** `**.git**` **内部文件。**

---

## 五、查看当前状态：git status

这是 Git 日常开发中最常用的命令之一。

```
git status
```

例如：

```
modified: main.go
untracked: test.go
```

表示：

```
main.go
    已修改

test.go
    新文件，尚未被 Git 跟踪
```

一个非常实用的习惯：

> **不知道 Git 当前是什么状态时，先执行** `**git status**`**。**

---

## 六、查看修改内容：git diff

修改代码之后，可以：

```
git diff
```

查看：

> 工作区相对于暂存区发生了什么变化。

例如：

```
- old code
+ new code
```

如果已经 `git add`：

```
git diff
```

可能看不到已经暂存的修改。

这时使用：

```
git diff --cached
```

查看：

> 暂存区准备提交什么。

所以：

```
git diff
    ↓
看工作区修改

git diff --cached
    ↓
看暂存区修改
```

---

## 七、提交代码：add + commit

这是 Git 最核心的日常操作。

### 1\. 添加文件

```
git add main.go
```

添加所有修改：

```
git add .
```

---

### 2\. 提交

```
git commit -m "新增用户查询功能"
```

于是：

```
工作区
   ↓ git add
暂存区
   ↓ git commit
本地仓库
```

---

## 八、一次完整的日常开发流程

绝大多数普通开发工作，都可以浓缩成：

```
git status

git diff

git add .

git commit -m "完成 xxx 功能"

git push
```

如果不确定自己做了什么，可以：

```
git status
git diff
```

确认无误后：

```
git add .
git commit -m "xxx"
git push
```

---

## 九、查看提交历史

最基础：

```
git log
```

更适合日常查看：

```
git log --oneline
```

例如：

```
7e31c21 优化 PDF 导出
4a812ef 增加报告接口
```

查看图形化分支关系：

```
git log --oneline --graph --decorate --all
```

这个命令非常值得掌握。

---

## 十、Git 中的 HEAD 是什么？

可以简单理解：

> **HEAD 表示你当前正在什么位置。**

例如：

```
main
  ↓
A → B → C
        ↑
       HEAD
```

如果切换到另一个分支：

```
dev
  ↓
A → B → C → D
            ↑
           HEAD
```

HEAD 就跟着当前分支移动。

所以：

```
git status
```

通常会告诉你当前所在分支。

---

## 十一、撤销工作区修改

假设：

```
main.go
```

你修改了一堆代码，但是发现：

> 这些修改全部不要了。

Git 2.41 推荐使用：

```
git restore main.go
```

恢复所有工作区修改：

```
git restore .
```

注意：

> **这个操作会丢失尚未提交的工作区修改。**

所以执行之前一定确认。

---

## 十二、撤销 git add

假设：

```
git add main.go
```

之后发现：

> 我不想暂存这个文件了。

可以：

```
git restore --staged main.go
```

此时：

```
暂存区
   ↓ git restore --staged
工作区
```

注意：

> 它只是取消暂存，不会删除你写的代码。

---

## 十三、三个 restore 场景一定要分清

### ① 丢弃工作区修改

```
git restore main.go
```

意思：

> 我刚才修改的不要了。

---

### ② 取消暂存

```
git restore --staged main.go
```

意思：

> 不提交这个文件，但代码修改保留。

---

### ③ 同时取消暂存 + 丢弃修改

需要谨慎：

```
git restore --staged main.go
git restore main.go
```

最终恢复到最近一次提交的状态。

---

## 十四、撤销最近一次 commit

假设：

```
A → B → C
        ↑
       HEAD
```

刚提交完：

```
C
```

发现：

> 这个 commit 提交错了。

### 1\. 撤销 commit，但保留修改并继续暂存

```
git reset --soft HEAD~1
```

结果：

```
HEAD
 ↓
B

C 中的修改仍然保留在暂存区
```

适合：

> commit message 写错、想重新提交。

---

### 2\. 撤销 commit，同时取消暂存

```
git reset HEAD~1
```

修改仍然保留在工作区。

---

### 3\. 连代码修改一起删除

```
git reset --hard HEAD~1
```

非常危险。

因为：

> commit 和对应的工作区修改都会被丢弃。

因此：

```
soft
    ↓
保留修改 + 保留暂存

mixed
    ↓
保留修改 + 取消暂存

hard
    ↓
修改也删除
```

---

## 十五、已经提交并推送到远程怎么办？

这是非常重要的区别。

如果 commit：

```
只在本地
```

可以考虑：

```
git reset
```

但是如果：

```
已经 push 到 GitHub/GitLab
```

不要轻易：

```
git reset --hard
git push --force
```

因为这可能修改共享历史。

更安全的方式通常是：

```
git revert <commit>
```

例如：

```
git revert 9f9bc9a
```

Git 会创建一个新的 commit：

```
A → B → C → D
             ↑
          revert C
```

历史仍然完整。

---

## 十六、找回“丢失”的提交：git reflog

这是 Git 非常强大的功能。

例如：

```
git reset --hard HEAD~1
```

结果发现：

> 我刚才那个提交其实还想要。

可以：

```
git reflog
```

看到类似：

```
9f9bc9a HEAD@{1}: commit: pdf 工程实践
7e31c21 HEAD@{2}: checkout: moving from dev to main
```

然后：

```
git reset --hard 9f9bc9a
```

恢复过去的状态。

所以可以记住：

> `**git log**` **看提交历史，**`**git reflog**` **看 HEAD 曾经去过哪里。**

---

## 十七、删除文件

如果要删除文件：

```
git rm test.txt
```

然后：

```
git commit -m "删除 test.txt"
```

如果只是误删了文件，但还没有提交：

```
git restore test.txt
```

可以恢复。

---

## 十八、Git 分支是什么？

可以把分支理解成：

> **从某个提交开始，创建一条独立的开发路线。**

例如：

```
                 feature
                    ↓
A → B → C ─────────D
         \
          E → F
              ↑
             main
```

不同分支可以独立开发。

---

## 十九、查看分支

```
git branch
```

例如：

```
* main
  dev
  feature/login
```

`*` 表示当前分支。

---

## 二十、创建分支

Git 2.41 推荐：

```
git switch -c feature/login
```

等价于：

```
git branch feature/login
git switch feature/login
```

如果项目使用老版本 Git，也可以：

```
git checkout -b feature/login
```

所以：

> 新 Git 推荐 `switch`；老项目中仍经常看到 `checkout`。

---

## 二十一、切换分支

现代 Git：

```
git switch main
```

老方式：

```
git checkout main
```

---

## 二十二、删除分支

删除已经合并的本地分支：

```
git branch -d feature/login
```

强制删除：

```
git branch -D feature/login
```

`-D` 要谨慎，因为它可能删除尚未合并的分支。

---

## 二十三、合并分支

假设：

```
main
feature/login
```

现在想：

> 把 feature/login 合并到 main。

一定记住：

> **先进入目标分支，再执行 merge。**

```
git switch main
git merge feature/login
```

意思是：

```
feature/login
       ↓
      main
```

不是反过来。

---

## 二十四、什么是 Fast-forward？

如果：

```
A → B → C
         ↑
      feature
```

而 main 还停留在：

```
A → B
     ↑
    main
```

执行：

```
git merge feature
```

Git 可以直接把 main 移动到 C。

这就是：

```
Fast-forward
```

---

## 二十五、什么是 Merge Conflict？

例如：

```
main
    修改 main.go 第 10 行

feature
    也修改 main.go 第 10 行
```

两个分支对同一位置进行了不同修改。

Git 无法判断：

> 到底应该保留谁？

于是产生冲突。

文件中可能出现：

```
<<<<<<< HEAD
main 分支代码
=======
feature 分支代码
>>>>>>> feature
```

人工修改成最终正确代码后：

```
git add main.go
git commit
```

完成冲突解决。

如果发现：

> 这次 merge 我不想继续了。

可以：

```
git merge --abort
```

回到 merge 前的状态。

---

## 二十六、推荐一种实际开发方式

例如：

```
main
 │
 └── dev
      │
      ├── feature/login
      ├── feature/report
      └── bugfix/pdf
```

日常：

```
main
    ↓
稳定版本

dev
    ↓
日常开发

feature/*
    ↓
具体功能

bugfix/*
    ↓
Bug 修复
```

开发完成后：

```
feature/login
       ↓
      dev
       ↓
      main
```

具体采用哪种分支模型，要根据团队实际流程决定。

不要机械照搬某一种 Git Flow。

---

## 二十七、临时切走当前工作：git stash

这是实际工作中非常常用的功能。

例如你正在：

```
feature/report
```

代码写了一半：

```
A
B
C
```

但是突然需要：

> 紧急修复一个 Bug。

此时这些代码又不适合 commit。

可以：

```
git stash
```

当前工作就被暂时保存起来。

然后：

```
git switch main
```

处理紧急任务。

---

## 二十八、查看 stash

```
git stash list
```

例如：

```
stash@{0}: WIP on feature/report
stash@{1}: WIP on dev
```

---

## 二十九、恢复 stash

恢复但保留 stash：

```
git stash apply
```

恢复并删除 stash：

```
git stash pop
```

删除：

```
git stash drop stash@{0}
```

所以：

```
stash apply
    ↓
恢复，但 stash 还在

stash pop
    ↓
恢复，同时删除 stash
```

---

## 三十、远程仓库是什么？

Git 本身是版本控制系统。

GitHub、GitLab、Gitee 等则是：

> **承载 Git 远程仓库的平台。**

例如：

```
本地 Git 仓库
       │
       │ push
       ↓
GitHub
```

---

## 三十一、查看远程仓库

```
git remote
```

查看详细地址：

```
git remote -v
```

例如：

```
origin  https://github.com/zhangsan/zhangsan.github.io.git (fetch)
origin  https://github.com/zhangsan/zhangsan.github.io.git (push)
```

这里的：

```
origin
```

只是一个**远程仓库别名**。

它不是 GitHub 的固定名称。

你完全可以叫：

```
github
gitee
company
upstream
```

但如果只有一个远程仓库，通常使用：

```
origin
```

最符合 Git 社区习惯。

---

## 三十二、添加远程仓库

```
git remote add origin https://github.com/username/project.git
```

查看：

```
git remote -v
```

---

## 三十三、修改远程仓库地址

如果 remote 已经存在：

```
git remote set-url origin https://github.com/username/project.git
```

不需要删除再重新添加。

---

## 三十四、修改 remote 名称

例如：

```
github
```

改成：

```
origin
```

执行：

```
git remote rename github origin
```

然后：

```
git remote -v
```

得到：

```
origin  https://github.com/zhangsan/zhangsan.github.io.git (fetch)
origin  https://github.com/zhangsan/zhangsan.github.io.git (push)
```

---

## 三十五、推送到远程仓库

第一次：

```
git push -u origin main
```

这里：

```
-u
```

的作用是：

> 建立本地 `main` 与远程 `origin/main` 的关联。

以后就可以：

```
git push
```

而不需要每次写：

```
git push origin main
```

---

## 三十六、理解 upstream

这是很多 Git 初学者容易忽略的概念。

执行：

```
git push -u origin main
```

以后：

```
本地 main
      │
      │ upstream
      ↓
origin/main
```

于是：

```
git push
```

Git 就知道：

> 当前 main 应该推送到 origin/main。

查看：

```
git branch -vv
```

例如：

```
* main  9f9bc9a [origin/main] pdf 工程实践
```

---

## 三十七、git fetch 和 git pull 的区别

这是必须理解的。

### git fetch

```
git fetch origin
```

意思：

> 把远程最新信息下载到本地，但不自动修改当前工作分支。

可以理解为：

```
GitHub
   ↓
本地远程跟踪分支
```

---

### git pull

```
git pull
```

可以简单理解为：

```
git fetch
+
把远程更新整合到当前分支
```

所以：

```
fetch
    ↓
只获取

pull
    ↓
获取 + 整合
```

实际工作中，如果你想先看看远程发生了什么：

```
git fetch
git log --oneline --graph --decorate --all
```

通常比直接 `git pull` 更容易理解和控制。

---

## 三十八、克隆远程仓库

如果远程已经有项目：

```
git clone https://github.com/username/project.git
```

例如：

```
git clone https://github.com/zhangsan/zhangsan.github.io.git
```

Git 会自动：

1.  下载仓库

2.  创建本地目录

3.  初始化 Git

4.  创建远程 `origin`

5.  建立远程跟踪关系


---

## 三十九、多人协作时的基本流程

假设团队共同开发：

```
origin/dev
```

你准备开发：

```
feature/report
```

可以：

```
git fetch origin

git switch dev

git pull

git switch -c feature/report
```

开发：

```
git add .
git commit -m "新增报告功能"
```

推送：

```
git push -u origin feature/report
```

然后通过 GitLab/GitHub 创建 Merge Request / Pull Request。

---

## 四十、如果 push 被拒绝怎么办？

常见情况：

```
rejected
non-fast-forward
```

通常意味着：

> 远程分支已经比本地更新。

例如：

```
远程：
A → B → C

本地：
A → B → D
```

你直接 push：

```
git push
```

Git 会拒绝。

先获取远程更新：

```
git fetch origin
```

然后根据团队策略决定：

```
git pull --rebase
```

或者：

```
git pull
```

解决冲突后：

```
git push
```

---

## 四十一、Git 中最重要的几个“不要”

### 1\. 不要随便 reset --hard

```
git reset --hard
```

可能直接丢失未提交代码。

---

### 2\. 不要随便 force push

```
git push --force
```

可能覆盖别人已经推送的提交。

如果确实需要强制推送，优先：

```
git push --force-with-lease
```

它比 `--force` 更安全。

---

### 3\. 不要看到冲突就直接删代码

先理解：

```
<<<<<<<
=======
>>>>>>>
```

分别代表什么，再决定最终代码。

---

### 4\. 不要把所有修改都直接 git add .

尤其是在大型项目中。

先：

```
git status
git diff
```

确认修改内容。

再：

```
git add
```

---

## 四十二、日常最值得掌握的 Git 命令

可以把 Git 命令分成几个类别。

### 状态

```
git status
```

### 查看修改

```
git diff
git diff --cached
```

### 提交

```
git add .
git commit -m "xxx"
```

### 历史

```
git log --oneline
git reflog
```

### 撤销

```
git restore <file>
git restore --staged <file>
git reset
git revert
```

### 分支

```
git branch
git switch
git switch -c
git merge
```

### 临时保存

```
git stash
git stash list
git stash pop
```

### 远程仓库

```
git remote -v
git remote add
git remote rename
git remote set-url
```

### 远程同步

```
git fetch
git pull
git push
```

---

## 四十三、真正需要记住的 Git 思维模型

不要试图背几十条命令。

首先记住这一张图：

```
                 git add
工作区 ─────────────────────→ 暂存区
  ↑                              │
  │                              │ git commit
  │                              ↓
  │                           本地仓库
  │                              │
  │                              │ git push
  │                              ↓
  │                           远程仓库
  │                              │
  │                              │ git fetch
  └──────────────────────────────┘
```

然后再记住：

```
git status
    ↓
我现在是什么状态？

git diff
    ↓
我改了什么？

git add
    ↓
我准备提交什么？

git commit
    ↓
我保存了一个版本

git log
    ↓
以前发生过什么？

git restore
    ↓
我不要当前修改了

git reset
    ↓
我要调整本地提交历史

git revert
    ↓
已经共享的提交，我用一个新提交把它抵消

git branch
    ↓
我要创建另一条开发路线

git merge
    ↓
我要把另一条开发路线合进来

git stash
    ↓
我暂时不想提交，但又要切走

git fetch
    ↓
远程有什么新东西？

git pull
    ↓
把远程更新拿下来并整合

git push
    ↓
把我的本地提交发布到远程
```

---

## 四十四、推荐的一套日常 Git SOP

如果是普通开发：

```
# 1. 查看状态
git status

# 2. 查看修改
git diff

# 3. 获取远程最新信息
git fetch

# 4. 开始开发
# ...

# 5. 再次检查
git status
git diff

# 6. 暂存
git add .

# 7. 提交
git commit -m "完成 xxx 功能"

# 8. 推送
git push
```

如果多人协作：

```
                    GitHub / GitLab
                           ↑
                           │ push
                           │
                     feature 分支
                           ↑
                           │ commit
                           │
                        本地仓库
                           ↑
                           │ add
                           │
                         工作区
```

这就是绝大多数软件开发人员每天实际使用 Git 的核心流程。

---

## 四十五、最后：不要把 Git 学成“命令大全”

Git 真正难的地方不是：

```
git xxx
```

而是理解：

```
当前在哪个分支？
       ↓
当前有什么修改？
       ↓
修改在哪个区域？
       ↓
准备提交什么？
       ↓
这个提交是否已经共享？
       ↓
远程是否比本地更新？
       ↓
这次操作会不会改写别人历史？
```

一旦这个思维建立起来，Git 命令反而很容易查。

**建议把 Git 的学习重点放在四条主线：**

```
① 工作区 → 暂存区 → 本地仓库
② 本地分支 → 远程分支
③ commit → merge → revert/reset
④ fetch → pull → push
```

这四条线真正掌握以后，日常开发中 80%～90% 的 Git 问题都能定位。

---

## 附：Git 2.41 与老教程的几个重要变化

老教程中可能大量使用：

```
git checkout
git reset --hard
master
git push origin master
```

这些命令本身并没有全部“过时”，但现代 Git 更推荐根据意图使用更明确的命令，例如：

```
git switch
git restore
main
git push -u origin main
```

特别是：

```
git checkout
```

过去同时承担：

```
切换分支
+
创建分支
+
恢复文件
```

现代 Git 将其中的职责拆开：

```
git switch
    → 分支切换

git restore
    → 文件恢复
```

这样对于初学者更加直观。

同时，`reset`、`revert`、`restore` 三者不要混为一谈：

```
restore
    → 修改文件状态

reset
    → 调整本地历史/HEAD

revert
    → 用新提交抵消已有提交
```

这也是现代 Git 学习中非常值得建立的一个核心概念。


