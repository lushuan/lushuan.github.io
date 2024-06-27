# Git 常用操作

## 创建版本库
```
$ git clone <url>   # 克隆远程仓库
$ git init          # 初始化本地仓库
```
## Git 全局设置
```
git config --global user.name "lu.shua"
git config --global user.email "lu.shuan@iwhalecloud.com"
```
## 修改和提交
```
$ git status        # 查看状态
$ git diff          # 查看变更内容
$ git add .         # 跟踪所有改动过的文件
$ git add <file>    # 跟踪指定的文件
$ git mv <old> <new># 文件改名
$ git rm <file>     # 删除文件
$ git rm --cached <file> # 停止跟踪文件但不删除
$ git commit -m "commit message" # 修改所有更新过的文件
$ git commit --amend # 修改最后一次提交
```
## 查看提交历史
```
$ git log           # 查看提交历史
$ git log -p <file> # 查看指定文件的提交历史
$ git blame <file>  # 以列表的方式查看指定文件的提交历史
```

## 撤销操作
```js
# 恢复暂存区的指定文件到工作区
$ git checkout [file]

# 恢复暂存区当前目录的所有文件到工作区
$ git checkout .

# 恢复工作区到指定 commit
$ git checkout [commit]

# 重置暂存区的指定文件，与上一次 commit 保持一致，但工作区不变
$ git reset [file]

# 重置当前分支的指针为指定 commit，同时重置暂存区，但工作区不变
$ git reset [commit]

# 重置当前分支的HEAD为指定 commit，同时重置暂存区和工作区，与指定 commit 一致
$ git reset --hard [commit]
```
## 分支与标签
```
$ git branch        # 显示所有本地分支
$ git checkout <branch/tag> # 切换到指定分支活标签
$ git branch <new-branch>   # 创建新分支
$ git branch -d <branch>    # 删除本地分支
$ git tag                   # 列出本地标签
$ git tag <tagname>         # 基于最新提交创建标签
$ git tag -d <tagname>      # 删除标签
```
## 合并与衍合
```
$ git merge <branch>        # 合并指定分支到当前分支
$ git rebase <branch>       # 衍合指定分支到当前分支
```
## 远程操作
```
$ git remote -v             # 查看远程版本库信息
$ git remove show <remote>  # 查看指定远程版本库信息
$ git remote add <remote> <url> # 添加远程版本库
$ git fetch <remote>        # 从远程库获取代码
$ git pull <remote> <branch># 下载代码及快速合并
$ git push <remote> <branch># 上传代码及快速合并
$ git push <remote> :<branch/tag-name>  # 删除远程分支活标签
$ git push --tags           # 上传所有标签
```

## 创建一个新仓库
```
git clone git@gitlab.iwhalecloud.com:monitor/nms-arm.git
cd nms-arm
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master
```
## 推送现有文件夹至remote repository

```
cd existing_folder
git init
git remote add origin git@gitlab.iwhalecloud.com:monitor/nms-arm.git
git add .
git commit -m "Initial commit"
git push -u origin master
```



## github 创建仓库引导命令
create a new repository on the command line
```
echo "# blog" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/username/blog.git
git push -u origin main
```

push an existing repository from the command line
```
git remote add origin https://github.com/username/blog.git
git branch -M main
git push -u origin main
```
