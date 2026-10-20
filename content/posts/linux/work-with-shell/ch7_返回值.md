---
title: "shell 返回值"
subtitle: ""
date: 2020-04-22T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Shell"]
tags: ["Shell"]
slug: "shell-ch7-return-values"
summary: "说明 shell 函数返回值的两种方式：用 echo 输出并通过 $() 捕获返回值，用 return（0-255）判断函数执行是否成功。"
---

shell 函数的 return 只支持 0-255 的数值，适合用于判断函数执行是否成功。若需要真正返回数据，应在函数内用 echo 输出结果，再通过 $（） 命令替换捕获。

<!--more-->

通常函数的 return 返回值只支持 0-255，因此想要获得返回值，可以通过下面的方式。

```shell
function myfunc() {
 local myresult='some value'
 echo $myresult
}
val=$(myfunc) #val的值为some value

```
通过 return 的方式适用于判断函数的执行是否成功：
```shell
function myfunc() {
 #do something
 return 0
}

if myfunc;then
 echo "success"
else
 echo "failed"
fi
```