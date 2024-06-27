# shell 函数

为了完成某一功能的程序指令（语句）的集合，称为函数。Shell 函数的本质是一段可以重复使用的脚本代码，这段代码被提前编写好了，放在了指定的位置，使用时直接调取即可

在程序中，编写函数的主要目的是将一个需要很多行代码的复杂问题分解为一系列简单的任务来解决，而且，同一个任务（函数）可以被多次调用，有助于代码重用。

## 定义函数

```shell
function myfunc() 
{
  echo "hello world $1"
}

function myfunc
{
  echo "hello world $1"
}
```

或者
```shell
myfunc() 
{
  echo "hello world $1"
}
```
说明
- 可以省略 function 关键词
- 如果写了 function 关键字，也可以省略函数名后面的小括号 
## 函数参数及调用
在 Shell 中，我们定义 函数 时，不像 C 语言、 C++、 Python、 Java 和 Golang 那样，需要传递参数，Shell 中的函数在定义时不能指明参数，但是在调用时却可以传递参数。

函数参数是 Shell 位置参数的一种，在函数内部可以使用 $n 来接收，例如，$1 表示第一个参数，$2 表示第二个参数，依次类推。除了 n，还有另外三个比较重要的变量，# 可以获取传递的参数的个数，$@ 或者 $* 可以一次性获取所有的参数。


```shell
#!/bin/bash
function show(){
	echo "Name is: $1"
	echo "Site is: $2"
	echo "Age is: $3"
}
show shuanlu https://lu_shuan.gitee.io/gitbook 109
```


