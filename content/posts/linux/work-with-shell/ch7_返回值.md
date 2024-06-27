---
title: "shell 返回值"
subtitle: ""
date: 2020-04-22T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Shell"]
tags: ["Shell"]
---
通常函数的return返回值只支持0-255，因此想要获得返回值，可以通过下面的方式。

```shell
function myfunc() {
 local myresult='some value'
 echo $myresult
}
val=$(myfunc) #val的值为some value

```

通过return的方式适用于判断函数的执行是否成功：
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