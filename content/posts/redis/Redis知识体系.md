---
title: "Redis 知识体系"
subtitle: ""
date: 2022-04-19T12:06:37+08:00
lastmod: 2023-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Redis"]
tags: ["Redis"]
---
## 基本概念
Redis（REmote DIctionary Service）是一个开源的键值对数据库服务器。

Redis是一个开源（BSD许可），内存存储的数据结构服务器，可用作数据库，高速缓存和消息队列代理。它支持字符串、哈希表、列表、集合、有序集合，位图，hyperloglogs等数据类型。内置复制、Lua脚本、LRU收回、事务以及不同级别磁盘持久化功能，同时通过Redis Sentinel提供高可用，通过Redis Cluster提供自动分区。

![redis explained](/images/redis/redis-explained.png "redis explained")


### 使用场景
1. 数据库
2. 缓存
3. MQ

### Redis优势
- 性能极高
- 数据类型丰富，单键值对最大支持512M大小的数据
- 简单易用，支持所有主流编程语言
- 支持数据持久化，主从复制，哨兵模式等高可用特性
### 竞品对比
Memcached  vs Redis

![redis-1](/images/redis/redis-vs-memcached.png "redis-vs-memcached")


## 安装配置
这里安装的环境是Linux Ubuntu 22.04.4 LTS 操作系统，其它安装方式见[安装介绍](https://redis.ac.cn/docs/install/install-redis/)
```shell
sudo apt install lsb-release curl gpg

curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

sudo apt-get update
sudo apt-get install redis
```

Linux Centos 安装方式
```shell
$ yum install redis -y
```
### 启动 Redis
```
root@k8s-master01:~/redis# redis-server
9204:C 17 Jul 2024 17:47:22.376 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
9204:C 17 Jul 2024 17:47:22.377 * Redis version=7.2.5, bits=64, commit=00000000, modified=0, pid=9204, just started
9204:C 17 Jul 2024 17:47:22.377 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
9204:M 17 Jul 2024 17:47:22.377 * Increased maximum number of open files to 10032 (it was originally set to 1024).
9204:M 17 Jul 2024 17:47:22.377 * monotonic clock: POSIX clock_gettime
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 7.2.5 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                  
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 9204
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           https://redis.io       
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

9204:M 17 Jul 2024 17:47:22.378 # Warning: Could not create server TCP listening socket *:6379: bind: Address already in use
9204:M 17 Jul 2024 17:47:22.378 # Failed listening on port 6379 (tcp), aborting.
root@k8s-master01:~/redis# netstat -anp|grep 6379
tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN      8799/redis-server 1 
tcp6       0      0 ::1:6379                :::*                    LISTEN      8799/redis-server 1 

```
客户端命令连接redis 服务，并通过命令操作redis
```
root@k8s-master01:~/redis# redis-cli
127.0.0.1:6379> set key1 value1
OK
127.0.0.1:6379> get key1
"value1"
```
备注：如果设置的键中有中文查看时是返回的二进制，需要使用`redis-cli --raw` 登录

### 图形化工具RedisInsight
### 使用方式
- CLI 命令行
- API (Application Programming interface)
- GUI (Graphical User Interface)
## 常用命令
redis 的[使用命令](https://redis.io/docs/latest/commands/)很多,这里重点介绍几种常用的命令
### Key Commands
```
SET key value #: set a key to a value
GET key #: get the value of a key (string only)
DEL key #: delete a key
EXPIRE key seconds #: set a timeout on a key
TTL key #: get the remaining time to live of a key
KEYS pattern #: find all keys matching a pattern
SCAN cursor [MATCH pattern] [COUNT count] [TYPE type] #: iterate over keys with a cursor
```
### String Commands
```
APPEND key value #: append a value to a key
STRLEN key #: get the length of the value of a key
INCR key #: increment the value of a key by 1
DECR key #: decrement the value of a key by 1
INCRBY key increment #: increment the value of a key by a specified amount
DECRBY key decrement #: decrement the value of a key by a specified amount
```
### Hash Commands
```
HSET key field value #: set a field in a hash to a value
HGET key field #: get the value of a field in a hash
HDEL key field [field …] #: delete one or more fields from a hash
HGETALL key #: get all fields and values from a hash
HKEYS key #: get all fields from a hash
HVALS key #: get all values from a hash
HEXISTS key field #: check if a field exists in a hash
HLEN key #: get the number of fields in a hash
```
hash 命令实际使用方式
```
# Initialize the hash
HMSET user name "John Doe" age 30 email "lushuan2071@126.com"

# Get the value of a field
HGET user name   # Output: "John Doe"

# Get the values of multiple fields
HMGET user name age   # Output: 1) "John Doe" 2) "30"

# Get all fields and values
HGETALL user   # Output: 1) "name" 2) "John Doe" 3) "age" 4) "30" 5) "email" 6) "johndoe@example.com"

# Get all fields
HKEYS user   # Output: 1) "name" 2) "age" 3) "email"

# Get all values
HVALS user   # Output: 1) "John Doe" 2) "30" 3) "johndoe@example.com"

# Get the number of fields
HLEN user   # Output: 3

# Modify a field
HSET user age 31
HGET user age   # Output: "31"

# Increment a field
HINCRBY user age 1
HGET user age   # Output: "32"

# Delete a field
HDEL user email
HKEYS user   # Output: 1) "name" 2) "age"

```
### List Commands
```
LPUSH key value [value …] #: prepend one or more values to a list
RPUSH key value [value …] #: append one or more values to a list
LPOP key #: remove and return the first element of a list
RPOP key #: remove and return the last element of a list
LINDEX key index #: get the element at a specified index in a list
LLEN key #: get the length of a list
LRANGE key start stop #: get a range of elements from a list
LREM key count value #: remove a specified number of occurrences of a value from a list
```
list 使用实例
```
# Initialize the list
LPUSH numbers 3 2 1

# Get the length of the list
LLEN numbers   # Output: 3

# Get an element by index
LINDEX numbers 1   # Output: "2"

# Get a range of elements
LRANGE numbers 0 1   # Output: 1) "1" 2) "2"

# Append an element
RPUSH numbers 4
LRANGE numbers 0 -1   # Output: 1) "1" 2) "2" 3) "3" 4) "4"

# Pop an element
LPOP numbers   # Output: "1"
RPOP numbers   # Output: "4"
LRANGE numbers 0 -1   # Output: 1) "2" 2) "3"

# Insert an element
LINSERT numbers BEFORE 2 1
LRANGE numbers 0 -1   # Output: 1) "1" 2) "2" 3) "3"

# Remove an element
LREM numbers 1 2
LRANGE numbers 0 -1   # Output: 1) "1" 2) "3"

# Trim the list
LTRIM numbers 0 0
LRANGE numbers 0 -1   # Output: 1) "1"
```
### Set Commands
```
SADD key member [member …] #: add one or more members to a set
SREM key member [member …] #: remove one or more members from a set
SMEMBERS key #: get all members of a set
SISMEMBER key member #: check if a member exists in a set
SINTER key [key …] #: get the intersection of sets
SUNION key [key …] #: get the union of sets
```
Set 使用实例，注意，由于set不是有序数据类型，所以运行以下命令时，项的顺序可能会有所不同:
```
# Initialize the set
SADD fruits apple banana orange

# Get the number of elements in the set
SCARD fruits   # Output: 3

# Check if an element is a member of the set
SISMEMBER fruits apple   # Output: 1 (true)

# Get all members of the set
SMEMBERS fruits   # Output: 1) "apple" 2) "banana" 3) "orange"

# Remove an element from the set
SREM fruits banana
SMEMBERS fruits   # Output: 1) "apple" 2) "orange"

# Add multiple elements to the set
SADD fruits pear peach
SMEMBERS fruits   # Output: 1) "apple" 2) "orange" 3) "pear" 4) "peach"

# Get the intersection of multiple sets
SADD fruits2 orange kiwi
SINTER fruits fruits2   # Output: 1) "orange"

# Get the union of multiple sets
SUNION fruits fruits2   # Output: 1) "apple" 2) "orange" 3) "pear" 4) "peach" 5) "kiwi"

# Get a random element from the set
SRANDMEMBER fruits   # Output: "pear"

```
### Sorted Set Commands
```
ZADD key score member [score member …] #: add one or more members with scores to a sorted set
ZREM key member [member …] #: remove one or more members from a sorted set
ZRANGE key start stop [WITHSCORES] #: The order of elements is from the lowest to the highest score. Elements with the same score are ordered lexicographically.
ZREVRANGE key start stop [WITHSCORES] #: Returns the specified range of elements in the sorted set stored at key. The elements are considered to be ordered from the highest to the lowest score.
ZSCORE key member #: get the score of a member in a sorted set
```
实例
```
# Initialize the sorted set
ZADD scores 90 "Alice" 85 "Bob" 95 "Charlie"

# Get the number of elements in the sorted set
ZCARD scores   # Output: 3

# Get the rank of an element
ZRANK scores "Charlie"   # Output: 2

# Get the score of an element
ZSCORE scores "Bob"   # Output: 85

# Get a range of elements by rank
ZRANGE scores 0 1   # Output: 1) "Bob" 2) "Alice"

# Get a range of elements by score
ZRANGEBYSCORE scores 90 95   # Output: 1) "Alice" 2) "Charlie"

# Increment the score of an element
ZINCRBY scores 10 "Alice"
ZSCORE scores "Alice"   # Output: 100

# Remove an element from the sorted set
ZREM scores "Bob"
ZRANGE scores 0 -1   # Output: 1) "Charlie" 2) "Alice"

# Get the highest-scoring element(s)
ZREVRANGE scores 0 0   # Output: 1) "Alice"

```

## 十大数据模型
redis 初始化不同的数据类型，更多数据类型可以查看[官网介绍](https://redis.io/docs/latest/develop/data-types/)

![redis-datatype](/images/redis/redis-datatype.png "redis-datatype")

```
# String
SET key value
SET name "John Doe"

# List
RPUSH key element [element ...]
RPUSH numbers 1 2 3 4 5

# Set
SADD key member [member ...]
SADD colors red green blue

# Hash
HSET key field value [field value ...]
HSET user id 1 name "John Doe" email "john.doe@example.com"

# Sorted set
ZADD key score member [score member ...]
ZADD scores 90 "John Doe" 80 "Jane Doe" 95 "Bob Smith"

```
### 5种基本数据类型
首先对redis来说，所有的key（键）都是字符串。我们在谈基础数据结构时，讨论的是存储值的数据类型，主要包括常见的5种数据类型，分别是：String、List、Set、Zset、Hash。

![redis-datatype](/images/redis/db-redis-ds-1.jpg "redis datatype common in use")

| 结构类型         | 结构存储的值                               | 结构的读写能力                                               |
| ---------------- | ------------------------------------------ | ------------------------------------------------------------ |
| **String字符串** | 可以是字符串、整数或浮点数                 | 对整个字符串或字符串的一部分进行操作；对整数或浮点数进行自增或自减操作； |
| **List列表**     | 一个链表，链表上的每个节点都包含一个字符串 | 对链表的两端进行push和pop操作，读取单个或多个元素；根据值查找或删除元素； |
| **Set集合**      | 包含字符串的无序集合                       | 字符串的集合，包含基础的方法有看是否存在添加、获取、删除；还包含计算交集、并集、差集等 |
| **Hash散列**     | 包含键值对的无序散列表                     | 包含方法有添加、获取、删除单个元素                           |
| **Zset有序集合** | 和散列一样，用于存储键值对                 | 字符串成员与浮点数分数之间的有序映射；元素的排列顺序由分数的大小决定；包含方法有添加、获取、删除单个元素以及根据分值范围或成员来获取元素 |

#### String字符串
String是redis中最基本的数据类型，一个key对应一个value。

String类型是二进制安全的，意思是 redis 的 string 可以包含任何数据。如数字，字符串，jpg图片或者序列化的对象。

#### List列表
Redis中的List其实就是链表（Redis用双端链表实现List）。

使用List结构，我们可以轻松地实现最新消息排队功能（比如新浪微博的TimeLine）。List的另一个应用就是消息队列，可以利用List的 PUSH 操作，将任务存放在List中，然后工作线程再用 POP 操作将任务取出进行执行。
#### Set 集合
Redis 的 Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。

Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。

#### Hash 散列
Redis hash 是一个 string 类型的 field（字段） 和 value（值） 的映射表，hash 特别适合用于存储对象。

#### Zset有序集合
Redis 有序集合和集合一样也是 string 类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个 double 类型的分数。redis 正是通过分数来为集合中的成员进行从小到大的排序。

有序集合的成员是唯一的, 但分数(score)却可以重复。有序集合是通过两种数据结构实现：



### 5种高级数据类型
- 消息队列 Stream
- 地理空间 Geospatial
- HyperLogLog 是一种用来做基数统计的算法
- Bitmap 即位图数据结构，都是操作二进制位来进行记录，只有0 和 1 两个状态
- 位域 Bitfield

## 事务
### 什么是 Redis 事务
Redis 事务的本质是一组命令的集合。事务支持一次执行多个命令，一个事务中所有命令都会被序列化。在事务执行过程，会按照顺序串行化执行队列中的命令，其他客户端提交的命令请求不会插入到事务执行命令序列中。总结说：redis事务就是一次性、顺序性、排他性的执行一个队列中的一系列命令。

### Redis事务相关命令和使用
- MULTI ：开启事务，redis会将后续的命令逐个放入队列中，然后使用EXEC命令来原子化执行这个命令系列。
- EXEC：执行事务中的所有操作命令。
- DISCARD：取消事务，放弃执行事务块中的所有命令。
- WATCH：监视一个或多个key,如果事务在执行前，这个key(或多个key)被其他命令修改，则事务被中断，不会执行事务中的任何命令。
- UNWATCH：取消WATCH对所有key的监视。
### 实例
```
127.0.0.1:6379> MULTI 
OK
127.0.0.1:6379(TX)> SET key1 1
QUEUED
127.0.0.1:6379(TX)> SET key2 2
QUEUED
127.0.0.1:6379(TX)> EXEC
OK
OK
127.0.0.1:6379> GET key1
1

```

## 数据持久化
为了防止数据丢失以及服务重启时能够恢复数据，Redis支持数据的持久化，主要分为两种方式，分别是RDB和AOF; 当然实际场景下还会使用这两种的混合模式。

- RDB(Redis Database)
- AOF(Append Only File)

### RDB
RDB 就是 Redis DataBase 的缩写，中文名为快照/内存快照，RDB持久化是把当前进程数据生成快照保存到磁盘上的过程，由于是某一时刻的快照，那么快照中的值要早于或者等于内存中的值。

可以理解为数据的快照机制，是某一个时间点上数据的完整副本，就是定时或者手动的给数据做一下快照，所以触发方式有两种，定时或者手动触发

在某个快照时间点修改过的数据就会丢失掉，所以RDB 更适合用来做备份
#### Redis中默认的周期新设置
```
# 周期性执行条件的设置格式为
save <seconds> <changes>

# 默认的设置为：
save 900 1
save 300 10
save 60 10000

# 以下设置方式为关闭RDB快照功能
save ""
```
#### 实例
`xxd` 是linux 查看二进制或者16进制的文件内容的命令，rdb备份文件默认目录是`/var/lib/redis/`
```
root@k8s-master01:/etc/redis# redis-cli 
127.0.0.1:6379> set name lushuan
OK
127.0.0.1:6379> save
OK
127.0.0.1:6379> exit
root@k8s-master01:/etc/redis# cd /var/lib/redis/
root@k8s-master01:/var/lib/redis# ls
dump.rdb
root@k8s-master01:/var/lib/redis# xxd dump.rdb 
00000000: 5245 4449 5330 3031 31fa 0972 6564 6973  REDIS0011..redis
00000010: 2d76 6572 0537 2e32 2e35 fa0a 7265 6469  -ver.7.2.5..redi
00000020: 732d 6269 7473 c040 fa05 6374 696d 65c2  s-bits.@..ctime.
00000030: ffd0 9766 fa08 7573 6564 2d6d 656d c220  ...f..used-mem. 
00000040: 7812 00fa 0861 6f66 2d62 6173 65c0 00fe  x....aof-base...
00000050: 00fb 0600 0007 636f 7572 7365 321b 4859  ......course2.HY
00000060: 4c4c 0100 0000 0300 0000 0000 0000 6821  LL............h!
00000070: 8048 6c80 4366 844c 0600 0663 6f75 7273  .Hl.Cf.L...cours
00000080: 6524 4859 4c4c 0100 0000 0400 0000 0000  e$HYLL..........
00000090: 0080 4303 8442 5284 4af7 8057 cf80 486c  ..C..BR.J..W..Hl
000000a0: 8043 6684 4c06 0004 6e61 6d65 076c 7573  .Cf.L...name.lus
000000b0: 6875 616e 0006 7265 7375 6c74 2448 594c  huan..result$HYL
000000c0: 4c01 0000 0006 0000 0000 0000 0043 0384  L............C..
000000d0: 4252 844a f780 57cf 8048 6c80 4366 844c  BR.J..W..Hl.Cf.L
000000e0: 0600 046b 6579 32c0 0200 046b 6579 31c0  ...key2....key1.
000000f0: 01ff 7d49 6715 3957 c40c                 ..}Ig.9W..
root@k8s-master01:/var/lib/redis# 

```
#### 触发方式
- save命令：阻塞当前Redis服务器，直到RDB过程完成为止，对于内存 比较大的实例会造成长时间阻塞，线上环境不建议使用
- bgsave命令：Redis进程执行fork操作创建子进程，RDB持久化过程由子 进程负责，完成后自动结束。阻塞只发生在fork阶段，一般时间很短

#### RDB 优缺点
- 优点
  - RDB文件是某个时间节点的快照，默认使用LZF算法进行压缩，压缩后的文件体积远远小于内存大小，适用于备份、全量复制等场景；
  - Redis加载RDB文件恢复数据要远远快于AOF方式；

- 缺点
  - RDB方式实时性不够，无法做到秒级的持久化；
  - 每次调用bgsave都需要fork子进程，fork子进程属于重量级操作，频繁执行成本较高；
  - RDB文件是二进制的，没有可读性，AOF文件在了解其结构的情况下可以手动修改或者补全；
  - 版本兼容RDB文件问题；

针对RDB不适合实时持久化的问题，Redis提供了AOF持久化方式来解决


### AOF
Redis是“写后”日志，Redis先执行命令，把数据写入内存，然后才记录日志。日志里记录的是Redis收到的每一条命令，这些命令是以文本形式保存。PS: 大多数的数据库采用的是写前日志（WAL），例如MySQL，通过写前日志和两阶段提交，实现数据和逻辑的一致性。

为了解决RDB 在快照时阻塞服务状态，即便是RDB 的bgsave fork一个子进程备份时时间很短，不过在这很短的时间内redis 还是不能处理任何请求，无法做到秒级的快照，为了解决这个问题，可以使用AOF(追加文件)的持久化方式。

当开启AOF redis 重启后会先从磁盘上读取AOF 持久化的数据至内存中。

虽然 bgsave 执行时不阻塞主线程，但是，如果频繁地执行全量快照，也会带来两方面的开销：
-  一方面，频繁将全量数据写入磁盘，会给磁盘带来很大压力，多个快照竞争有限的磁盘带宽，前一个快照还没有做完，后一个又开始做了，容易造成恶性循环。
-  另一方面，bgsave 子进程需要通过 fork 操作从主线程创建出来。虽然，子进程在创建后不会再阻塞主线程，但是，fork 这个创建过程本身会阻塞主线程，而且主线程的内存越大，阻塞时间越长。如果频繁 fork 出 bgsave 子进程，这就会频繁阻塞主线程了。

那么，有什么其他好方法吗？此时，我们可以做增量快照，就是指做了一次全量快照后，后续的快照只对修改的数据进行快照记录，这样可以避免每次全量快照的开销。这个比较好理解。

但是它需要我们使用额外的元数据信息去记录哪些数据被修改了，这会带来额外的空间开销问题。那么，还有什么方法既能利用 RDB 的快速恢复，又能以较小的开销做到尽量少丢数据呢？
这个时候就需要引入RDB和AOF的混合方式解决方案了。
#### redis.conf中配置AOF
默认情况下，Redis是没有开启AOF的，可以通过配置redis.conf文件来开启AOF持久化，关于AOF的配置如下：
```
# appendonly参数开启AOF持久化
appendonly no

# AOF持久化的文件名，默认是appendonly.aof
appendfilename "appendonly.aof"

# AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的
dir ./

# 同步策略
# appendfsync always
appendfsync everysec
# appendfsync no

# aof重写期间是否同步
no-appendfsync-on-rewrite no

# 重写触发配置
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 加载aof出错如何处理
aof-load-truncated yes

# 文件重写策略
aof-rewrite-incremental-fsync yes
```
以下是Redis中关于AOF的主要配置信息：
- appendonly：默认情况下AOF功能是关闭的，将该选项改为yes以便打开Redis的AOF功能。- - -- --- appendfilename：这个参数项很好理解了，就是AOF文件的名字。
- appendfsync：这个参数项是AOF功能最重要的设置项之一，主要用于设置“真正执行”操作命令向AOF文件中同步的策略。

### RDB和AOF混合方式
Redis 4.0 中提出了一个混合使用 AOF 日志和内存快照的方法。简单来说，内存快照以一定的频率执行，在两次快照之间，使用 AOF 日志记录这期间的所有命令操作。

这样一来，快照不用很频繁地执行，这就避免了频繁 fork 对主线程的影响。而且，AOF 日志也只用记录两次快照间的操作，也就是说，不需要记录所有操作了，因此，就不会出现文件过大的情况了，也可以避免重写开销。


## 主从复制
我们知道要避免单点故障，即保证高可用，便需要冗余（副本）方式提供集群服务。而Redis 提供了主从库模式，以保证数据副本的一致，主从库之间采用的是读写分离的方式。本文主要阐述Redis的主从复制。

主从复制，是指将一台Redis服务器的数据，复制到其他的Redis服务器。前者称为主节点(master)，后者称为从节点(slave)；数据的复制是单向的，只能由主节点到从节点。

主从复制的作用主要包括：
- 数据冗余：主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。
- 故障恢复：当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复；实际上是一种服务的冗余。
- 负载均衡：在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务（即写Redis数据时应用连接主节点，读Redis数据时应用连接从节点），分担服- 务器负载；尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量。
- 高可用基石：除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是Redis高可用的基础。

主从库之间采用的是读写分离的方式。
- 读操作：主库、从库都可以接收；
- 写操作：首先到主库执行，然后，主库将写操作同步给从库。

![redis explained](/images/redis/db-redis-copy-1.png "redis 读写分离")


这里在同一台主机上演示，备份`redis.conf` 为`redis-6380.conf`，修改以下四个配置项
```
port 6380
...
...
pidfile /var/run/reids_6380.pid
...
...
dbfilename dump-6380.rdb # 在同一台主机做测试时使用
...
...
replicaof 127.0.0.1 6379
```
### 启动从节点
```
redis-server redis-6380.conf
```

客户端连接从节点,可以看到角色为从节点，这样表示主从配置就成功了。
```

root@k8s-master01:/etc/redis# redis-cli -p 6380
127.0.0.1:6379> info replication
# Replication
role:slave
...
```

## 高可用：哨兵机制（Redis Sentinel）
在上文主从复制的基础上，如果主节点出现故障该怎么办呢？ 在 Redis 主从集群中，哨兵机制是实现主从库自动切换的关键机制，它有效地解决了主从复制模式下故障转移的问题。

### 哨兵机制（Redis Sentinel）
Redis Sentinel，即Redis哨兵，在Redis 2.8版本开始引入。哨兵的核心功能是主节点的自动故障转移。

哨兵集群监控的逻辑图：

![redis explained](/images/redis/db-redis-sen-1.png "redis 哨兵机制")

哨兵实现了什么功能呢？下面是Redis官方文档的描述：

- 监控（Monitoring）：哨兵会不断地检查主节点和从节点是否运作正常。
- 自动故障转移（Automatic failover）：当主节点不能正常工作时，哨兵会开始自动故障转移操作，它会将失效主节点的其中一个从节点升级为新的主节点，并让其他从节点改为复制新的主节点。
- 配置提供者（Configuration provider）：客户端在初始化时，通过连接哨兵来获得当前Redis服务的主节点地址。
- 通知（Notification）：哨兵可以将故障转移的结果发送给客户端。

其中，监控和自动故障转移功能，使得哨兵可以及时发现主节点故障并完成转移；而配置提供者和通知功能，则需要在与客户端的交互中才能体现。

### 哨兵集群的组建
上图中哨兵集群是如何组件的呢？哨兵实例之间可以相互发现，要归功于 Redis 提供的 pub/sub 机制，也就是发布 / 订阅机制。

### 启动哨兵节点

`sentinel.conf`,模拟还是当前主机下
```
sentinel monitor master 127.0.0.1 6379
```
启动哨兵节点，会以哨兵模式进行启动
```
redis-sentinel sentinel.conf
```
注意：生产环境需要三个哨兵节点

## 参考
- https://redis.io/docs/latest/develop/
- https://www.redis.net.cn/
- [Redis 设计与实现](https://huangz.works/redisbook1e/index.html#id9)
- https://datmt.com/backend/redis-command-cheatsheet/
- https://pdai.tech/md/db/nosql-redis/db-redis-overview.html







