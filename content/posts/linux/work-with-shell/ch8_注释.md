---
title: "shell 注释"
subtitle: ""
date: 2020-04-21T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Shell"]
tags: ["Shell"]
slug: "shell-ch8-comments"
summary: "介绍 shell 中单行注释（#）以及两种多行注释写法：:'...' 和 :<<EOF...EOF。"
---

shell 使用 # 进行单行注释。多行注释有两种常见写法：利用 :'...' 空命令加字符串，或使用 :<<EOF...EOF  here document 的方式。

<!--more-->

shell通过#来注释一行内容，前面我们已经看到过了
```shell
#!/bin/bash
# 这是一行注释
:'
这是
多行
注释
'
ls

:<<EOF
这也可以
达到
多行注释
的目的
EOF
```
