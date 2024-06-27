# shell 算术表达式


## 方式一 命令expr
命令expr打印算术表达式的结果，但必须小心 * 号
```shell
expr 3 + 12      # 打印 15
expr 3 * 12      # (probably) crashes: * 需要进行转义，这行语句会执行报错 
expr 3 \* 12     # 打印 36
```

## 方式二 (( assignable = expression ))
```shell
(( x = 3 + 12 )); echo $x    # 打印 15
(( x = 3 * 12 )); echo $x    # 打印 36
```

## 方式三
```shell
echo $(( 3 + 12 ))   # 打印 15
echo $(( 3 * 12 ))   # 打印 36
```

## 指定类型
虽然隐式声明变量是bash中的规范，但可以显式声明变量并为其指定类型

指定整型
```shell
declare -i number
number=2+4*10
echo $number        # 打印 42

another=2+4*10
echo $another       # 打印 2+4*10

number="foobar"
echo $number        # 打印 0
```

