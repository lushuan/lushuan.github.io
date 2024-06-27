# shell 数组

要给某个环境变量设置多个值，可以把值放在括号里，值与值之间用空格分隔。
## 声明
```shell
mytest=(one two three four five)

echo ${mytest[2]}
# 输出 three

echo ${mytest[*]} 
# 输出 one two three four five
```

## 修改数组值

```shell
mytest=(one two three four five)

mytest[2]=seven
```

## 删除数组中的值
```shell
mytest=(one two three four five)
unset mytest[2] # 删除单个元素，索引下标保留映射的值被置空
echo ${mytest[2]}
# 输出为空
unset mytest  # 删除整个数组
```


## 示例
docker 中的应用启动后再进行健康检查是一个启动脚本
```
#!/bin/bash

APP=("java" "health_check")
java=(
     "java --add-opens java.base/java.lang=ALL-UNNAMED -DAPP_NAME=zcmMonitor -XX:MaxRAMPercentage=70.0 -Dspring.config.location=/app/conf/monitorConfig.properties -Dlogging.config=/app/conf/logback.xml -cp /app/lib/zcm-monitor-9.1.3-sec-SNAPSHOT.jar:/app/lib/run/* com.ztesoft.zsmart.zcm.monitor.App" # start command
     "APP_NAME=zcmMonitor" # process key
     "echo OK"    # liveness check command
     "OK"         # liveness successful code
     "curl -k -m 10 -s https://127.0.0.1:${NMS_MONITOR_SSL_PORT}/app/health 2>/dev/null"    # readiness check command
     "UP"         # readiness successful code
)
health_check=(
    "sh /app/health_check.sh" # start command
     "health_check.sh"       # process key
     "echo OK"    # liveness check command
     "OK"         # liveness successful code
     "echo OK"    # readiness check command
     "OK"         # readiness successful code
)
```
然后解析该数组配置设置启动顺序

## 总结
有时数组变量会让事情很麻烦，所以在shell脚本编程时并不常用。对其他shell而言，数组变
量的可移植性并不好，如果需要在不同的shell环境下从事大量的脚本编写工作，这会带来很多不
便

|command | 释义|
|---|---|
|A=(a b c def) | 将变量定义为数組 | 
|${#A[*]}      | 数组个数 | 
|${A[*]}       | 数组所有元素,大字符串 | 
|${A[@]}       | 数组所有元素,类似列表可迭代 | 
|${A[2]}       | 脚本的一个参数或数组第三位 |
