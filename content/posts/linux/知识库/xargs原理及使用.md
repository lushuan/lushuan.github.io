---
title: "xargs原理及使用"
subtitle: ""
date: 2021-02-28T12:06:37+08:00
lastmod: 2022-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Linux"]
tags: ["Linux"]
slug: "xargs-guide"
featuredImage: "/images/scenery/cover-20.jpg"
summary: "介绍 xargs 命令的原理与参数用法：把标准输入转换为命令行参数，解决 ls、rm 等命令不支持管道传参的问题，详解 -0、-I、-n、-d、-p 等常用选项。"
---

很多 Linux 命令（如 ls、rm）从命令行参数而非标准输入接受输入，xargs 就是为此设计的"参数转换器"，把 stdin 的输入按分隔符切分后作为参数调用后续命令。本文讲解 xargs 的原理，并结合 find 演示 -print0/-0、-I {} 占位符、-n、-d、-p 等常用参数的用法与示例。

<!--more-->

## 一、简介
之所以能用到这个命令，是由于很多 Linux 命令不支持用管道传递参数，xargs 可以理解为参数转换器，例如

```shell
#错误指令
find /sbin -perm +700 | ls -l
#正确指令
find /sbin -perm +700 |xargs ls -l   
```
通常 Linux 命令可以用|首尾相连，上一个命令的 stdout 连接到下一个命令的 stdin。但是有些命令，比如 ls、rm 等，是从命令行参数接受输入的。这时候如果想把上一个命令的输出传给它们，就不好办了。所以就有了 xargs。

简单而言，xargs 可以把从 stdin 接受到的输入，按空白符或回车符分隔开，然后依次作为参数去调用 xargs 后面的命令

xargs 默认的分隔符：空格 或 回车

## 二、参数使用
想把所有.jpg 文件删除，当然你可以 rm *.jpg，但是如果要递归操作所有子目录下的文件呢？

可以这样：

- `find . -name "*.jpg" | xargs rm`
> 这样，所有被 find 找到的文件名，都会作为参数来调用 rm 命令了
对于大多数情况，这一行命令没有问题，但是如果有些文件名中包含空格，就会有问题了。
xargs 默认以空白符分隔接受到的输入，所以一个含有空格的文件名会被当做多个参数，分别传给 rm。所以在处理文件名这类命令时，通常要这样  　　
- `find . -name "*.jpg" -print0 | xargs -0 rm`
> 这里的 -print0 是告诉 find 命令，在每个输出后面以'\0'作为结束。-0 是告诉 xargs，使用'\0'来分隔输入，而不是空白符。这样就避免出现问题了。
下面再考虑另一种情况，假设不是删除，而是想把符合要求的文件名都添加上后缀.bak 怎么办？这时候需要这样
- `find . -name "*.jpg" -print0 | xargs -0 -I {} mv {} {}.bak`
> 常用命令 其中的-I {}（initial arguments）是告诉 xargs，后面的命令中，用{}表示占位符，将会被实际的参数替代。这样就行了。
其他有用的参数还有： 
-n 用于指定每次传递几个参数 
-d 用于指定切分输入内容时，具体的分隔符 

- `-a file 从文件中读入作为sdtin`
```shell
 $ cat 1.txt 
 aaa  bbb ccc ddd
 a    b
 $ xargs -a 1.txt echo
 aaa bbb ccc ddd a b
```
- `-e flag `
> 注意有的时候可能会是-E，flag 必须是一个以空格分隔的标志，当 xargs 分析到含有 flag 这个标志的时候就停止
```shell
$ xargs -E 'ddd'  -a 1.txt echo
aaa bbb ccc
$ cat 1.txt |xargs -E 'ddd' echo
aaa bbb ccc
```
- `-n num `
> 后面加次数，表示命令在执行的时候一次用的 argument 的个数，默认是用所有的。
```shell
$ cat 1.txt |xargs -n 2 echo
 aaa bbb
 ccc ddd
 a b
```
- `-p `
> 操作具有可交互性，每次执行 command 都交互式提示用户选择，当每次执行一个 argument 的时候询问一次用户
```shell
$ cat 1.txt |xargs -p echo
echo aaa bbb ccc ddd a b ?...y
aaa bbb ccc ddd a b
$ cat 1.txt |xargs -p echo
echo aaa bbb ccc ddd a b ?...n
```
- `-t `
> 表示先打印命令，然后再执行
```shell
$ cat 1.txt |xargs -t echo
echo aaa bbb ccc ddd a b 
aaa bbb ccc ddd a b
```
- 占位
> -i 或者是-I，这得看 Linux 支持了，将 xargs 的每项名称，一般是一行一行赋值给{}，可以用{}代替。小写的-i 带参数时和大写的-I 是一模一样的，小写的-i 可以不带参数，这时候相当于大写的-I {}。 
> 不过手册里面不建议使用小写的-i，可能会有什么问题 xargs 加上-I（这里手册建议使用大写的-I）后就可以用 {}表示管道传过来的参数放到该位置
```shell
$ ls
1.txt  2.txt  3.txt  log.xml
$ ls *.txt | xargs -t -i mv {} {}.bak
mv 1.txt 1.txt.bak 
mv 2.txt 2.txt.bak 
mv 3.txt 3.txt.bak 
$ ls
1.txt.bak  2.txt.bak  3.txt.bak  log.xml
```
- 空参数传递
> -r  no-run-if-empty 如果没有要处理的参数传递给 xargs xargs 默认是带空参数运行一次，如果你希望无参数时，停止 xargs，直接退出，使用 -r 选项即可，其可以防止 xargs 后面命令带空参数运行报错
```shell
$ echo ""|xargs -t mv
mv 
mv: missing file operand
Try `mv --help' for more information.
$ echo ""|xargs -t -r mv         #直接退出
-s num xargs后面那个命令的最大命令行字符数(含空格) 
```
-  `-L `
> 从标准输入一次读取 num 行送给 Command 命令，-l 和-L 功能一样
```shell
$ cat 1.txt.bak 
aaa bbb ccc ddd
a b
ccc
dsds
$ cat 1.txt.bak |xargs  -L 4 echo
aaa bbb ccc ddd a b ccc dsds
$ cat 1.txt.bak |xargs  -L 1 echo
aaa bbb ccc ddd
a b
ccc
dsds
-d delim 分隔符，默认的xargs分隔符是回车，argument的分隔符是空格，这里修改的是xargs的分隔符

$ cat 1.txt.bak 
aaa@ bbb ccc@ ddd
a b

$ cat 1.txt.bak |xargs  -d '@' echo
aaa  bbb ccc  ddd
a b
```

## 三、 xargs 结合 find 使用
举例

```shell
$find . -name "install.log" -print | cat
./install.log                                                 #显示从管道传来的内容，仅仅作为字符串来处理
$find . -name "install.log" -print | xargs cat
aaaaaa                                                      #将管道传来的内容作为文件，交给cat执行。也就是说，该命令执行的是如果存在install.log，那么就打印出这个文件的内容。
来看看xargs命令是如何同find命令一起使用的，并给出一些例子。

1、在当前目录下查找所有用户具有读、写和执行权限的文件，并收回相应的写权限：
# find . -perm -7 -print | xargs chmod o-w
2、查找系统中的每一个普通文件，然后使用xargs命令来测试它们分别属于哪类文件
# find . -type f -print | xargs file
./liyao: empty

3、尝试用rm 删除太多的文件，你可能得到一个错误信息：/bin/rm Argument list too long. 用xargs 去避免这个问题
$find ~ -name ‘*.log’ -print0 | xargs -i -0 rm -f {}

4、查找所有的jpg 文件，并且压缩它
# find / -name *.jpg -type f -print | xargs tar -cvzf images.tar.gz
5、拷贝所有的图片文件到一个外部的硬盘驱动 
# ls *.jpg | xargs -n1 -i cp {} /external-hard-drive/directory
```
## 四、总结  

|命令|解释|
|---|---|
|echo -n "hello#word" &#124;xargs -d "#"|使用#号进行拆分|
|echo "hello word"&#124;xargs -n 1|一次传递一个参数|
|echo -n "hello#word"&#124;xargs -d "#" -t|直接输出打印命令无用户交互|
|echo -n "hello#word"&#124;xargs -d "#" -p|用户交互选择是否打印命令并且输出打印命令|
|echo -n "hello#word"&#124;xargs -I {} echo {}|占位符 initial arguments|
|echo -n ""&#124;xargs -r  echo |小写 r（no run if empty），如果传递的参数为空则不执行|