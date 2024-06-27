# shell 循环

## for 循环
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
每隔3打印一次，即打印5,8,11,14

示例
```shell
 for f in *.c
 do
   gcc -o ${f%.c} $f
 done
```
## while 循环
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

## until 循环
```shell
num=1
#  当command不为0时循环
until [ $num -gt 10 ]
do
  echo until $num
  ((num=$num+2))
done
```


