# Python3 快速上手指南


## 介绍
### Python 简介
{{< admonition abstract >}}
Python 是一个高层次的结合了解释性、编译性、互动性和面向对象的脚本语言。
Python 的设计具有很强的可读性，相比其他语言经常使用英文关键字，其他语言的一些标点符号，它具有比其他语言更有特色语法结构。

Python 是一种解释型语言： 这意味着开发过程中没有了编译这个环节。类似于PHP和Perl语言。

Python 是交互式语言： 这意味着，您可以在一个Python提示符，直接互动执行写你的程序。

Python 是面向对象语言: 这意味着Python支持面向对象的风格或代码封装在对象的编程技术。

Python 是初学者的语言：Python 对初级程序员而言，是一种伟大的语言，它支持广泛的应用程序开发，从简单的文字处理到 WWW 浏览器再到游戏。
{{< /admonition >}}


### Python 特点
{{< admonition info >}}
1. 易于学习：Python有相对较少的关键字，结构简单，和一个明确定义的语法，学习起来更加简单。
2. 易于阅读：Python代码定义的更清晰。
3. 易于维护：Python的成功在于它的源代码是相当容易维护的。
4. 一个广泛的标准库：Python的最大的优势之一是丰富的库，跨平台的，在UNIX，Windows和Macintosh兼容很好。
5. 互动模式：互动模式的支持，您可以从终端输入执行代码并获得结果的语言，互动的测试和调试代码片断。
6. 可移植：基于其开放源代码的特性，Python已经被移植（也就是使其工作）到许多平台。
7. 可扩展：如果你需要一段运行很快的关键代码，或者是想要编写一些不愿开放的算法，你可以使用C或C++完成那部分程序，然后从你的Python程序中调用。
8. 数据库：Python提供所有主要的商业数据库的接口。
9. GUI编程：Python支持GUI可以创建和移植到许多系统调用。
10. 可嵌入: 你可以将Python嵌入到C/C++程序，让你的程序的用户获得"脚本化"的能力。
{{< /admonition >}}


## 安装 Python 3
Python最新源码，二进制文档，新闻资讯等可以在Python的官网查看到：

Python官网：https://www.python.org/

你可以在以下链接中下载 Python 的文档，你可以下载 HTML、PDF 和 PostScript 等格式的文档。

Python文档下载地址：https://www.python.org/doc/
### Window 平台安装 Python
以下为在 Window 平台上安装 Python 的简单步骤：

打开 WEB 浏览器访问https://www.python.org/downloads/windows/

## 设置开发环境
### 使用虚拟环境（virtualenv 或 venv）命令行
- venv，Python官方用于创建虚拟环境的工具。

  ```
  cd xxx/xxx/crm
  python3.9 -m venv ddd
  python3.7 -m venv xxxx
  python3.7 -m venv /xxx/xxx/xxx/xx/ppp
  ```

- virtualenv 【推荐】
  ```
  pip install virtualenv

  cd /xxx/xx/
  virtualenv ddd --python=python3.9
  # or
  virtualenv /xxx/xx/ddd --python=python3.9
  ```

- 激活虚拟环境
  - win
    ```
    cd F:\envs\crm\Scripts
    activate
    ```
  - mac
    ```
    source /虚拟环境目录/bin/activate
    ```
###  通过配置编辑器（如 VS Code、PyCharm 等）
通过配置编辑器如Pycharm 创建虚拟环境是一种更方便的方式，原理还是通过命令行，通过界面化操作会直观一些

![virtualenv](/images/python/pycharm-1.png)

## 基础语法
### 变量
变量规则
1. 变量只能包含字母、数字和下划线
2. 变量不可以包含空格，可以使用下划线来分隔单词
3. 不要将python 关键字作为变量
4. 变量名应该简短具有描述性
5. 慎用小写字母i和大写字母O，会被误看出数字1和0

### 控制流（if、for、while）
#### 条件语句
if 条件语句
```
if 判断条件：
    执行语句……
else：
    执行语句……
```
多条件匹配时
```
if 判断条件1:
    执行语句1……
elif 判断条件2:
    执行语句2……
elif 判断条件3:
    执行语句3……
else:
    执行语句4……
```
#### 循环语句
- while 循环语句
```
while 判断条件：
    执行语句……
```
Gif 演示 Python while 语句执行过程

![loop-over-python-list-animation](/images/python/loop-over-python-list-animation.gif)

- for循环语句
```
for iterating_var in sequence:
   statements(s)
```
流程图

![loop-over-python-list-animation](/images/python/python_for_loop.jpg)

实例
```Python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
for letter in 'Python':     # 第一个实例
   print('当前字母 :', letter)
 
fruits = ['banana', 'apple',  'mango']
for fruit in fruits:        # 第二个实例
   print('当前水果 :', fruit)
 
print("Good bye!")
```

通过序列索引迭代
```Python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
fruits = ['banana', 'apple',  'mango']
for index in range(len(fruits)):
   print('当前水果 :', fruits[index])
 
print("Good bye!")
```
### 函数
- 函数定义和调用
```Python
def greet_user(): # 1. 关键字def
    """显示简单的问候语，文档字符串注释，方便生成程序文档""" # 2. 注释
    print("hello")   # 3. 函数体

greet_user()    # 4. 调用
```
#### 参数
- 实参和形参
```Python
def greet_user(username):
	pass
	
username="xiao hua"
greet_user(username) # 形参
greet_user("xiao hua") # 实参
```
- 默认参数
```Python
# 可写函数说明
def printinfo(name, age=35):
    """打印任何传入的字符串"""
    print("Name: ", name)
    print("Age ", age)
    return


# 调用printinfo函数
printinfo(age=50, name="miki")
printinfo(name="miki")
```

### 异常处理
捕捉异常可以使用try/except语句。

try/except语句用来检测try语句块中的错误，从而让except语句捕获异常信息并处理。

如果你不想在异常发生时结束你的程序，只需在try里捕获它。

#### try....except...else
以下为简单的try....except...else的语法：
```
try:
<语句>        #运行别的代码
except <名字>：
<语句>        #如果在try部份引发了'name'异常
except <名字>，<数据>:
<语句>        #如果引发了'name'异常，获得附加的数据
else:
<语句>        #如果没有异常发生
```
try的工作原理是，当开始一个try语句后，python就在当前程序的上下文中作标记，这样当异常出现时就可以回到这里，try子句先执行，接下来会发生什么依赖于执行时是否出现异常。

如果当try后的语句执行时发生异常，python就跳回到try并执行第一个匹配该异常的except子句，异常处理完毕，控制流就通过整个try语句（除非在处理异常时又引发新的异常）。

如果在try后的语句里发生了异常，却没有匹配的except子句，异常将被递交到上层的try，或者到程序的最上层（这样将结束程序，并打印缺省的出错信息）。

如果在try子句执行时没有发生异常，python将执行else语句后的语句（如果有else的话），然后控制流通过整个try语句。

实例

下面是简单的例子，它打开一个文件，在该文件中的内容写入内容，且并未发生异常：

```Python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

try:
    fh = open("testfile", "w")
    fh.write("这是一个测试文件，用于测试异常!!")
except IOError:
    print("Error: 没有找到文件或读取文件失败")
else:
    print("内容写入文件成功")
    fh.close()
```
输出结果
```shell
$ python test.py 
内容写入文件成功
$ cat testfile       # 查看写入的内容
这是一个测试文件，用于测试异常!!
```
#### try-finally 语句
try-finally 语句无论是否发生异常都将执行最后的代码。
```
try:
<语句>
finally:
<语句>    #退出try时总会执行
raise
```
示例
```Python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

try:
    fh = open("testfile", "w")
    fh.write("这是一个测试文件，用于测试异常!!")
finally:
    print("Error: 没有找到文件或读取文件失败")
```
输出
```shell
$ python test.py 
Error: 没有找到文件或读取文件失败
```
## 数据结构
### 列表（List）
列表式什么?

`列表由一系列按特定顺序排列的元素组成`

Python有6个序列的内置类型，但最常见的是列表和元组。

序列都可以进行的操作包括索引，切片，加，乘，检查成员。

此外，Python已经内置确定序列的长度以及确定最大和最小的元素的方法。

列表是最常用的Python数据类型，它可以作为一个方括号内的逗号分隔值出现。

列表的数据项不需要具有相同的类型

创建一个列表，只要把逗号分隔的不同的数据项使用方括号括起来即可。如下所示：
```Python
list1 = ['physics', 'chemistry', 1997, 2000]
list2 = [1, 2, 3, 4, 5 ]
list3 = ["a", "b", "c", "d"]
```
列表操作
```Python
list1 = ['physics', 'chemistry', 1997, 2000]
list2 = [1, 2, 3, 4, 5, 6, 7]

# 1. 访问列表中的值
print("list1[0]: ", list1[0])
print("list2[1:5]: ", list2[1:5])

# 2. 更新列表
list2.append(8)
print(list2) # [1, 2, 3, 4, 5, 6, 7, 8]

# 3. 删除列表中的值，通过下标索引删除
del list2[1]
print(list2) # [1, 3, 4, 5, 6, 7, 8]
```
#### 列表脚本操作符

| Python 表达式                | 结果                         | 描述                 |
| :--------------------------- | :--------------------------- | :------------------- |
| len([1, 2, 3])               | 3                            | 长度                 |
| [1, 2, 3] + [4, 5, 6]        | [1, 2, 3, 4, 5, 6]           | 组合                 |
| ['Hi!'] * 4                  | ['Hi!', 'Hi!', 'Hi!', 'Hi!'] | 重复                 |
| 3 in [1, 2, 3]               | True                         | 元素是否存在于列表中 |
| for x in [1, 2, 3]: print x, | 1 2 3                        | 迭代                 |
#### Python列表函数&方法

| 序号 | 函数                                                         |
| :--- | ------------------------------------------------------------ |
| 1    | [cmp(list1, list2)](https://www.codexy.cn/python/att-list-cmp.html) 比较两个列表的元素 |
| 2    | [len(list)](https://www.codexy.cn/python/att-list-len.html) 列表元素个数 |
| 3    | [max(list)](https://www.codexy.cn/python/att-list-max.html) 返回列表元素最大值 |
| 4    | [min(list)](https://www.codexy.cn/python/att-list-min.html) 返回列表元素最小值 |
| 5    | [list(seq)](https://www.codexy.cn/python/att-list-list.html) 将元组转换为列表 |

Python包含以下方法:

| 序号 | 方法                                                         |
| :--- | :----------------------------------------------------------- |
| 1    | [list.append(obj)](https://www.codexy.cn/python/att-list-append.html) 在列表末尾添加新的对象 |
| 2    | [list.count(obj)](https://www.codexy.cn/python/att-list-count.html) 统计某个元素在列表中出现的次数 |
| 3    | [list.extend(seq)](https://www.codexy.cn/python/att-list-extend.html) 在列表末尾一次性追加另一个序列中的多个值（用新列表扩展原来的列表） |
| 4    | [list.index(obj)](https://www.codexy.cn/python/att-list-index.html) 从列表中找出某个值第一个匹配项的索引位置 |
| 5    | [list.insert(index, obj)](https://www.codexy.cn/python/att-list-insert.html) 将对象插入列表 |
| 6    | [list.pop([index=-1])](https://www.codexy.cn/python/att-list-pop.html) 移除列表中的一个元素（默认最后一个元素），并且返回该元素的值 |
| 7    | [list.remove(obj)](https://www.codexy.cn/python/att-list-remove.html) 移除列表中某个值的第一个匹配项 |
| 8    | [list.reverse()](https://www.codexy.cn/python/att-list-reverse.html) 反向列表中元素 |
| 9    | [list.sort(cmp=None, key=None, reverse=False)](https://www.codexy.cn/python/att-list-sort.html) 对原列表进行排序 |

### 元组（Tuple）
Python的元组与列表类似，不同之处在于元组的元素不能修改。

元组使用小括号，列表使用方括号。

元组创建很简单，只需要在括号中添加元素，并使用逗号隔开即可。

示例
```Python
tup1 = ('physics', 'chemistry', 1997, 2000)
tup2 = (1, 2, 3, 4, 5 )
tup3 = "a", "b", "c", "d"
# 空元组
tup1 = ()
# 元组中只包含一个元素时，需要在元素后面添加逗号
tup1 = (50,)
```
#### 元组操作
```Python
#!/usr/bin/python

tup1 = ('physics', 'chemistry', 1997, 2000)
tup2 = (1, 2, 3, 4, 5, 6, 7)
# 1. 访问元组
print("tup1[0]: ", tup1[0])
print("tup2[1:5]: ", tup2[1:5])

# 2. 修改元组(元组中的元素值是不允许修改的，但我们可以对元组进行连接组合)
# 以下修改元组元素操作是非法的。
#tup2[0] = 100

# 创建一个新的元组
tup3 = tup1 + tup2
print(tup3)

# 3. 删除元组(元组中的元素值是不允许删除的，但我们可以使用del语句来删除整个元组)
print(tup2)
del tup2
print("After deleting tup2 : ")
print(tup2)
```
#### 元组运算符

与字符串一样，元组之间可以使用 + 号和 * 号进行运算。这就意味着他们可以组合和复制，运算后会生成一个新的元组。

| Python 表达式                | 结果                         | 描述         |
| :--------------------------- | :--------------------------- | :----------- |
| len((1, 2, 3))               | 3                            | 计算元素个数 |
| (1, 2, 3) + (4, 5, 6)        | (1, 2, 3, 4, 5, 6)           | 连接         |
| ('Hi!',) * 4                 | ('Hi!', 'Hi!', 'Hi!', 'Hi!') | 复制         |
| 3 in (1, 2, 3)               | True                         | 元素是否存在 |
| for x in (1, 2, 3): print x, | 1 2 3                        | 迭代         |

#### 元组索引，截取

因为元组也是一个序列，所以我们可以访问元组中的指定位置的元素，也可以截取索引中的一段元素，如下所示：

元组：

```Python
L = ('spam', 'Spam', 'SPAM!')
```

| Python 表达式 | 结果              | 描述                         |
| :------------ | :---------------- | :--------------------------- |
| L[2]          | 'SPAM!'           | 读取第三个元素               |
| L[-2]         | 'Spam'            | 反向读取，读取倒数第二个元素 |
| L[1:]         | ('Spam', 'SPAM!') | 截取元素                     |

#### 元组内置函数

| 序号 | 方法及描述                                                   |
| :--- | :----------------------------------------------------------- |
| 1    | [cmp(tuple1, tuple2)](https://www.codexy.cn/python/att-tuple-cmp.html) 比较两个元组元素。 |
| 2    | [len(tuple)](https://www.codexy.cn/python/att-tuple-len.html) 计算元组元素个数。 |
| 3    | [max(tuple)](https://www.codexy.cn/python/att-tuple-max.html) 返回元组中元素最大值。 |
| 4    | [min(tuple)](https://www.codexy.cn/python/att-tuple-min.html) 返回元组中元素最小值。 |
| 5    | [tuple(seq)](https://www.codexy.cn/python/att-tuple-tuple.html) 将列表转换为元组。 |

### 字典（Dictionary）
字典是另一种可变容器模型，且可存储任意类型对象。

字典的每个键值 key=>value 对用冒号 : 分割，每个键值对之间用逗号 , 分割，整个字典包括在花括号 {} 中 ,格式如下所示
```Python
d = {key1 : value1, key2 : value2 }
```
键一般是唯一的，如果重复最后的一个键值对会替换前面的，值不需要唯一。
#### 字典操作
```Python
#!/usr/bin/python

dict = {'Name': 'Zara', 'Age': 7, 'Class': 'First'}

# 1. 访问字典
print("dict['Name']: ", dict['Name'])
print("dict['Age']: ", dict['Age'])

# 2. 修改字典
dict['Age'] = 8 # 更新
dict['School'] = "CODEXY" # 添加
print("dict['Age']: ", dict['Age'])
print("dict['School']: ", dict['MIT'])

# 3. 删除字典元素
del dict['Name']  # 删除键是'Name'的条目
dict.clear()      # 清空词典所有条目
del dict          # 删除词典

```
#### 字典的两个特性
1. 不允许同一个键出现两次。创建时如果同一个键被赋值两次，后一个值会被覆盖
2. 键必须不可变，所以可以用数字，字符串或元组充当，所以用列表就不行
#### 字典内置函数&方法

Python字典包含了以下内置函数：

| 序号 | 函数及描述                                                   |
| :--- | :----------------------------------------------------------- |
| 1    | [cmp(dict1, dict2)](https://www.codexy.cn/python/att-dictionary-cmp.html) 比较两个字典元素。 |
| 2    | [len(dict)](https://www.codexy.cn/python/att-dictionary-len.html) 计算字典元素个数，即键的总数。 |
| 3    | [str(dict)](https://www.codexy.cn/python/att-dictionary-str.html) 输出字典可打印的字符串表示。 |
| 4    | [type(variable)](https://www.codexy.cn/python/att-dictionary-type.html) 返回输入的变量类型，如果变量是字典就返回字典类型。 |

Python字典包含了以下内置方法：

| 序号 | 函数及描述                                                   |
| :--- | :----------------------------------------------------------- |
| 1    | [dict.clear()](https://www.codexy.cn/python/att-dictionary-clear.html) 删除字典内所有元素 |
| 2    | [dict.copy()](https://www.codexy.cn/python/att-dictionary-copy.html) 返回一个字典的浅复制 |
| 3    | [dict.fromkeys(seq[, val])](https://www.codexy.cn/python/att-dictionary-fromkeys.html) 创建一个新字典，以序列 seq 中元素做字典的键，val 为字典所有键对应的初始值 |
| 4    | [dict.get(key, default=None)](https://www.codexy.cn/python/att-dictionary-get.html) 返回指定键的值，如果值不在字典中返回default值 |
| 5    | [dict.has_key(key)](https://www.codexy.cn/python/att-dictionary-has_key.html) 如果键在字典dict里返回true，否则返回false |
| 6    | [dict.items()](https://www.codexy.cn/python/att-dictionary-items.html) 以列表返回可遍历的(键, 值) 元组数组 |
| 7    | [dict.keys()](https://www.codexy.cn/python/att-dictionary-keys.html) 以列表返回一个字典所有的键 |
| 8    | [dict.setdefault(key, default=None)](https://www.codexy.cn/python/att-dictionary-setdefault.html) 和get()类似, 但如果键不存在于字典中，将会添加键并将值设为default |
| 9    | [dict.update(dict2)](https://www.codexy.cn/python/att-dictionary-update.html) 把字典dict2的键/值对更新到dict里 |
| 10   | [dict.values()](https://www.codexy.cn/python/att-dictionary-values.html) 以列表返回字典中的所有值 |
| 11   | [pop(key[,default])](https://www.codexy.cn/python/python-att-dictionary-pop.html) 删除字典给定键 key 所对应的值，返回值为被删除的值。key值必须给出。 否则，返回default值。 |
| 12   | [popitem()](https://www.codexy.cn/python/python-att-dictionary-popitem.html) 随机返回并删除字典中的一对键和值。 |


### 集合（Set）
在 Python 中，集合（Set）是一种无序且元素唯一的数据类型。它基于哈希表实现，因此支持高效的成员检测和删除操作。
#### 创建集合
可以使用大括号 {} 或者 set() 构造函数来创建集合。
```Python
# 使用大括号创建集合
my_set = {1, 2, 3, 4, 5}

# 使用 set() 构造函数创建集合
another_set = set([5, 6, 7, 8, 9])
```
#### 集合的特点
- 无序性：集合中的元素没有固定的顺序。
- 唯一性：集合中不允许包含重复的元素。
- 可变性：集合是可变的，可以添加或删除元素。
- 哈希性：集合中的元素必须是可哈希的，即不可变类型，如整数、浮点数、元组、字符串等。


## 面向对象编程
Python从设计之初就已经是一门面向对象的语言，正因为如此，在Python中创建一个类和对象是很容易的。
#### 面向对象技术简介
1. 类(Class): 用来描述具有相同的属性和方法的对象的集合。它定义了该集合中每个对象所共有的属性和方法。对象是类的实例。
2. 类变量：类变量在整个实例化的对象中是公用的。类变量定义在类中且在函数体之外。类变量通常不作为实例变量使用。
3. 数据成员：类变量或者实例变量, 用于处理类及其实例对象的相关的数据。
4. 方法重写：如果从父类继承的方法不能满足子类的需求，可以对其进行改写，这个过程叫方法的覆盖（override），也称为方法的重写。
5. 局部变量：定义在方法中的变量，只作用于当前实例的类。
6. 实例变量：在类的声明中，属性是用变量来表示的。这种变量就称为实例变量，是在类声明的内部但是在类的其他成员方法之外声明的。
7. 继承：即一个派生类（derived class）继承基类（base 1. class）的字段和方法。继承也允许把一个派生类的对象作为一个基类对象对待。例如，有这样一个设计：一个Dog类型的对象派生自Animal类，这是模拟"是一个（is-a）"关系（例图，Dog是一个Animal）。
8. 实例化：创建一个类的实例，类的具体对象。
9. 方法：类中定义的函数。
10. 对象：通过类定义的数据结构实例。对象包括两个数据成员（类变量和实例变量）和方法。


### 类和对象
#### 创建类
```Python
class ClassName:
   '''类的帮助信息'''   #类文档字符串
   class_suite  #类体
```
类的帮助信息可以通过ClassName.__doc__查看。
class_suite 由类成员，方法，数据属性组成。

示例
```Python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

class Employee:
    '所有员工的基类'
    empCount = 0

    def __init__(self, name, salary):
        self.name = name
        self.salary = salary
        Employee.empCount += 1

    def displayCount(self):
        print("Total Employee %d" % Employee.empCount)

    def displayEmployee(self):
        print("Name : ", self.name, ", Salary: ", self.salary)
```
#### 创建类的实例对象
实例化类其他编程语言中一般用关键字 new，但是在 Python 中并没有这个关键字，类的实例化类似函数调用方式。
以下使用类的名称 Employee 来实例化，并通过 `__init__` 方法接收参数。
```Python
"创建 Employee 类的第一个对象"
emp1 = Employee("Zara", 2000)
"创建 Employee 类的第二个对象"
emp2 = Employee("Manni", 5000)
```
#### Python 内置类属性
```
__dict__ : 类的属性（包含一个字典，由类的数据属性组成）
__doc__ :类的文档字符串
__name__: 类名
__module__: 类定义所在的模块（类的全名是'__main__.className'，如果类位于一个导入模块mymod中，那么className.__module__ 等于 mymod）
__bases__ : 类的所有父类构成元素（包含了一个由所有父类组成的元组）
```


### 类的继承
面向对象的编程带来的主要好处之一是代码的重用，实现这种重用的方法之一是通过继承机制。

通过继承创建的新类称为子类或派生类，被继承的类称为基类、父类或超类。

继承语法
```
class 派生类名(基类名)
    ...
```
在python中继承中的一些特点：

1、如果在子类中需要父类的构造方法就需要显示的调用父类的构造方法，或者不重写父类的构造方法。详细说明可查看：python 子类继承父类构造函数说明。
2、在调用基类的方法时，需要加上基类的类名前缀，且需要带上 self 参数变量。区别在于类中调用普通函数时并不需要带上 self 参数
3、Python 总是首先查找对应类型的方法，如果它不能在派生类中找到对应的方法，它才开始到基类中逐个查找。（先在本类中查找调用的方法，找不到才去基类中找）。


## 模块和包
Python 模块(Module)，是一个 Python 文件，以 .py 结尾，包含了 Python 对象定义和Python语句。

模块让你能够有逻辑地组织你的 Python 代码段。

把相关的代码分配到一个模块里能让你的代码更好用，更易懂。

模块能定义函数，类和变量，模块里也能包含可执行的代码。
### 模块的导入
模块定义好后，我们可以使用 import 语句来引入模块，语法如下：
```
import module1[, module2[,... moduleN]]
```
比如要引用模块 math，就可以在文件最开始的地方用 import math 来引入。在调用 math 模块中的函数时，必须这样引用：
```
模块名.函数名

```
#### from…import 语句
Python 的 from 语句让你从模块中导入一个指定的部分到当前命名空间中。语法如下：
```
from modname import name1[, name2[, ... nameN]]
```
#### from…import* 语句
把一个模块的所有内容全都导入到当前的命名空间也是可行的，只需使用如下声明：
```
from modname import *
```


## 文件操作
### 打开和关闭文件
Python 提供了必要的函数和方法进行默认情况下的文件基本操作。你可以用 file 对象做大部分的文件操作。
#### open 函数
你必须先用Python内置的open()函数打开一个文件，创建一个file对象，相关的方法才可以调用它进行读写。

语法：
```
file object = open(file_name [, access_mode][, buffering])
```
各个参数的细节如下：

- file_name：file_name变量是一个包含了你要访问的文件名称的字符串值。
- access_mode：access_mode决定了打开文件的模式：只读，写入，追加等。所有可取值见如下的完全列表。这个参数是非强制的，默认文件访问模式为只读(r)。
- buffering:如果buffering的值被设为0，就不会有寄存。如果buffering的值取1，访问文件时会寄存行。如果将buffering的值设为大于1的整数，表明了这就是的寄存区的缓冲大小。如果取负值，寄存区的缓冲大小则为系统默认。



#### 读写文件内容
不同模式打开文件的完全列表：

| 模式 | 描述                                                         |
| :--- | :----------------------------------------------------------- |
| t    | 文本模式 (默认)。                                            |
| x    | 写模式，新建一个文件，如果该文件已存在则会报错。             |
| b    | 二进制模式。                                                 |
| +    | 打开一个文件进行更新(可读可写)。                             |
| U    | 通用换行模式（不推荐）。                                     |
| r    | 以只读方式打开文件。文件的指针将会放在文件的开头。这是默认模式。 |
| rb   | 以二进制格式打开一个文件用于只读。文件指针将会放在文件的开头。这是默认模式。一般用于非文本文件如图片等。 |
| r+   | 打开一个文件用于读写。文件指针将会放在文件的开头。           |
| rb+  | 以二进制格式打开一个文件用于读写。文件指针将会放在文件的开头。一般用于非文本文件如图片等。 |
| w    | 打开一个文件只用于写入。如果该文件已存在则打开文件，并从开头开始编辑，即原有内容会被删除。如果该文件不存在，创建新文件。 |
| wb   | 以二进制格式打开一个文件只用于写入。如果该文件已存在则打开文件，并从开头开始编辑，即原有内容会被删除。如果该文件不存在，创建新文件。一般用于非文本文件如图片等。 |
| w+   | 打开一个文件用于读写。如果该文件已存在则打开文件，并从开头开始编辑，即原有内容会被删除。如果该文件不存在，创建新文件。 |
| wb+  | 以二进制格式打开一个文件用于读写。如果该文件已存在则打开文件，并从开头开始编辑，即原有内容会被删除。如果该文件不存在，创建新文件。一般用于非文本文件如图片等。 |
| a    | 打开一个文件用于追加。如果该文件已存在，文件指针将会放在文件的结尾。也就是说，新的内容将会被写入到已有内容之后。如果该文件不存在，创建新文件进行写入。 |
| ab   | 以二进制格式打开一个文件用于追加。如果该文件已存在，文件指针将会放在文件的结尾。也就是说，新的内容将会被写入到已有内容之后。如果该文件不存在，创建新文件进行写入。 |
| a+   | 打开一个文件用于读写。如果该文件已存在，文件指针将会放在文件的结尾。文件打开时会是追加模式。如果该文件不存在，创建新文件用于读写。 |
| ab+  | 以二进制格式打开一个文件用于追加。如果该文件已存在，文件指针将会放在文件的结尾。如果该文件不存在，创建新文件用于读写。 |


下图很好的总结了这几种模式：

![file-accessmode](/images/python/file-accessmode.png)

|    模式    |  r   |  r+  |  w   |  w+  |  a   |  a+  |
| :--------: | :--: | :--: | :--: | :--: | :--: | :--: |
|     读     |  +   |  +   |      |  +   |      |  +   |
|     写     |      |  +   |  +   |  +   |  +   |  +   |
|    创建    |      |      |  +   |  +   |  +   |  +   |
|    覆盖    |      |      |  +   |  +   |      |      |
| 指针在开始 |  +   |  +   |  +   |  +   |      |      |
| 指针在结尾 |      |      |      |      |  +   |  +   |


### 使用 with 语句处理文件
在 Python 中，with 语句用于管理代码块执行过程中的上下文环境，确保在进入和退出代码块时，资源的正确获取和释放。它通常与支持上下文管理协议（Context Management Protocol）的对象一起使用，如文件操作、数据库连接、网络连接等需要在使用后及时关闭或释放资源的情况。
```Python
with open('example.txt', 'r') as file:
    content = file.read()
    print(content)
# 在退出 with 块后，文件会自动关闭，即使出现异常也能保证资源的释放
```

## 常用标准库
- 操作系统接口（os 模块）
- 文件路径操作（pathlib 模块）
- 时间和日期处理（datetime 模块）
- 正则表达式（re 模块）

[Python 常用模块和常用第三方模块示例](https://gitee.com/lu_shuan/python-demo/tree/master)
## 进阶话题
- 并发编程（多线程和多进程）
- 异步编程（asyncio 模块）
- 数据库访问（sqlite3、SQLAlchemy 等）

## 实际应用示例
- 简单的 Web 开发（使用 Flask 或 Django）
- 数据分析与可视化（使用 Pandas 和 Matplotlib）


## 资源推荐
- [官方文档链接](https://www.python.org/)
- [谷歌Python代码风格指南 中文翻译](https://github.com/shendeguize/GooglePythonStyleGuideCN)
- [30个Python常用极简代码，拿走就用](https://www.51cto.com/article/620877.html)
- [收藏 | 学习Python的11个顶级Github存储库](https://zhuanlan.zhihu.com/p/431382214)
- [改善Python程序的91个建议](https://zhuanlan.zhihu.com/p/32817459)
- [Django](https://www.djangoproject.com/start/)
- [Django-中文](https://docs.djangoproject.com/zh-hans/5.0/)

