---
title: "文本三剑客之 grep"
subtitle: ""
date: 2021-06-20T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["linux"]
tags: ["linux文本三剑客"]
---
## 为什么使用Linux grep命令?
Grep是一个非常有用的命令行工具，可以用来在文件中查找指定的文本模式。通过使用grep命令，你可以快速定位和提取包含特定模式的行，从而加快查找和处理文本数据的效率。这是一个在日常工作中经常需要使用的任务，因此grep命令在Linux中非常流行。

## Linux grep命令是什么？
Grep是一个在Linux和其他类Unix系统上可用的命令行实用程序，用于搜索和匹配文本。它的名字来自于全局正则表达式（global regular expression print），它的主要功能是根据给定的模式搜索文件中的文本，并打印匹配的行。grep命令支持使用简单的文本模式或正则表达式进行搜索，并且可以通过命令选项进行进一步的定制。

## 如何使用Linux grep命令？
要使用grep命令，在命令行中输入grep，后跟要搜索的模式和要搜索的文件的路径。下面是一些常用的grep命令选项和示例：

示例列表 

|命令|解释|
|---|---|
| grep -i "pattern" file.txt|忽略大小写进行搜索|
| grep -r "pattern" directory/|递归搜索子目录 |
| grep -E "pattern" file.txt|使用正则表达式进行搜索 |
| grep -A 2 "pattern" file.txt|显示匹配模式之前或之后的文本行 |
| grep -C 2 "pattern" file.txt|输出匹配模式的上下文行 |
| grep -i "pattern" file.txt|忽略大小写 |
|  grep -w "pattern" file.txt|精确匹配 |
|  grep -e hello -e world file.txt|多个关键词匹配 |
| grep -v "pattern" file.txt|反向查找，不包含某个关键词的行 |
| grep -lr "pattern" file.txt|递归匹配哪些文件名包含匹配的关键词|
| grep -o "pattern" file.txt|只输出匹配的内容|

![grep](/images/linux/grep.jpg "grep")

## 最佳实践
### 搜索文件中最后匹配的一个关键词的上下200行
```bash
$ tac file.txt | grep -m 1 "/enforcementGroupInfo/editPassword" -B 200 -A 200 | tac > /tmp/1.txt
# or
$ tac file.txt | grep -m 1 "/enforcementGroupInfo/editPassword" -C 200 | tac > /tmp/1.txt
```

#### 命令解释：
1. `tac file.txt` - 反向读取文件内容（从最后一行开始）
2. `grep -m 1` - 只匹配第一个出现的模式（即原文件最后一个匹配项）
3. `-B 200 -A 200` - 显示匹配行前后的200行内容
4. `-C 200 ` - 输出匹配模式的上下文行
5. `tac` - 再次反向输出，恢复原始行顺序
6. `> /tmp/1.txt` - 将结果重定向到目标文件

#### 备选方案（适用于大文件）
如果文件非常大，可以使用更节省内存的方法：
```bash
# 获取最后一个匹配行号
line=$(grep -n "/enforcementGroupInfo/editPassword" file.txt | tail -1 | cut -d: -f1)

# 提取该行上下200行内容
[ -n "$line" ] && sed -n "$((line>200 ? line-200 : 1)),$((line+200))p" file.txt > /tmp/1.txt
```
#### 验证结果
检查输出文件内容是否正确：
```bash
wc -l /tmp/1.txt  # 应该最多401行（200+1+200）
tail -n 10 /tmp/1.txt | grep "/enforcementGroupInfo/editPassword"  # 确认包含关键词
```
#### 注意事项
1. 如果文件中没有匹配项，`/tmp/1.txt` 会是空文件
2. 如果匹配项在文件开头，前200行会从第1行开始
3. 如果匹配项在文件结尾，后200行会到文件末尾结束
4. 对于二进制文件，添加 `-a` 选项：`grep -a`


## 参考
1. [是真的很详细了！Linux中的Grep命令使用实例](https://cloud.tencent.com/developer/article/1554542)