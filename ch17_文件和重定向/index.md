# shell 文件和重定向


Unix中的每个进程默认都可以访问三个输入/输出通道：STDIN（标准输入）、STDOUT（标准输出）和STDERR（标准错误）。

##  打印myfile中包含单词foo的行
```shell
grep foo < myfile
```
##  文件合并
一般配合文件切割使用

将文件file1、file2 合并至新文件combined
```shell
cat file1 file2 > combined 
```
##  文件追加
```shell
date >> log
```
##  endmarker
要在脚本中从字面上指定STDIN的内容，请使用<<endmarker表示法：
```shell
cat > file1 <<UNTILHERE
All of this will be printed out.

Since all of this is going into cat on STDIN.

UNTILHERE

# >> 追加
cat >> file1 <<UNTILHERE
All of this will be printed out.

Since all of this is going into cat on STDIN.

UNTILHERE

# 一般使用EOF
cat >> file1 <<EOF
All of this will be printed out.

Since all of this is going into cat on STDIN.

EOF

```
##  输出错误日志至文件
```shell
httpd 2> error.log
```

##  chanel 输出类型
STDIN is channel 0, STDOUT is channel 1, while STDERR is channel 2.
```shell
 grep foo nofile 2>&1 # errors will appear on STDOUT，既正常输出也输出错误日志
```
## 总结
重定向

|command | 释义|
|---|---|
| cmd 1> fiel       | 把 标准输出 重定向到 file 文件中  | 
| cmd > file 2>&1   | 把 标准输出 和 标准错误 一起重定向到 file 文件中  | 
| cmd 2> file       | 把 标准错误 重定向到 file 文件中  | 
| cmd 2>> file      | 把 标准错误 重定向到 file 文件中(追加)  | 
| cmd >> file 2>&1  | 把 标准输出 和 标准错误 一起重定向到 file 文件中(追加)  | 
| cmd < file >file2 | cmd 命令以 file 文件作为 stdin(标准输入)，以 file2 文件作为 标准输出  | 
| cat <>file        | 以读写的方式打开 file  | 
