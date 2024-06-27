# shell 变量

给变量赋值，使用等号即可，但是等号两边千万不要有空格，等号右边有空格的字符串也必须用引号引起来

## 变量定义
```shell
para1="hello world" #字符串直接赋给变量para1
#unset用于取消变量
unset para1
```
定义变量类型：  
declare 或 typeset
- -r 只读(readonly一样)
- -i 整形
- -a 数组
- -f 函数
- -x export

`declare -i n=0`

## 系统变量
|command | 释义|
|---|---|
|$0   |  脚本启动名(包括路径) | 
|$n   |  第n个参数,n=1,2,…9 | 
|$*   |  所有参数列表(不包括脚本本身) | 
|$@   |  所有参数列表(独立字符串) | 
|$#   |  参数个数(不包括脚本本身) | 
|$$   |  当前程式的PID | 
|$!   |  执行上一个指令的PID | 
|$?   |  执行上一个指令的返回值 | 

## 变量引用技巧
|command | 释义|
|---|---|
|${name:+value}        | 如果设置了name,就把value显示,未设置则为空 | 
|${name:-value}        | 如果设置了name,就显示它,未设置就显示value | 
|${name:?value}        | 未设置提示用户错误信息value  | 
|${name:=value}        | 如未设置就把value设置并显示<写入本地中> | 
|${#A}                 | 可得到变量中字节数 | 
|${A:4:9}              | 取变量中第4位到后面9位 | 
|${A:(-1)}             | 倒叙取最后一个字符 | 
|${A/www/http}         | 取变量并且替换每行第一个关键字 | 
|${A//www/http}        | 取变量并且全部替换每行关键字 | 

定义一个变量：file=/dir1/dir2/dir3/my.file.txt

|command | 释义|
|---|---|
| ${file#*/}     | 去掉第一条 / 及其左边的字串：dir1/dir2/dir3/my.file.txt | 
| ${file##*/}    | 去掉最后一条 / 及其左边的字串：my.file.txt | 
| ${file#*.}     | 去掉第一个 .  及其左边的字串：file.txt | 
| ${file##*.}    | 去掉最后一个 .  及其左边的字串：txt | 
| ${file%/*}     | 去掉最后条 / 及其右边的字串：/dir1/dir2/dir3 | 
| ${file%%/*}    | 去掉第一条 / 及其右边的字串：(空值) | 
| ${file%.*}     | 去掉最后一个 .  及其右边的字串：/dir1/dir2/dir3/my.file | 
| ${file%%.*}    | 去掉第一个 .  及其右边的字串：/dir1/dir2/dir3/my | 

说明            
- \# 是去掉左边(在键盘上 # 在 $ 之左边)
- % 是去掉右边(在键盘上 % 在 $ 之右边)
- 单一符号是最小匹配﹔两个符号是最大匹配

## 操作变量
```shell
echo "para1 is $para1"
#将会输出 para1 is hello world

# or
echo "para1 is ${para1}!"
#将会输出 para1 is hello world!
```
### 变量替代
```shell
foo="I'm a cat."
echo ${foo/cat/dog}  # 打印  "I'm a dog."
```

使用两个反斜线进行全力替换
```shell
foo="I'm a cat, and she's cat."
echo ${foo/cat/dog}   # 打印 "I'm a dog, and she's a cat."
echo ${foo//cat/dog}  # 打印 "I'm a dog, and she's a dog."
```
打印时不会修改变量
```shell
foo="hello" 
echo ${foo/hello/goodbye}  # 打印 "goodbye"
echo $foo                  # 仍然打印 "hello"
```

### 变量匹配删除
没有替换的字符串则直接删除
```shell
foo="I like meatballs."
echo ${foo/balls}       # 打印 I like meat.
```
${name#pattern}操作删除${name}匹配模式的最短前缀，而##删除最长前缀：
```shell
minipath="/usr/bin:/bin:/sbin"
echo ${minipath#/usr}           # 打印 /bin:/bin:/sbin
echo ${minipath#*/bin}          # 打印 :/bin:/sbin
echo ${minipath##*/bin}         # 打印 :/sbin ,就是*/bin 匹配的/bin及之前的字符串一并删除
```
运算符%是相同的，只不过匹配的是后缀而不是前缀：
```shell
minipath="/usr/bin:/bin:/sbin"
echo ${minipath%/usr*}           # 打印 nothing
echo ${minipath%/bin*}           # 打印 /usr/bin:
echo ${minipath%%/bin*}          # 打印 /usr ,就是%/bin 匹配的/bin及之后的字符串一并删除
```
分片切割
```shell
string="I'm a fan of dogs."
echo ${string:6:3}           # 打印 fan
```

## 是否存在默认值
```shell
username=shuan_lu
unset username
echo ${username-default}        # 打印 default

username=admin
echo ${username-default}        # 打印 admin
```
对于测试是否设置了变量的操作，可以通过添加冒号（":"）来强制检查变量是否已设置且是否为空：
```shell
foo=""
bar=""

echo ${foo-123}   # 打印 nothing
echo ${bar:-456}  # 打印 456
```
运算符=（或:=）类似于运算符-，只是如果变量没有值，它也会设置变量：
```shell
unset cache
echo ${cache:=1024}   # 打印 1024
echo $cache           # 打印 1024

echo ${cache:=2048}   # 打印 1024
echo $cache           # 打印 1024
```

## 单引号和双引号
- shell 会忽略单引号中所有的特殊字符，其中的所有内容都会被当作一个元素
- 双引号几乎与单引号相似。这里之所以说“几乎”是因为他们也会忽略所有特殊字符，除了：
  - 美元符号：$
  - 反引号：`
  - 反斜杠：\

```shell
world=Earth
foo='Hello, $world!'
bar="Hello, $world"
echo $foo            # 打印 Hello, $world!
echo $bar            # 打印 Hello, Earth
```
## 总结
|command | 释义|
|---|---|
|A="a b c def"           |  将字符串复制给变量 | 
|A=`cmd`                 |  将命令结果赋给变量 | 
|A=$(cmd)                |  将命令结果赋给变量 | 
|eval a=\$$a             |  间接调用 | 
|i=2&&echo $((i+3))      |  计算后打印新变量结果 | 
|i=2&&echo $[i+3]        |  计算后打印新变量结果 | 
|a=$((2>6?5:8))          |  判断两个值满足条件的赋值给变量,类似三目运算符 | 
|$1  $2  $*              |  位置参数 *代表所有 | 
|env                     |  查看环境变量 | 
|env | grep "name"       |  查看定义的环境变量 | 
|set                     |  查看环境变量和本地变量 | 
|read name               |  输入变量 | 
|readonly name           |  把name这个变量设置为只读变量,不允许再次设置 | 
|readonly                |  查看系统存在的只读文件 | 
|export name             |  变量name由本地升为环境 | 
|export name="RedHat"    |  直接定义name为环境变量 | 
|export Stat$nu=2222     |  变量引用变量赋值 | 
|unset name              |  变量清除 | 
|export -n name          |  去掉只读变量 | 
|shift                   |  用于移动位置变量,调整位置变量,使$3的值赋给$2.$2的值赋予$1 | 
|name + 0                |  将字符串转换为数字 | 
|number " "              |  将数字转换成字符串 | 
|a='ee';b='a';echo ${!b} |  间接引用name变量的值 | 
|: ${a="cc"}             |  如果a有值则不改变,如果a无值则赋值a变量为cc | 

