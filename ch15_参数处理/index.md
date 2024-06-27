# shell 参数处理

Shell 参数处理是指在 Shell 脚本中，对命令行传入的参数进行处理和操作的技术。这在编写 Shell 脚本时非常重要，因为它允许脚本根据不同的参数执行不同的操作，增强了脚本的灵活性和适用性。

在 Shell 脚本中，可以使用特殊变量 "$1"、"$2"、"$3" 等来引用命令行传入的参数，其中 "$1" 表示第一个参数，"$2" 表示第二个参数，依此类推。这些参数可以根据脚本需要进行操作，比如作为文件名、目录名、网址等。

以下是一些示例，演示了如何在 Shell 脚本中处理参数：

##  显示脚本名称和参数：
```shell
#!/bin/bash
echo "脚本名称：$0"
echo "第一个参数：$1"
echo "第二个参数：$2"
```
运行 `./script.sh argument1 argument2`，将会输出：
```shell
脚本名称：./script.sh
第一个参数：argument1
第二个参数：argument2
```

##  判断参数个数是否正确：
```shell
#!/bin/bash
if [ $# -ne 2 ]; then
    echo "需要两个参数"
    exit 1
fi
echo "参数个数正确"
```
运行 `./script.sh argument1`，将会输出：`需要两个参数`


##  使用参数作为文件名：
```shell
#!/bin/bash
filename="$1"
if [ -f "$filename" ]; then
    echo "文件存在"
else
    echo "文件不存在"
fi
```
运行 `./script.sh myfile.txt`，将会输出：`文件存在`


##  循环处理多个参数：
```shell
#!/bin/bash
for arg in "$@"
do
    echo "参数: $arg"
done
```
运行 `./script.sh argument1 argument2 argument3`，将会输出：
```
参数: argument1
参数: argument2
参数: argument3
```

##  特别的参数
注意参数超过个位数时的处理

```shell
echo $0      # 打印脚本名称
echo $1      # 打印第一个参数 
echo $2      # 打印第二个参数
echo $9      # 打印第九个参数
echo $10     # 打印第一个参数后面跟一个0 
echo ${10}   # 打印第十个参数
echo $#      # 打印参数的数量
```

这些示例展示了 Shell 参数处理的一些基本用法，可以根据实际需求灵活运用。
