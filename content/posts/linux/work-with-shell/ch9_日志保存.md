---
title: "shell 日志保存"
subtitle: ""
date: 2020-04-20T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Shell"]
tags: ["Shell"]
slug: "shell-ch9-logging"
summary: "介绍 shell 脚本日志保存的三种重定向方式：> 保存标准输出、> file 2>&1 同时保存标准错误、|tee 边输出边保存。"
---

脚本执行后通常需要记录日志，最常用的方法是重定向。本文介绍三种方式：`> log.dat` 仅保存标准输出，`> log.dat 2>&1` 将标准输出和标准错误一并保存，`|tee log.dat` 在保存日志的同时输出到控制台。

<!--more-->

脚本执行后免不了要记录日志，最常用的方法就是重定向。以下面的脚本为例：

```shell
#!/bin/bash
#test.sh
lll #这个命令是没有的，因此会报错
date

```
## 一、方式一
将标准输出保存到文件中，打印标准错误：
```shell
./test.sh > log.dat
```
这种情况下，如果命令执行出错，错误将会打印到控制台。所以如果你在程序中调用，这样将不会将错误信息保存在日志中。
## 二、方式二 
```shell
./test.sh > log.dat 2>&1
```

标准输出和标准错误都保存到日志文件中

这里的 2>&1 是什么意思？  
`表明将./test.sh的输出重定向到log.txt文件中，同时将标准错误也重定向到log.txt文件中。`

## 三、方式三
保存日志文件的同时，也输出到控制台
```shell
./test.sh |tee log.dat
```