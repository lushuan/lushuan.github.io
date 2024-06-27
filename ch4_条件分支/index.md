# shell 条件分支

一般说明，如果命令执行成功，则其返回值为0，否则为非0，因此，可以通过下面的方式判断上条命令的执行结果：

##  if 分支
```shell
if [ $? -eq 0 ]
then
  echo "success"
elif [ $? -eq 1 ]
then
  echo "failed,code is 1"
else
  echo "other code"
fi

```
### 多个条件
方式一
```shell
if [ 10 -gt 5 -o 10 -gt 4 ];then
echo "10>5 or 10 >4"
fi

```

方式二
```shell
if [ 10 -gt 5 ] || [ 10 -gt 4 ];then
echo "10>5 or 10 >4"
fi
```

总结：
- -o or 或者，同||
- -a and 与，同&&
- ! 非


##  case 分支
```shell
name="aa"

case $name in
  "aa")
      echo "name is $name"
  ;;
  "")
      echo "name is empty"
  ;;
  "bb")
      echo "name is $name"
  ;;
  *)
      echo "other name"
  ;;
esac

```shell
注意：
- []前面要有空格，它里面是逻辑表达式
- if elif后面要跟then，然后才是要执行的语句
- 如果想打印上一条命令的执行结果，最好的做法是将 $?赋给一个变量，因为一旦执行了一条命令，$?的值就可能会变。
- case每个分支最后以两个分号结尾，最后是case反过来写，即esac。





