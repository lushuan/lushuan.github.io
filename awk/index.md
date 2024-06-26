# 文本三剑客之 awk

awk 是 Linux/Unix 系统中的一个强大的文本处理工具,主要用于处理文本文件和字符串。

## 特性
awk 的一些关键特性:

- 可以根据字段进行操作,如打印指定列、求列的总和等
- 支持条件判断和循环语句,可以实现复杂的文本处理
- 使用正则表达式进行模式匹配和内容提取
- 内置字符串操作函数,如匹配、替换、截取等

## 适用场景
awk 适用于报表生成、格式转换、数学运算等场景。掌握了 awk,可以大大提高 Bash 脚本的文本处理能力

## 内置函数
内置函数

|command | 释义|
|---|---|
| gsub(r,s)          | 在整个$0中用s替代r   相当于 sed 's///g' | 
| gsub(r,s,t)        | 在整个t中用s替代r | 
| index(s,t)         | 返回s中字符串t的第一位置 | 
| length(s)          | 返回s长度 | 
| match(s,r)         | 测试s是否包含匹配r的字符串 | 
| split(s,a,fs)      | 在fs上将s分成序列a | 
| sprint(fmt,exp)    | 返回经fmt格式化后的exp | 
| sub(r,s)           | 用$0中最左边最长的子串代替s   相当于 sed 's///' | 
| substr(s,p)        | 返回字符串s中从p开始的后缀部分 | 
| substr(s,p,n)      | 返回字符串s中从p开始长度为n的后缀部分 |

## awk 判断
- awk '{print ($1>$2)?"第一排"$1:"第二排"$2}'      # 条件判断 括号代表if语句判断 "?"代表then ":"代表else
- awk '{max=($1>$2)? $1 : $2; print max}'          # 条件判断 如果$1大于$2,max值为为$1,否则为$2
- awk '{if ( $6 > 50) print $1 " Too high" ;else print "Range is OK"}' file
- awk '{if ( $6 > 50) { count++;print $3 } else { x+5; print $2 } }' file

## awk 循环
- awk '{i = 1; while ( i <= NF ) { print NF, $i ; i++ } }' file
- awk '{ for ( i = 1; i <= NF; i++ ) print NF,$i }' file

## 示例
- awk '{print $1, $3}' file.txt   每行按空格分割成多个字段,输出第1个和第3个字段   
- awk '{sum+=$1*$1} END {print sum}' file.txt    求第一列的平方和    
- awk 'length > 80' file.txt    查找长度大于80的行    
- awk '/regex/ {print $0}' log.txt    使用正则表达式匹配内容    
- awk '{print $NF}' file.txt     打印最后一列   
- awk '{print $(NF-1) }' file.txt    打印倒数第二列    
- awk '{OFS="#";$1=$1;print $0}' file.txt    OFS 输出记录分隔符    
- awk '{if(NR==3){print $0} else {print "不是第三行"}}' file.txt    打印指定行    
- awk 'NR==1||NR==3{print $0}' test.txt    打印第一行或者第三行  
- awk '/Tom/' file               # 打印匹配到得行
- awk '/^Tom/{print $1}'         # 匹配Tom开头的行 打印第一个字段
- awk '$1 !~ /ly$/'              # 显示所有第一个字段不是以ly结尾的行
- awk '$3 <40'                   # 如果第三个字段值小于40才打印
- awk '$4==90{print $5}'         # 取出第四列等于90的第五列
- awk '/^(no|so)/' test          # 打印所有以模式no或so开头的行
- awk '$3 * $4 > 500'            # 算术运算(第三个字段和第四个字段乘积大于500则显示)
- awk '{print NR" "$0}'          # 加行号
- awk '/tom/,/suz/'              # 打印tom到suz之间的行
- awk '{a+=$1}END{print a}'      # 列求和
- awk 'sum+=$1{print sum}'       # 将$1的值叠加后赋给sum
- awk '{a+=$1}END{print a/NR}'   # 列求平均值
- awk '!s[$1 $3]++' file         # 根据第一列和第三列过滤重复行
- awk -F'[ :\t]' '{print $1,$2}'           # 以空格、:、制表符Tab为分隔符
- awk '{print "'"$a"'","'"$b"'"}'          # 引用外部变量
- awk '{if(NR==52){print;exit}}'           # 显示第52行
- awk '/关键字/{a=NR+2}a==NR {print}'      # 取关键字下第几行
- awk 'gsub(/liu/,"aaaa",$1){print $0}'    # 只打印匹配替换后的行
- ll | awk -F'[ ]+|[ ][ ]+' '/^$/{print $8}'             # 提取时间,空格不固定
- awk '{$1="";$2="";$3="";print}'                        # 去掉前三列
- echo aada:aba|awk '/d/||/b/{print}'                    # 匹配两内容之一
- echo aada:abaa|awk -F: '$1~/d/||$2~/b/{print}'         # 关键列匹配两内容之一
- echo Ma asdas|awk '$1~/^[a-Z][a-Z]$/{print }'          # 第一个域匹配正则
- echo aada:aaba|awk '/d/&&/b/{print}'                   # 同时匹配两条件
- awk 'length($1)=="4"{print $1}'                        # 字符串位数
- awk '{if($2>3){system ("touch "$1)}}'                  # 执行系统命令
- awk '{sub(/Mac/,"Macintosh",$0);print}'                # 用Macintosh替换Mac
- awk '{gsub(/Mac/,"MacIntosh",$1); print}'              # 第一个域内用Macintosh替换Mac
- awk -F '' '{ for(i=1;i<NF+1;i++)a+=$i  ;print a}'      # 多位数算出其每位数的总和.比如 1234， 得到 10
- awk '{ i=$1%10;if ( i == 0 ) {print i}}'               # 判断$1是否整除(awk中定义变量引用时不能带 $ )
- awk 'BEGIN{a=0}{if ($1>a) a=$1 fi}END{print a}'        # 列求最大值  设定一个变量开始为0，遇到比该数大的值，就赋值给该变量，直到结束
- awk 'BEGIN{a=11111}{if ($1<a) a=$1 fi}END{print a}'    # 求最小值
- awk '{if(A)print;A=0}/regexp/{A=1}'                    # 查找字符串并将匹配行的下一行显示出来，但并不显示匹配行
- awk '/regexp/{print A}{A=$0}'                          # 查找字符串并将匹配行的上一行显示出来，但并不显示匹配行
- awk '{if(!/mysql/)gsub(/1/,"a");print $0}'             # 将1替换成a，并且只在行中未出现字串mysql的情况下替换
- awk 'BEGIN{srand();fr=int(100*rand());print fr;}'      # 获取随机数
- awk '{if(NR==3)F=1}{if(F){i++;if(i%7==1)print}}'       # 从第3行开始，每7行显示一次
- awk '{if(NF<1){print i;i=0} else {i++;print $0}}'      # 显示空行分割各段的行数
- echo +null:null  |awk -F: '$1!~"^+"&&$2!="null"{print $0}'       # 关键列同时匹配
- awk -v RS=@ 'NF{for(i=1;i<=NF;i++)if($i) printf $i;print ""}'    # 指定记录分隔符
- awk '{b[$1]=b[$1]$2}END{for(i in b){print i,b[i]}}'              # 列叠加
- awk '{ i=($1%100);if ( $i >= 0 ) {print $0,$i}}'                 # 求余数
- awk '{b=a;a=$1; if(NR>1){print a-b}}'                            # 当前行减上一行
- awk '{a[NR]=$1}END{for (i=1;i<=NR;i++){print a[i]-a[i-1]}}'      # 当前行减上一行
- awk -F: '{name[x++]=$1};END{for(i=0;i<NR;i++)print i,name[i]}'   # END只打印最后的结果,END块里面处理数组内容
- awk '{sum2+=$2;count=count+1}END{print sum2,sum2/count}'         # $2的总和  $2总和除个数(平均值)
- awk -v a=0 -F 'B' '{for (i=1;i<NF;i++){ a=a+length($i)+1;print a  }}'     # 打印所以B的所在位置
- awk 'BEGIN{ "date" | getline d; split(d,mon) ; print mon[2]}' file        # 将date值赋给d，并将d设置为数组mon，打印mon数组中第2个元素
- awk 'BEGIN{info="this is a test2010test!";print substr(info,4,10);}'      # 截取字符串(substr使用)
- awk 'BEGIN{info="this is a test2010test!";print index(info,"test")?"ok":"no found";}'      # 匹配字符串(index使用)
- awk 'BEGIN{info="this is a test2010test!";print match(info,/[0-9]+/)?"ok":"no found";}'    # 正则表达式匹配查找(match使用)
- awk '{for(i=1;i<=4;i++)printf $i""FS; for(y=10;y<=13;y++)  printf $y""FS;print ""}'        # 打印前4列和后4列
- awk 'BEGIN{for(n=0;n++<9;){for(i=0;i++<n;)printf i"x"n"="i*n" ";print ""}}'                # 乘法口诀
- awk 'BEGIN{info="this is a test";split(info,tA," ");print length(tA);for(k in tA){print k,tA[k];}}'             # 字符串分割(split使用)
- awk '{if (system ("grep "$2" tmp/* > /dev/null 2>&1") == 0 ) {print $1,"Y"} else {print $1,"N"} }' a            # 执行系统命令判断返回状态
- awk  '{for(i=1;i<=NF;i++) a[i,NR]=$i}END{for(i=1;i<=NF;i++) {for(j=1;j<=NR;j++) printf a[i,j] " ";print ""}}'   # 将多行转多列

常用示例

```shell
#删除temp文件的重复行
awk '!($0 in array) { array[$0]; print }' temp

#查看最长使用的10个unix命令
awk '{print $1}' ~/.bash_history | sort | uniq -c | sort -rn | head -n 10

#查看机器的ip列表
ifconfig -a | awk '/Bcast/{print $2}' | cut -c 5-19

#查看机器的每个远程链接机器的连接数
netstat -antu | awk '$5 ~ /[0-9]:/{split($5, a, ":"); ips[a[1]]++} END {for (ip in ips) print ips[ip], ip | "sort -k1 -nr"}'

#查看某个进程打开的socket数量
ps aux | grep [process] | awk '{print $2}' | xargs -I % ls /proc/%/fd | wc -l


#查看无线网络的ip
sudo ifconfig wlan0 | grep inet | awk 'NR==1 {print $2}' | cut -c 6-

#批量重命名文件
find . -name '*.jpg' | awk 'BEGIN{ a=0 }{ printf "mv %s name%01d.jpg\n", $0, a++ }' | bash

#查看某个用户打开的文件句柄列表
for x in `ps -u 500 u | grep java | awk '{ print $2 }'`;do ls /proc/$x/fd|wc -l;done

#计算文件temp的第一列的值的和
awk '{s+=$1}END{print s}' temp

#查看最常用的命令和使用次数
history | awk '{if ($2 == "sudo") a[$3]++; else a[$2]++}END{for(i in a){print a[i] " " i}}' |  sort -rn | head

#查找某个时间戳的文件列表
cp -p `ls -l | awk '/Apr 14/ {print $NF}'` /usr/users/backup_dir

#格式化输出当前的进程信息
ps -ef | awk -v OFS="\n" '{ for (i=8;i<=NF;i++) line = (line ? line FS : "") $i; print NR ":", $1, $2, $7, line, ""; line = "" }'

#查看输入数据的特定位置的单个字符
echo "abcdefg"|awk 'BEGIN {FS="''"} {print $2}'

#打印行号
ls | awk '{print NR "\t" $0}'

#打印当前的ssh 客户端
netstat -tn | awk '($4 ~ /:22\s*/) && ($6 ~ /^EST/) {print substr($5, 0, index($5,":"))}'

#打印文件第一列不同值的行
awk '!array[$1]++' file.txt

#打印第二列唯一值
awk '{ a[$2]++ } END { for (b in a) { print b } }' file

#查看系统所有分区
awk '{if ($NF ~ "^[a-zA-Z].*[0-9]$" && $NF !~ "c[0-9]+d[0-9]+$" && $NF !~ "^loop.*") print "/dev/"$NF}'  /proc/partitions

#查看2到100所有质数
for num in `seq 2 100`;do if [ `factor $num|awk '{print $2}'` == $num ];then echo -n "$num ";fi done;echo

#查看第3到第6行
awk 'NR >= 3 && NR <= 6' /path/to/file

#逆序查看文件
awk '{a[i++]=$0} END {for (j=i-1; j>=0;) print a[j--] }'

#打印99乘法表
seq 9 | sed 'H;g' | awk -v RS='' '{for(i=1;i<=NF;i++)printf("%dx%d=%d%s", i, NR, i*NR, i==NR?"\n":"\t")}'
```

## 思维导图
![常用参数和常用内置变量](/images/linux/akw-1.jpg)













































