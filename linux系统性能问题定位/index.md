# Linux 系统性能问题定位

# 监控系统资源和进程工具
> top 命令

快捷键及使用场景
- 1：显示每个 CPU 核心的详细信息。当你需要查看系统中每个 CPU 核心的负载和使用情况时，可以按下 1 键。
- E：切换累计模式（Cumulative Mode）。在累计模式下，CPU 使用率和内存占用等指标会显示所有进程的累计值而不是单个进程的值。
- M：按内存使用排序。按下 M 键后，top 命令会按照内存使用量的大小对进程进行排序，并显示最高的内存使用进程。
- P：按 CPU 使用排序。按下 P 键后，top 命令会按照 CPU 使用率的大小对进程进行排序，并显示最高的 CPU 使用进程。

## 内存
内存相关名词释义

1）Working Set
Size(WSS)是指一个app保持正常运行所须的内存。比如一个应用在初始阶段申请了100G主存，在实际正常运行时每秒只需要50M，那么这里的50M就是一个WSS  
2）RES：resident memory usage。常驻内存  
3）RSS Resident Set Size 实际使用物理内存，（包含共享库占用的内存）  
4）CACHE，是一种特殊的内存，因为主内存速度不够快，用少量的特别快的但特别昂贵的内存来做缓存加速,就是cache。简单来说，buffer是即将要被写入磁盘的，而cache是被从磁盘中读出来的。  
5）SWAP 交换区间，内存不够用使用磁盘充当内存  

- 查看占用内存较高的10个进程
```shell script
ps -aux | sort -k4nr | head
```

- 查看进程占用的内存大小
```shell
# 通过端口获取进程号
netstat -anp|grep 9100
tcp6       0      0 :::9100                 :::*                    LISTEN      5505/node_exporter

# 通过进程id获取进程占用内存大小
cat /proc/5505/status|grep VmRSS
VmRSS:     31500 kB
```
注意：VmRSS对应的值就是物理内存占用

## IO
- 查看磁盘io及传递状态
```shell script
iostat -x 1 3
```

### 如何判断磁盘 io 性能达到瓶颈？
iostat 是一个用于监控系统磁盘IO使用情况的工具，可以显示每个磁盘的IO操作、IO延迟和数据吞吐量等信息。当磁盘IO达到瓶颈时，可以通过 iostat 来判断。

有以下几个指标可以帮助你判断磁盘IO是否达到瓶颈：

- %util：这是磁盘IO利用率的百分比，表示磁盘正在处理IO请求所占用的时间比例。当 %util 高于 80% 或 90% 时，磁盘IO可能已经达到瓶颈状态。

- await：这是平均每个IO请求在队列中等待被处理的时间，表示系统平均IO响应时间。当 await 较高(通常大于 10ms)时，说明磁盘IO负载过重，也可能意味着磁盘IO已达到瓶颈。

- svctm：这是平均每个IO请求的服务时间，表示磁盘处理IO请求的平均时间。当 svctm 较高时，系统处理IO请求的速度较慢，可能代表磁盘IO已经达到瓶颈状态。

总之，当磁盘IO利用率高、IO等待时间长或服务时间长时，磁盘IO可能已达到瓶颈状态。通常还需要综合考虑诸如系统性能需求、磁盘类型和配置等因素，才能确定是否对系统进行优化或升级。

### 查看磁盘io占用较高的几个进程
可以使用 iotop 命令来查看磁盘IO占用较高的几个进程
```shell
$ iotop -oP
Total DISK READ :     577.97 K/s | Total DISK WRITE :     115.06 K/s
Actual DISK READ:      64.22 K/s | Actual DISK WRITE:    1934.59 K/s
   PID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND                                                                                                                          
   574 be/4 root      577.97 K/s    0.00 B/s  0.00 %  8.68 % [xfsaild/dm-0]
 73282 be/4 root        0.00 B/s   45.49 K/s  0.00 %  0.47 % influxd
 53910 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.02 % [kworker/7:0]
 55873 be/4 systemd-    0.00 B/s    2.68 K/s  0.00 %  0.02 % mysqld
```

## 监控进程使用情况
pidstat 是一个用于监控进程资源使用情况的工具，可以提供有关 CPU、内存、磁盘IO、网络和上下文切换等方面的统计信息。以下是 pidstat 常用命令及其使用场景：

> 1.查看特定进程的资源使用情况：

- pidstat -p <pid>  
该命令可用于监控指定 PID 的进程的资源使用情况，包括 CPU 占用率、内存使用量、上下文切换次数、磁盘IO等。

> 2.实时监控整个系统中进程的资源使用情况：

- pidstat -r  
该命令可实时显示所有进程的内存使用情况，包括虚拟内存大小、物理内存大小、共享内存大小等。

> 3.监控进程的CPU使用情况：

- pidstat -u <interval> <times>  
通过设置 <interval> 和 <times> 参数，该命令可定期显示进程的 CPU 使用情况，包括用户空间和内核空间的 CPU 使用时间、CPU 使用百分比等。



pidstat 命令常用于性能分析和资源监控，可以帮助定位和解决系统性能问题。具体使用场景包括但不限于：
查找 CPU 密集型进程、监测内存泄漏、检测磁盘IO瓶颈、分析网络传输情况等。

# 最佳实践
## 手动清理缓存buffer/cache
1.清理pagecache(页面缓存)
```shell
echo 1 > /proc/sys/vm/drop_caches #或者 sysctl -w vm.drop_caches=1
```
2.清理目录缓存
```shell
echo 2 > /proc/sys/vm/drop_caches #或者 sysctl -w vm.drop_caches=2
```
3.清理pagecache、dentries和inodes
```shell
echo 3 > /proc/sys/vm/drop_caches #或者 sysctl -w vm.drop_caches=3
```
上面三种方式都是临时释放缓存的方法，想要永久释放缓存，需要在/etc/sysctl.conf文件中配置： vm.drop_caches=1/2/3,然后sysctl -p生效即可

温馨提示：  
上面操作在大多数情况下都不会对系统造成伤害，只会有助于释放不用的内存

### 查看僵尸进程并kill掉
查看
```shell
ps -ef | grep defunct
```
kill
```shell
kill -9 父进程号
```

在linux系统中，进程有如下几种状态，它们随时可能处于以上状态中的一种：

- D = 不可中断的休眠
- I = 空闲
- R = 运行中
- S = 休眠
- T = 被调度信号终止
- t = 被调试器终止
- Z = 僵尸状态

我们可以在命令终端中通过top命令来查看系统进程和它的当前状态。

## 查看进程占用的的物理内存大小指令
```shell
cat /proc/`ps -ef|grep grafana|grep -v grep|awk '{print $2}'`/status |grep VmRSS
```

指令分解
```shell
# 通过关键词xxx确认进程id，进程关键词如grafana
$ ps -ef|grep grafana|grep -v grep|awk '{print $2}'
118581
# 通过进程标识查看服务器物理内存占用
$ cat /proc/118581/status |grep VmRSS
VmRSS:    100788 kB
# 动态查看进程资源占用
top -p 118581

```

## 查看TCP连接数
```shell
netstat -n | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}'
```

# 总结
Linux 服务器性能资源排查思路
在排查 Linux 服务器性能问题时，可以按照以下思路进行操作：

1.监测系统资源使用情况：

- 使用命令 top 或 htop 实时监视 CPU、内存和交换空间的使用情况。
- 使用命令 df 查看磁盘空间使用情况。
- 使用命令 free 查看内存使用情况。
- 使用命令 iostat 查看磁盘IO使用情况。
- 使用命令 sar 查看系统整体的资源利用率。

2.检查进程和服务：

- 使用命令 ps 或 top 查看正在运行的进程和它们的资源占用情况。
- 使用命令 systemctl status 检查服务的状态。

3.定位高负载进程：

- 使用命令 top 或 htop 查看 CPU 使用率最高的进程，并观察其 PID 和资源消耗情况。
- 使用命令 pidstat 监测指定 PID 的进程的资源使用情况。

4.分析日志信息：

- 查看系统日志（如 /var/log/messages）以了解系统运行期间发生的错误或异常情况。
- 查看应用程序日志，特别是与性能相关的日志，以寻找可能的瓶颈或错误。

5.检查网络连接和流量：

- 使用命令 netstat 或 ss 查看网络连接状态。
- 使用命令 iftop 或 nethogs 实时监测网络流量和使用情况。

6.进行性能调优：

- 调整系统内核参数，如文件句柄数、内存分配等，以提高性能。
- 优化关键应用程序的配置，如数据库、Web 服务器等。

7.使用性能分析工具：

- 使用工具如 perf、strace、tcpdump 等进行更深入的性能分析和故障排查。

这些步骤可以帮助你定位服务器的性能问题，并找出潜在的瓶颈或异常。根据具体的问题，你可能需要进一步深入研究和分析，以找到解决方案。同时，及时备份数据是必要的，以免在调优过程中发生数据丢失或其他意外。

