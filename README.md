以下是为你的项目量身定制的 README.md，可直接替换仓库中的现有文件：

---

# 🌐 lushuan.github.io

[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-live-brightgreen?logo=github)](https://blog.dazhongma.top/)
[![Hugo](https://img.shields.io/badge/Hugo-LoveIt-FF4088?logo=hugo)](https://hugo-theme-loveit.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)

> 个人技术博客源码，基于 **Hugo + LoveIt** 主题构建，通过 GitHub Actions 自动部署至 GitHub Pages。

🔗 **在线访问：**

## 📁 项目结构

```
lushuan.github.io/
├── archetypes/default.md   # 文章模板
├── assets/                 # CSS / 图片等资源（Hugo Pipes 处理）
│   ├── css/
│   └── images/
├── config.toml             # 站点全局配置
├── content/                # 文章内容
│   ├── friends/            # 友链页
│   └── posts/              # 博客文章
├── layouts/partials/       # 自定义局部模板（覆盖主题）
├── static/                 # 静态资源（直接复制到 public/）
│   ├── CNAME               # 自定义域名绑定
│   └── images/
├── themes/LoveIt/          # LoveIt 主题
└── LICENSE
```

## 🚀 快速开始

### 环境要求

| 工具 | 最低版本 | 说明 |
| :--- | :--- | :--- |
| Hugo | ≥ 0.126.0 | 需 Extended 版本 |
| Git | ≥ 2.x | 含 submodule 支持 |
| Node.js | ≥ 18 | LoveIt 部分功能依赖 |

### 本地运行

```bash
# 1. 克隆仓库（含主题子模块）
git clone --recurse-submodules https://github.com/lushuan/lushuan.github.io.git
cd lushuan.github.io

# 2. 启动开发服务器
hugo server -D --navigateToChanged
```

访问 `http://localhost:1313` 即可预览。

## ✍️ 写作与发布

```bash
# 新建文章
hugo new posts/my-post.md

# 提交并推送 → 自动触发 CI 构建 & 部署
git add .
git commit -m "feat: add new post"
git push origin main
```

推送后 GitHub Actions 将自动完成构建并部署到 GitHub Pages，无需手动操作。

## ⚙️ CI/CD 与缓存

本项目使用 GitHub Actions 进行自动化部署，并配置了 `actions/cache` 加速构建：

-   **Hugo Modules 缓存**：基于 `go.sum` + `config.toml` 哈希
-   **npm 依赖缓存**：基于 `package-lock.json` 哈希
-   缓存命中时构建时间从 ~2 min 缩短至 ~30s

详细工作流配置见 `.github/workflows/` 目录。

## 📄 License

本项目采用 [MIT License](./LICENSE) 开源。

博客文章内容除特别注明外，均采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议。

