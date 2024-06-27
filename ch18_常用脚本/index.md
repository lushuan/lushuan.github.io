# shell 常用脚本

一些生产中常用到的片段脚本
## 字符串分割模板
IFS 默认是空格进行分割
```shell
oldIFS=$IFS

IFS=":"
data="大牛:二狗:三驴"
for item in $data;
do
     echo $item
done

IFS=$oldIFS
```

##  进程探测
```shell
#!/bin/bash

PROC_NAME=nms-prometheus
ProcNumber=`ps -ef |grep -w $PROC_NAME|grep -v grep|wc -l`
if [ $ProcNumber -le 0 ];then
   echo "nmsprometheus is not run"
   sh /home/zoms/monitor/start.sh
else
   echo "nmsprometheus is  running.."
fi

```

##  目录循环
```shell
#!/bin/sh

dirlist="ls"
for dir in `$dirlist`
do
  if [ -d $dir ]; then
    helm install $dir $dir -n zcm9 -f ../values.yaml
  fi
done

```

##  进程检测
判断进程是否存在，如果不存在则进行手动重启，如果重启还不存在则做另外的动作
```shell
#!/bin/sh
nginxpid=$(ps -C nginx --no-header|wc -l)
#1.判断Nginx是否存活,如果不存活则尝试启动Nginx
if [ $nginxpid -eq 0 ];then
    systemctl start nginx
    sleep 3
    #2.等待3秒后再次获取一次Nginx状态
    nginxpid=$(ps -C nginx --no-header|wc -l) 
    #3.再次进行判断, 如Nginx还不存活则停止Keepalived,让地址进行漂移,并退出脚本  
    if [ $nginxpid -eq 0 ];then
        systemctl stop keepalived
   fi
fi
```
## 定时清空文件内容
结合cron 使用
```shell
#!/bin/bash  
logfile=/tmp/`date +%H-%F`.log  
n=`date +%H`  
if [ $n -eq 00 ] || [ $n -eq 12 ]  
then  
#通过for循环，以find命令作为遍历条件，将目标目录下的所有文件进行遍历并做相应操作  
for i in `find /data/log/ -type f`  
do  
true > $i  
done  
else  
for i in `find /data/log/ -type f`  
do  
du -sh $i >> $logfile  
done  
fi
```

## 探测主机端口是否处于监听状态
```shell
#!/bin/bash
ip_list="10.1.1.81 10.1.1.83 10.1.11.159"
port=9100
function main() {
for ip in $ip_list
  do
    result=`echo -e "\n" |timeout --signal=9 2 telnet $ip $port 2>/dev/null | grep Connected | wc -l`
    if [ $result -ne 1 ]
    then
       echo $ip  >> $HOME/ip_refused.txt
    fi
  done
}
main
```
