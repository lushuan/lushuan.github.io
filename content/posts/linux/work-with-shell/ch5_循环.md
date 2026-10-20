---
title: "shell 循环"
subtitle: ""
date: 2020-04-24T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Shell"]
tags: ["Shell"]
slug: "shell-ch5-loops"
featuredImage: "/images/scenery/cover-17.jpg"
summary: "总结 shell 中 for、while、until 三种循环的多种写法与示例，包括遍历参数、C 风格循环、序列区间、条件循环等。"
---

本文汇总 shell 循环的常见写法：for 循环的四种形式（遍历参数、C 风格、{1..5} 序列、带步长区间）及批量编译示例，while 循环的条件控制，以及 until 循环的反向条件用法。

<!--more-->

## 一、for 循环
### for 循环一
```shell
#遍历输出脚本的参数
for i in $@; do
  echo $i
done

```
### for 循环方式二
```shell
for ((i = 0 ; i < 10 ; i++)); do
  echo $i
done
```
### for 循环方式三
```shell
for i in {1..5}; do
  echo "Welcome $i"
done

```
### for 循环方式四
```shell
for i in {5..15..3}; do
  echo "number is $i"
done
```
每隔 3 打印一次，即打印 5,8,11,14

示例
```shell
 for f in *.c
 do
   gcc -o ${f%.c} $f
 done
```
## 二、while 循环
```shell
while [ "$ans" != "yes" ]
do
  read -p "please input yes to exit loop:" ans
done
# 只有当ans不是yes时，循环就终止。

# or
num=1
while [ $num -lt 10 ]
do
  echo $num
  ((num=$num+2))
done

```

## 三、until 循环
```shell
num=1
#  当command不为0时循环
until [ $num -gt 10 ]
do
  echo until $num
  ((num=$num+2))
done
```
