## 📝 一、内容创作

### 新建文章

```bash
# 标准文章（自动套用 archetypes/default.md 模板）
hugo new posts/my-new-post.md

# 指定自定义 archetype
hugo new posts/my-new-post.md --kind post-with-gallery
```

> 💡 LoveIt 主题的 archetype 通常位于 `themes/LoveIt/archetypes/`，可复制到项目根目录 `archetypes/` 下自定义修改。

### 编辑与预览

```bash
# 本地开发服务器（含草稿、热重载）
hugo server -D --navigateToChanged

# 指定端口 & 禁止浏览器自动打开
hugo server -D -p 1314 --noHTTPCache

# 仅预览已发布内容（不含草稿）
hugo server
```

| 常用标志 | 作用 |
| --- | --- |
| `-D` / `--buildDrafts` | 包含 front matter 中 `draft: true` 的文章 |
| `-F` / `--buildFuture` | 包含 `date` 在未来的文章 |
| `-E` / `--buildExpired` | 包含已过期的文章 |
| `--navigateToChanged` | 保存文件后浏览器自动跳转到对应页面 |
| `--disableFastRender` | 禁用快速渲染，用于排查缓存导致的显示异常 |

---

## 🔧 二、构建与调试

### 生产构建

```bash
# 标准生产构建（压缩 + GC）
hugo --gc --minify

# 指定 baseURL（覆盖 config.toml）
hugo --gc --minify --baseURL "https://lushuan.github.io"

# 构建到自定义输出目录
hugo --gc --minify -d ./dist
```

### 诊断与排错

```bash
# 查看 Hugo 环境信息（版本、Go、模块等）
hugo env

# 详细日志输出
hugo -v --gc --minify

# 打印所有配置值（排查配置合并问题）
hugo config

# 列出所有内容及其参数
hugo list all

# 检查未使用的静态资源
hugo --printUnusedTemplates --printPathWarnings
```

---

## 🗂️ 三、依赖与模块管理

```bash
# 初始化 Hugo Modules（首次使用或 go.mod 丢失时）
hugo mod init github.com/lushuan/lushuan.github.io

# 拉取/更新所有模块（含 LoveIt 主题）
hugo mod get -u

# 清理模块缓存
hugo mod clean

# 将模块依赖打包到 _vendor（离线构建 / CI 加速备选）
hugo mod vendor

# npm 依赖（LoveIt 部分功能需要）
npm install        # 安装
npm update         # 更新
```

> ⚠️ 执行 `hugo mod get -u` 后务必提交 `go.sum` 和 `go.mod` 的变更，否则 CI 构建可能因哈希不匹配而失败。

---

## 🚀 四、发布与部署

### Git 工作流（触发 GitHub Actions 自动部署）

```bash
# 日常发布三步曲
git add .
git commit -m "feat: add new post about xxx"
git push origin main
```

推送后 GitHub Actions 自动完成：构建 → 部署 GitHub Pages → 同步 Gitee

### 手动操作（特殊场景）

```bash
# 仅清理 Git 跟踪的临时文件（如误提交的 .hugo_build.lock）
git rm --cached .hugo_build.lock
git commit -m "chore: remove tracked lock file"

# 查看 Actions 缓存状态（确认缓存是否命中）
gh cache list                    # 需安装 GitHub CLI
gh cache delete <cache-id>       # 手动清除失效缓存

# 本地模拟 CI 构建（排查线上构建失败）
act -j build                     # 需安装 nektos/act
```

---

## 🔄 五、Gitee 同步管理

```bash
# 验证 SSH 密钥连通性
ssh -T git@gitee.com

# 手动触发镜像同步（Actions 失败时的应急方案）
# 在项目根目录执行
git push gitee-mirror main:main
```

> 前提：已添加 Gitee 远程仓库 `git remote add gitee-mirror git@gitee.com:lu_shuan/hugolu.git`

---

## 📋 Front Matter 速记（LoveIt 主题）

```yaml
---
title: "文章标题"
date: 2026-08-07T17:00:00+08:00
draft: false
tags: ["Hugo", "CI/CD"]
categories: ["技术笔记"]
featuredImage: "/images/cover.jpg"      # 封面图
summary: "自定义摘要，留空则自动截取"     # SEO & 列表页展示
toc: true                                # 目录开关
comment: true                            # Gitalk 评论开关
license: "CC BY-NC-SA 4.0"              # 文章许可
---
```

---

### 🎯 日常高频组合拳

```bash
# 写新文章 → 预览 → 发布（最常用）
hugo new posts/xxx.md && hugo server -D --navigateToChanged
# 满意后 Ctrl+C 停止服务器
git add . && git commit -m "feat: xxx" && git push origin main
```
