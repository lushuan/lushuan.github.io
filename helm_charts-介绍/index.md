# Helm Charts 介绍

Helm 使用一种名为 charts 的包格式，一个 chart 是描述一组相关的 Kubernetes 资源的文件集合，单个 chart 可能用于部署简单的应用，比如 memcached pod，或者复杂的应用，比如一个带有 HTTP 服务、数据库、缓存等等功能的完整 web 应用程序。

Charts 是创建在特定目录下面的文件集合，然后可以将它们打包到一个版本化的存档中来部署。接下来我们就来看看使用 Helm 构建 charts 的一些基本方法。

## 文件结构
chart 被组织为一个目录中的文件集合，目录名称就是 chart 的名称（不包含版本信息），下面是一个 WordPress 的 chart，会被存储在 wordpress/ 目录下面，基本结构如下所示：
```
wordpress/
  Chart.yaml          # 包含当前 chart 信息的 YAML 文件
  LICENSE             # 可选：包含 chart 的 license 的文本文件
  README.md           # 可选：一个可读性高的 README 文件
  values.yaml         # 当前 chart 的默认配置 values
  values.schema.json  # 可选: 一个作用在 values.yaml 文件上的 JSON 模式
  charts/             # 包含该 chart 依赖的所有 chart 的目录
  crds/               # Custom Resource Definitions
  templates/          # 模板目录，与 values 结合使用时，将渲染生成 Kubernetes 资源清单文件
  templates/NOTES.txt # 可选: 包含简短使用使用的文本文件
```

## Chart.yaml 文件
对于一个 chart 包来说 Chart.yaml 文件是必须的，它包含下面的这些字段：
```yaml
apiVersion: chart API 版本 (必须)
name: chart 名 (必须)
version: SemVer 2版本 (必须)
kubeVersion: 兼容的 Kubernetes 版本 (可选)
description: 一句话描述 (可选)
type: chart 类型 (可选)
keywords:
  - 当前项目关键字集合 (可选)
home: 当前项目的 URL (可选)
sources:
  - 当前项目源码 URL (可选)
dependencies: # chart 依赖列表 (可选)
  - name: chart 名称 (nginx)
    version: chart 版本 ("1.2.3")
    repository: 仓库地址 ("https://example.com/charts")
maintainers: # (可选)
  - name: 维护者名字 (对每个 maintainer 是必须的)
    email: 维护者的 email (可选)
    url: 维护者 URL (可选)
icon: chart 的 SVG 或者 PNG 图标 URL (可选).
appVersion: 包含的应用程序版本 (可选). 不需要 SemVer 版本
deprecated: chart 是否已被弃用 (可选, boolean)
```

## TEMPLATES 和 VALUES
Helm Chart 模板是用 Go template 语言 进行编写的，另外还额外增加了(【Sprig】](https://github.com/Masterminds/sprig)库中的50个左右的附加模板函数和一些其他[专用函数](https://helm.sh/docs/howto/charts_tips_and_tricks/)。

所有模板文件都存储在 chart 的 templates/ 目录下面，当 Helm 渲染 charts 的时候，它将通过模板引擎传递该目录中的每个文件。模板的 Values 可以通过两种方式提供：

- Chart 开发人员可以在 chart 内部提供一个名为 values.yaml 的文件，该文件可以包含默认的 values 值内容。
- Chart 用户可以提供包含 values 值的 YAML 文件，可以在命令行中通过 helm install 来指定该文件

当用户提供自定义 values 值的时候，这些值将覆盖 chart 中 values.yaml 文件中的相应的值。





