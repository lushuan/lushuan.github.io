# shell 脚本退出码

很多时候我们需要获取脚本的执行结果，即退出状态，通常0表示执行成功，而非0表示失败。为了获得退出码，我们需要使用exit

```shell
#!/bin/bash
function myfun()
{
 if [ $# -lt 2 ]
 then
  echo "para num error"
  exit 1
 fi

 echo "ok"
 exit 2
}

if [ $# -lt 1 ]
then
 echo "para num error"
 exit 1
fi

returnVal=`myfun aa`
echo "end shell"
exit 0
```
