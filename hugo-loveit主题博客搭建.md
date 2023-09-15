## 项目目录
```yaml
.
├── config.toml     # 网站的配置信息
├── archetypes      # 存放 .md 文件的模板
├── content         # 存放 .md 文件
├── data            # 存放 Hugo 数据
├── layouts         # 存放布局文件
├── public          # 公共文件夹，用于存放生成的站点文件
├── static          # 存放静态文件，比如图片、CSS、JS
└── themes          # 存放主题

```
### 快速操作
查看版本：hugo version

新建一篇文章,在content目录下创建一个post目录： hugo new post/my-first-blog.md

生成静态文件： hugo -t even

启动服务器： hugo server

正常启动服务后，在浏览器打开 http://localhost:1313/ 看到我们的网站。

## 文章编写与发布


1. 构建网站
```yaml
hugo
# or
hugo -D   #编译静态网站文件
```
会生成一个 public 目录, 其中包含你网站的所有静态内容和资源. 现在可以将其部署在任何 Web 服务器上.

2. 写文档
```yaml
hugo new posts/first/test.md
```

3. 本地启动，编译预览
```yaml
hugo server -t casper -D  # -D draft: true  自动调整为 draft: false

# or 
hugo serve --disableFastRender

```
4. 部署至github和gitee
```yaml
hugo --theme=LoveIt --baseUrl="http://lu_shuan.gitee.io/hugolu/"

```

### 内容文章的元信息
```yaml
---
title: "文章标题"           # 文章标题
author: "作者"              # 文章作者
description : "描述信息"    # 文章描述信息
date: 2015-09-28            # 文章编写日期
lastmod: 2015-04-06         # 文章修改日期
comments: true  # 是否开启Disqus评论功能
share: true  # 是否开启分

tags : [                    # 文章所属标签
    "文章标签1",
    "文章标签2"
]
categories : [              # 文章所属标签
    "文章分类1",
    "文章分类2",
]
keywords : [                # 文章关键词
    "Hugo",
    "static",
    "generator",
]

next: /tutorials/github-pages-blog      # 下一篇博客地址
prev: /tutorials/automated-deployments  # 上一篇博客地址
---
```
在使用hugo server -D预览网站无误后可正式发布网站到域名供大家浏览。将要发布的文章内draft改为false后执行命令

### 开发和生产环境的区别
Current environment is "development". The "comment system", "CDN" and "fingerprint" will be disabled.
当前运行环境是 "development". "评论系统", "CDN" 和 "fingerprint" 不会启用.



### 页面和模板的对应关系
页面和模板的应对关系是根据页面的一系列的属性决定的, 这些属性有: Kind, Output Format, Language, Layout, Type, Section. 他们不是同时起作用, 其中kind, layout, type, section用的比较多.

- kind: 用于确定页面的类型, 单页面使用single.html为默认模板页, 列表页使用list.html为默认模板页, 值不能被修改
- section: 用于确定section tree下面的文章的模板. section tree的结构是由content目录结构生成的, 不能被修改, content目录下的一级目录自动成为root section, 二级及以下的目录, 需要在目录下添加_index.md文件才能成为section tree的一部分. 如果页面不在section tree下section的值为空
- type: 可以在Front Matter中设置, 用户指定模板的类型. 如果没设定type的值, type的值等于section的值 或 等于page(section为空的时候)
- layout: 可以在Front Matter中设置, 用户指定具体的模板名称.

## 注意
themes 文件夹用来存放下载的主题，content｜static｜config.toml 这几个文件是要被主题替换的文件

## Disqus、Gitalk、Valine 这三个评论系统的区别
Disqus、Gitalk和Valine是三种常见的评论系统，它们在功能、特点和实现方式上有一些区别。

1. Disqus:
   - 最早出现的云评论系统之一，拥有较长的历史和广泛的用户基础。
   - 提供强大的社交化功能，支持用户在Disqus网站上进行注册、登录、点赞、关注等操作，可方便分享评论内容至社交媒体。
   - 支持多种语言、自定义样式和广告分成等功能。
   - 在国内，Disqus由于网络限制的原因，可能存在访问困难。

2. Gitalk:
   - 基于GitHub Issue和Preact开发的评论系统，支持使用GitHub账户进行登录和评论等操作。
   - 使用GitHub Issue作为存储评论的后台，方便开发者管理和回复评论。
   - 部署简单，不需要后端支持，只需要在页面中引入相关的脚本即可。
   - 支持语言国际化、Markdown语法、代码语法高亮等功能。
   - 可以在GitHub项目页面进行历史评论的查看和管理。

3. Valine:
   - 一个快速、简洁且轻量级的评论系统，专为个人博客网站设计。
   - 采用LeanCloud作为后端，提供云存储和云函数支持。
   - 支持使用微信、QQ、微博等第三方账户登录评论。
   - 提供防止垃圾评论的字符过滤功能，支持设置敏感词。
   - 可以使用Valine Admin工具进行评论的管理和审核。

总体而言，这三个评论系统都有各自的优势和适用场景。Disqus功能强大，适用于全球范围内的社交化评论需求；Gitalk简单易用，适合开发者使用；Valine轻便高效，适用于个人博客网站。选择适合自己需求的评论系统需要考虑使用成本、功能需求和用户体验等因素。
## 域名
1. [新网域名](https://www.xinnet.com/domain/domainQueryResult.html?prefix=xmddlu&suffix=.cn)
2. [gitee 关闭个人购买域名入口](https://help.gitee.com/services/gitee-pages/pro/)


## 参考
1. [Hugo的工作原理](https://hugo.aiaide.com/post/hugo%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/#:~:text=%E6%96%87%E7%AB%A0%E5%B0%B1%E6%98%AF%E4%BD%9C%E8%80%85%E9%9C%80%E8%A6%81%E6%92%B0%E5%86%99%E7%9A%84%E5%86%85%E5%AE%B9%2C%20%E4%BB%96%E4%BB%A5markdown%E6%A0%BC%E5%BC%8F%E7%9A%84%E6%96%87%E4%BB%B6%E5%AD%98%E6%94%BE%E5%9C%A8content%E7%9B%AE%E5%BD%95%E4%B8%8B%E9%9D%A2.%20%E6%88%91%E4%BB%AC%E6%97%A2%E5%8F%AF%E4%BB%A5%E9%80%9A%E8%BF%87%E5%91%BD%E4%BB%A4%E8%A1%8C%E7%9A%84%E6%96%B9%E5%BC%8F%E5%88%9B%E5%BB%BA%E6%96%87%E7%AB%A0%20hugo%20new%20about.md%2C%20%E4%B9%9F%E5%8F%AF%E4%BB%A5%E9%80%9A%E8%BF%87%E6%89%8B%E5%B7%A5%E7%9A%84%E6%96%B9%E5%BC%8F%E5%9C%A8content%E5%88%9B%E5%BB%BA.%20%E9%80%9A%E5%B8%B8%E6%88%91%E4%BB%AC%E6%8A%8A%E5%8D%95%E7%8B%AC%E7%9A%84%E6%96%87%E7%AB%A0%E5%86%85%E5%AE%B9%E6%94%BE%E5%9C%A8content%E7%9B%AE%E5%BD%95%E4%B8%8B%E9%9D%A2%2C,%E5%8D%B3%3A%20%E9%A1%B5%E9%9D%A2%20%3D%20%E6%96%87%E7%AB%A0%20%2B%20%E6%A8%A1%E6%9D%BF.%20hugo%E4%BC%9A%E6%A0%B9%E6%8D%AE%E4%B8%80%E5%AE%9A%E7%9A%84%E8%A7%84%E5%88%B6%E5%8E%BB%E5%AF%BB%E6%89%BE%E6%96%87%E7%AB%A0%E5%AF%B9%E5%BA%94%E7%9A%84%E6%A8%A1%E6%9D%BF%E9%A1%B5%E9%9D%A2%2C%20%E4%BB%8E%E8%80%8C%E7%94%9F%E6%88%90%E9%A1%B5%E9%9D%A2.)
2. [使用HUGO搭建个人博客](https://www.jianshu.com/p/4669fb3bf35a) 包含域名购买推荐
3. [Hugo LoveIt主题配置与使用](https://blog.csdn.net/qq_39132095/article/details/117122170)
4. [hugoloveit主题官网](https://hugoloveit.com/zh-cn/theme-documentation-basics/)
5. [hugo官网](https://gohugo.io/)
6. [玩遍博客网站，我整理了全套的建站技术栈](https://www.yulisay.com/d/kljqu.html)
7. [Hugo系列(3.3) - LoveIt主题美化与博客功能增强 · 第四章](https://lewky.cn/posts/hugo-3-3/)
8. [【博客写作指南】创意封面设计与选择](https://zhuanlan.zhihu.com/p/652711765)


## 采坑
1. [从Hexo迁移至Hugo以及使用LoveIt主题的踩坑记录](https://cloud.tencent.com/developer/article/1932817)
