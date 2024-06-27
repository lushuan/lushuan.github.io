---
title: "shell 注释"
subtitle: ""
date: 2020-04-21T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Shell"]
tags: ["Shell"]
---
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
