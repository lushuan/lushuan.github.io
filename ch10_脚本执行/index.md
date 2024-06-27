# shell 脚本执行

## 常见执行方式
```shell
./test.sh
```

## 其它执行方式
```shell
sh test.sh #在子进程中执行
sh -x test.sh #会在终端打印执行到命令，适合调试
source test.sh #test.sh在父进程中执行
. test.sh #不需要赋予执行权限，临时执行
```

