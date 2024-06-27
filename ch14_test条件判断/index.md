# shell test 条件判断


## 介绍
test 是 Shell 内置命令，用来检测某个条件是否成立。test 通常和 if 语句一起使用，并且大部分 if 语句都依赖 test。

test 命令有很多选项，可以进行数值、字符串和文件三个方面的检测。

Shell test 命令的用法为：


```shell
test expression
```


当 test 判断 expression 成立时，退出状态为 0，否则为非 0 值。

test 命令也可以简写为[]，它的用法为：


```shell
[ expression ]
```

注意[]和expression之间的空格，这两个空格是必须的，否则会导致语法错误。[]的写法更加简洁，比 test 使用频率高。

test 和 [] 是等价的，后续我们会交替使用 test 和 []，以让读者尽快熟悉。


## 字符串操作
|command | 释义|
|---|---|
|-z "$str1"           | str1是否为空字符串 |
|-n "$str1"           | str1是否不是空字符串 |
|"$str1" == "$str2"   | str1是否与str2相等 |
|"$str1" != "$str2"   | str1是否与str2不等 |
|"$str1" =~ "str2"    | str1是否包含str2 |

## 整数操作
|command | 释义|
|---|---|
| -eq | 两数是否相等 | 
| -ne | 两数是否不等 | 
| -gt | 前者是否大于后者（greater then） | 
| -lt | 前面是否小于后者（less than） | 
| -ge | 前者是否大于等于后者（greater then or equal） | 
| -le | 前者是否小于等于后者（less than or equal） | 

## 文件操作
|command | 释义|
|---|---|
| -a     | 并且，两条件为真 | 
| -b     | 是否块文件 | 
| -p     | 文件是否为一个命名管道 | 
| -c     | 是否字符文件 | 
| -r     | 文件是否可读 | 
| -d     | 是否一个目录 | 
| -s     | 文件的长度是否不为零 | 
| -e     | 文件是否存在 | 
| -S     | 是否为套接字文件 | 
| -f     | 是否普通文件 | 
| -x     | 文件是否可执行，则为真 | 
| -g     | 是否设置了文件的 SGID 位 | 
| -u     | 是否设置了文件的 SUID 位 | 
| -G     | 文件是否存在且归该组所有 | 
| -w     | 文件是否可写，则为真 | 
| -k     | 文件是否设置了的粘贴位 | 
| -o     | 或，一个条件为真 | 
| -O     | 文件是否存在且归该用户所有 | 
| !      | 取反 | 

## 示例
|command | 释义|
|---|---|
| test 10 -lt 5       | 判断大小 |
| echo $?             | 查看上句test命令返回状态  # 结果0为真,1为假 |
| test -n "hello"     | 判断字符串长度是否为0 |
| [ $? -eq 0 ] && echo "success" &#124;&#124; exit　 |　　# 判断成功提示,失败则退出 |


## 判断
###  整数判断
- -eq 两数是否相等
- -ne 两数是否不等
- -gt 前者是否大于后者（greater then）
- -lt 前面是否小于后者（less than）
- -ge 前者是否大于等于后者（greater then or equal）
- -le 前者是否小于等于后者（less than or equal）

###  字符串判断str1 exp str2：
- -z "$str1" str1是否为空字符串
- -n "$str1" str1是否不是空字符串
- "$str1" == "$str2" str1是否与str2相等
- "$str1" != "$str2" str1是否与str2不等
- "$str1" =~ "str2" str1是否包含str2

###  文件目录判断
- -f $filename 是否为文件
- -e $filename 是否存在
- -d $filename 是否为目录
- -s $filename 文件存在且不为空
- ! -s $filename 文件是否为空

## 流程控制
- break N     #  跳出几层循环
- continue N  #  跳出几层循环，循环次数不变
- continue    #  重新循环次数不变

## 示例
```shell
#!/bin/bash
read age
if test $age -le 2; then
echo "婴儿"
elif test $age -ge 3 && test $age -le 8; then
echo "幼儿"
elif [ $age -ge 9 ] && [ $age -le 17 ]; then
echo "少年"
elif [ $age -ge 18 ] && [ $age -le 25 ]; then
echo "成年"
elif test $age -ge 26 && test $age -le 40; then
echo "青年"
elif test $age -ge 41 && [ $age -le 60 ]; then
echo "中年"
else
echo "老年"
fi
```

## 总结
test 命令有点特别，>、<、\== 只能用来比较字符串，不能用来比较数字，比较数字需要使用 -eq、-gt 等选项；不管是比较字符串还是数字，test 都不支持 >= 和 <=。有经验的程序员需要慢慢习惯 test 命令的这些奇葩用法
