## 文章编写与发布


1. 构建网站
```yaml
hugo
# or
hugo -D 编译静态网站文件
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

## 参考
1. [Hugo的工作原理](https://hugo.aiaide.com/post/hugo%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/#:~:text=%E6%96%87%E7%AB%A0%E5%B0%B1%E6%98%AF%E4%BD%9C%E8%80%85%E9%9C%80%E8%A6%81%E6%92%B0%E5%86%99%E7%9A%84%E5%86%85%E5%AE%B9%2C%20%E4%BB%96%E4%BB%A5markdown%E6%A0%BC%E5%BC%8F%E7%9A%84%E6%96%87%E4%BB%B6%E5%AD%98%E6%94%BE%E5%9C%A8content%E7%9B%AE%E5%BD%95%E4%B8%8B%E9%9D%A2.%20%E6%88%91%E4%BB%AC%E6%97%A2%E5%8F%AF%E4%BB%A5%E9%80%9A%E8%BF%87%E5%91%BD%E4%BB%A4%E8%A1%8C%E7%9A%84%E6%96%B9%E5%BC%8F%E5%88%9B%E5%BB%BA%E6%96%87%E7%AB%A0%20hugo%20new%20about.md%2C%20%E4%B9%9F%E5%8F%AF%E4%BB%A5%E9%80%9A%E8%BF%87%E6%89%8B%E5%B7%A5%E7%9A%84%E6%96%B9%E5%BC%8F%E5%9C%A8content%E5%88%9B%E5%BB%BA.%20%E9%80%9A%E5%B8%B8%E6%88%91%E4%BB%AC%E6%8A%8A%E5%8D%95%E7%8B%AC%E7%9A%84%E6%96%87%E7%AB%A0%E5%86%85%E5%AE%B9%E6%94%BE%E5%9C%A8content%E7%9B%AE%E5%BD%95%E4%B8%8B%E9%9D%A2%2C,%E5%8D%B3%3A%20%E9%A1%B5%E9%9D%A2%20%3D%20%E6%96%87%E7%AB%A0%20%2B%20%E6%A8%A1%E6%9D%BF.%20hugo%E4%BC%9A%E6%A0%B9%E6%8D%AE%E4%B8%80%E5%AE%9A%E7%9A%84%E8%A7%84%E5%88%B6%E5%8E%BB%E5%AF%BB%E6%89%BE%E6%96%87%E7%AB%A0%E5%AF%B9%E5%BA%94%E7%9A%84%E6%A8%A1%E6%9D%BF%E9%A1%B5%E9%9D%A2%2C%20%E4%BB%8E%E8%80%8C%E7%94%9F%E6%88%90%E9%A1%B5%E9%9D%A2.)
2. [使用HUGO搭建个人博客](https://www.jianshu.com/p/4669fb3bf35a) 包含域名购买推荐
3. [Hugo LoveIt主题配置与使用](https://blog.csdn.net/qq_39132095/article/details/117122170)
4. [hugoloveit主题官网](https://hugoloveit.com/zh-cn/theme-documentation-basics/)
5. [hugo官网](https://gohugo.io/)


## 采坑
1. [从Hexo迁移至Hugo以及使用LoveIt主题的踩坑记录](https://cloud.tencent.com/developer/article/1932817)
