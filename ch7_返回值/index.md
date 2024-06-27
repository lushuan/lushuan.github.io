# shell 返回值

通常函数的return返回值只支持0-255，因此想要获得返回值，可以通过下面的方式。

```shell
function myfunc() {
 local myresult='some value'
 echo $myresult
}
val=$(myfunc) #val的值为some value

```

通过return的方式适用于判断函数的执行是否成功：
```shell
function myfunc() {
 #do something
 return 0
}

if myfunc;then
 echo "success"
else
 echo "failed"
fi
```
