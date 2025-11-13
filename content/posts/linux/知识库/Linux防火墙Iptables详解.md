---
title: "Linux 防火墙iptables详解"
subtitle: ""
date: 2024-11-19T12:06:37+08:00
lastmod: 2024-11-20T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Linux"]
tags: ["Linux","防火墙"]
---
iptables 是运维中的重点但不算是特别的难点，是网站访问的大门，本篇记录对 iptables 的常用梳理

## 概述
- 封端口
- 封IP
- 实现Nat上网
  - 共享上网
  - 端口映射(端口转发)，ip 映射

## 核心解释
- iptables适用linux所有发行版，yum安装方式为 "yum install iptables-services"
- iptables基于linux netfilter, Netfilter是Linux 2.4.x引入的一个子系统，它作为一个通用的、抽象的框架，提供一整套的hook函数的管理机制，使得诸如数据包过滤、网络地址转换(NAT)和基于协议类型的连接跟踪成为了可能
- linux 内核中，netfilter， nf_* 的配置都会影响netfilter的性能
- iptables结构：iptables -> Tables -> Chains -> Rules. 即tables由chains组成，而chains由rules组成
- iptables 支持表类型有 filter （默认）， nat， mangle， raw  ， 优先级  raw > mangle > nat > filter
- iptables 的 nat 功能依赖linux内核的net.ipv4.ip_forward， net.ipv6.ip_forward
- iptables 的 nat 功能包含snat，dnat，其中snat仅用于postrouting
    - MASQUERADE是SNAT的一个特例，主要用于动态返回snat地址
- 防火墙实例: (/etc/sysconfig/iptables）

## iptables和netfilter的关系
Iptables和netfilter的关系是一个很容易让人搞不清的问题。很多的知道iptables却不知道netfilter。其实iptables只是Linux防火墙的管理工具而已，位于/sbin/iptables。真正实现防火墙功能的是netfilter，它是Linux内核中实现包过滤的内部结构。

## iptables 管理
安装了`iptables-service`后方便通过systemd管理iptables服务
```shell
$ sudo yum install iptables-services
$ sudo rpm -ql iptables-services
/etc/sysconfig/ip6tables
/etc/sysconfig/iptables    # 防火墙配置文件
/usr/lib/systemd/system/ip6tables.service
/usr/lib/systemd/system/iptables.service   # 防火墙配置文件(命令) systemctl start iptables
...

$ sudo rpm -ql iptables
...
/usr/sbin/ip6tables
/usr/sbin/ip6tables-restore
/usr/sbin/ip6tables-save
/usr/sbin/iptables       # iptables 命令 添加/删除/查看 规则(4表5链)
/usr/sbin/iptables-restore  # 恢复
/usr/sbin/iptables-save   # iptables规则 输出(保存)
/usr/sbin/xtables-multi  
...
```

注意: 使用`iptables` 记得将`firewalld` 关闭掉`systemctl disable firewalld`

防火墙激活  
防护墙是默认嵌入到linux内核中，但是没有默认激活,`lsmod`查看内核中加载了哪些模块
```shell
$ sudo lsmod|egrep "iptables|nat|filter"
xt_nat                 12681  4 
nf_nat_masquerade_ipv4    13463  1 ipt_MASQUERADE
iptable_nat            12875  1 
nf_nat_ipv4            14115  1 iptable_nat
nf_nat                 26583  3 nf_nat_ipv4,xt_nat,nf_nat_masquerade_ipv4
nf_conntrack          143411  6 nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4
iptable_filter         12810  1 
br_netfilter           22256  0 
bridge                155432  1 br_netfilter
ip_tables              27126  2 iptable_filter,iptable_nat
libcrc32c              12644  3 xfs,nf_nat,nf_conntrack

```
临时加载和永久加载
```shell
# 1. 写入到开机启动
$ modprobe iptable_filter
$ modprobe iptable_nat
$ modprobe iptable_mangle
$ modprobe iptable_raw
$ modprobe nf_conntrack
$ modprobe nf_conntrack_ipv4
$ modprobe nf_nat
# 验证是否加载成功
$ lsmod | grep iptable
$ lsmod | grep nf_conntrack

# 2. 永久
$ cat > /etc/modules-load.d/iptables.conf <<EOF
iptable_filter
iptable_nat
iptable_mangle
iptable_raw
nf_conntrack
nf_conntrack_ipv4
nf_nat
EOF
# 验证是否加载成功
$ sudo lsmod|egrep "iptables|nat|filter"
```

查看指定表中的规则
```shell
# 1. 默认是查看filter 表
$ iptables -nL
# 2. 查看nat 表的规则
$ iptables -t nat -nL
```

示例
```shell
$ iptables -nL
Chain INPUT (policy ACCEPT) # 链默认规则
target     prot opt source               destination         
ACCEPT     tcp  --  192.168.47.0/24      0.0.0.0/0            tcp dpt:30181 # 规则

Chain FORWARD (policy ACCEPT) # 链默认规则
target     prot opt source               destination         
DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0   # 规则

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination  
```


## iptables传输数据包的过程
1. 当一个数据包进入网卡时，它首先进入PREROUTING链，内核根据数据包目的IP判断是否需要转送出去。
2. 如果数据包就是进入本机的，它就会沿着图向下移动，到达INPUT链。数据包到了INPUT链后，任何进程都会收到它。本机上运行的程序可以发送数据包，这些数据包会经过OUTPUT链，然后到达POSTROUTING链输出。
3. 如果数据包是要转发出去的，且内核允许转发，数据包就会如图所示向右移动，经过FORWARD链，然后到达POSTROUTING链输出。

![iptables-process](/images/linux/iptables-process.png "iptables传输数据包的过程")

## 4表-5链-规则

### 防火墙的种类及使用说明
- 硬件：整个企业入口
  - 三层路由：H3C 华为 Cisco(思科)
  - 防护墙：深信服
- 软件：开源软件 网站内部 封ip
  - iptables 写入到Linux 内核中 以后服务docker 工作在4层
  - firewalld C7
  - nftables C8
  - ufw(ubuntu fire wall) Ubuntu
- 云防火墙(公有云)
  - 阿里云
    - 安全组(封ip,封端口)
    - NAT网关(共享上网，端口映射)
    - waf应用防火墙
- waf防火墙(应用防火墙，处理7层的攻击)SQL注入

名词简要熟悉
- 容器
- 表：存放链的容器，防火墙最大概念
- 链：存放规则的容器
- 规则：准许或拒绝规则，未来写的防火墙条件就是各种防火墙规则

详细解释：
- 表（tables）提供特定的功能，iptables内置了4个表，即filter表、nat表、mangle表和raw表，分别用于实现包过滤，网络地址转换、包重构(修改)和数据跟踪处理。
- 链（chains）是数据包传播的路径，每一条链其实就是众多规则中的一个检查清单，每一条链中可以有一条或数条规则。当一个数据包到达一个链时，iptables就会从链中第一条规则开始检查，看该数据包是否满足规则所定义的条件。如果满足，系统就会根据该条规则所定义的方法处理该数据包；否则iptables将继续检查下一条规则，如果该数据包不符合链中任一条规则，iptables就会根据该链预先定义的默认策略来处理数据包。
- Iptables采用“表”和“链”的分层结构。在REHL4中是三张表五个链。现在REHL5、6成了四张表五个链了，不过多出来的那个表用的也不太多，所以基本还是和以前一样。下面罗列一下这四张表和五个链。注意一定要明白这些表和链的关系及作用。

![iptables-四表五链](/images/linux/iptables-四表五链.png "四表五链")

### 规则表
1. filter表——三个链：INPUT、FORWARD、OUTPUT  
作用：过滤数据包  内核模块：iptables_filter.  
2. Nat表——三个链：PREROUTING、POSTROUTING、OUTPUT  
作用：用于网络地址转换（IP、端口） 内核模块：iptable_nat  
3. Mangle表——五个链：PREROUTING、POSTROUTING、INPUT、OUTPUT、FORWARD  
作用：修改数据包的服务类型、TTL、并且可以配置路由实现QOS内核模块：iptable_mangle(别看这个表这么麻烦，咱们设置策略时几乎都不会用到它)  
4. Raw表——两个链：OUTPUT、PREROUTING  
作用：决定数据包是否被状态跟踪机制处理 内核模块：iptable_raw  

备注：常用的表有filter、Nat这两个表

### 规则链
1. INPUT——进来的数据包应用此规则链中的策略
2. OUTPUT——外出的数据包应用此规则链中的策略
3. FORWARD——转发数据包时应用此规则链中的策略
4. PREROUTING——对数据包作路由选择前应用此链中的规则  
（记住！所有的数据包进来的时侯都先由这个链处理）
5. POSTROUTING——对数据包作路由选择后应用此链中的规则  
（所有的数据包出来的时侯都先由这个链处理）

备注：常用的规则链有INPUT、OUTPUT、FORWARD

## 四表五链详解
`iptables` 是 Linux 系统中用于配置防火墙规则的核心工具，基于 **Netfilter** 框架实现。它的核心逻辑围绕 **四表五链** 展开，用于控制网络数据包的传输和处理流程。

---

### 一、四表（Tables）
四表是规则的分类容器，每个表负责不同的功能：

1. **filter 表**（默认表）  
   - **功能**：数据包的过滤（允许/拒绝）。  
   - **应用场景**：控制访问权限（如禁止某个 IP 访问）。  
   - 关联链：`INPUT`、`FORWARD`、`OUTPUT`。

2. **nat 表**（Network Address Translation）  
   - **功能**：网络地址转换（修改源/目标 IP 或端口）。  
   - **应用场景**：端口转发（DNAT）、共享上网（SNAT）。  
   - 关联链：`PREROUTING`（DNAT）、`POSTROUTING`（SNAT）、`OUTPUT`。

3. **mangle 表**  
   - **功能**：修改数据包内容（如 TTL、TOS 字段）或标记数据包。  
   - **应用场景**：高级流量控制（如 QoS 标记）。  
   - 关联链：所有五链（`PREROUTING`、`INPUT`、`FORWARD`、`OUTPUT`、`POSTROUTING`）。

4. **raw 表**  
   - **功能**：绕过连接跟踪（Connection Tracking）。  
   - **应用场景**：提高性能（如禁用某些连接的跟踪）。  
   - 关联链：`PREROUTING`、`OUTPUT`。

---

### 二、五链（Chains）
五链是数据包在传输过程中经过的“检查点”，决定了规则生效的时机：

1. **PREROUTING**  
   - **触发时机**：数据包进入网卡后，**路由决策之前**。  
   - **常用表**：`nat`（DNAT）、`mangle`、`raw`。

2. **INPUT**  
   - **触发时机**：数据包目标是**本机**（如访问本地服务）。  
   - **常用表**：`filter`（允许/拒绝）、`mangle`。

3. **FORWARD**  
   - **触发时机**：数据包需要**转发到其他设备**（如路由器）。  
   - **常用表**：`filter`（是否允许转发）、`mangle`。

4. **OUTPUT**  
   - **触发时机**：本机生成的**出站数据包**（如本机发起的请求）。  
   - **常用表**：`filter`、`nat`、`mangle`、`raw`。

5. **POSTROUTING**  
   - **触发时机**：数据包**离开网卡前**，路由决策之后。  
   - **常用表**：`nat`（SNAT）、`mangle`。

---

### 三、表和链的对应关系
| 表名      | 支持的链                                                                 |
|-----------|--------------------------------------------------------------------------|
| `filter`  | INPUT, FORWARD, OUTPUT                                                  |
| `nat`     | PREROUTING, OUTPUT, POSTROUTING                                         |
| `mangle`  | 所有五链（PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING）             |
| `raw`     | PREROUTING, OUTPUT                                                      |

---

### 四、处理优先级
1. **表的优先级**：`raw` → `mangle` → `nat` → `filter`。  
   （例如：`PREROUTING` 链中，先处理 `raw` 表规则，再处理 `mangle` 表，最后是 `nat` 表。）

2. **链的流程**：  
   - **入站流量**：PREROUTING → INPUT  
   - **转发流量**：PREROUTING → FORWARD → POSTROUTING  
   - **出站流量**：OUTPUT → POSTROUTING

---

### 五、常见操作示例
1. **禁止某个 IP 访问本机**（filter 表 + INPUT 链）：
   ```bash
   iptables -A INPUT -s 192.168.1.100 -j DROP
   ```

2. **将 80 端口转发到 8080**（nat 表 + PREROUTING 链）：
   ```bash
   iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
   ```

3. **修改出站数据包的 TTL**（mangle 表 + OUTPUT 链）：
   ```bash
   iptables -t mangle -A OUTPUT -j TTL --ttl-set 64
   ```

---

通过理解四表五链的作用和流程，可以更精准地控制网络行为，例如实现防火墙、NAT、流量标记等功能。

## iptables 执行流程
工作流程小结：
1. 防火墙是层层过滤的，实际是按照配置规则的顺序从上到下，从前到后进行过滤的。
2. 如果匹配成功规则，即明确表示是拒绝(DROP)还是接受(ACCEPT),数据包就不再向下匹配新的规则
3. 如果规则中没有明确表明是阻止还是通过的，也就是没有匹配规则，向下进行匹配，直到匹配默认规则得到明确的阻止还是通过
4. 防火墙的默认规则是所有规则都匹配完才会匹配的

## 常用 `iptables` 参数缩写及含义
| **短选项** | **完整形式/含义**       | **功能说明**                                                                 |
|------------|-------------------------|-----------------------------------------------------------------------------|
| `-s`       | `--source`              | 匹配 **源 IP 地址或网段**（如 `-s 192.168.1.100` 或 `-s 10.0.0.0/24`）。     |
| `-d`       | `--destination`         | 匹配 **目标 IP 地址或网段**（如 `-d 203.0.113.5`）。                         |
| `-p`       | `--protocol`            | 指定 **协议类型**（如 `-p tcp`、`-p udp`、`-p icmp`）。                      |
| `-j`       | `--jump`                | 指定规则匹配后的 **动作**（如 `-j ACCEPT`、`-j DROP`、`-j REJECT`）。        |
| `-i`       | `--in-interface`        | 匹配 **输入网络接口**（如 `-i eth0`）。                                       |
| `-o`       | `--out-interface`       | 匹配 **输出网络接口**（如 `-o wlan0`）。                                      |
| `-m`       | `--match`               | 启用 **扩展模块**（如 `-m multiport`、`-m state --state ESTABLISHED`）。      |
| `--sport`  | `--source-port`         | 匹配 **源端口**（需配合 `-p tcp` 或 `-p udp`，如 `--sport 22`）。             |
| `--dport`  | `--destination-port`    | 匹配 **目标端口**（需配合 `-p tcp` 或 `-p udp`，如 `--dport 80`）。           |
| `-A`       | `--append`              | 在规则链 **末尾追加** 新规则（如 `-A INPUT`）。                               |
| `-I`       | `--insert`              | 在规则链 **指定位置插入** 新规则（默认插入到开头，如 `-I INPUT 2` 插到第2条）。|
| `-D`       | `--delete`              | 从规则链中 **删除** 指定规则（如 `-D INPUT 3` 删除第3条规则）。               |
| `-L`       | `--list`                | 列出规则链内容（如 `iptables -L INPUT`）。                                    |
| `-F`       | `--flush`               | 清空规则链（如 `iptables -F INPUT`）。                                        |
| `-N`       | `--new-chain`           | 创建自定义链（如 `iptables -N MY_CHAIN`）。                                   |


## iptables filter 表案例
```shell
# 屏蔽单个IP的命令
$ iptables -I INPUT -s 223.70.232.41 -j DROP
# 查看防火墙规则信息
$ iptables -nvL --line-numbers
$ sudo iptables-save > /etc/sysconfig/iptables
$ sudo iptables-restore < /etc/sysconfig/iptables
# 改动之前先备份，养成好习惯
$ cp /etc/sysconfig/iptables /etc/sysconfig/iptables.bak 
$ iptables -F #清空防火墙规则 
# 只是拒绝某个ip 的访问不需要加协议
$ iptables -I INPUT 1 -s 192.168.143.102  -j DROP
# 拒绝网段访问本机
$ iptables -I INPUT 1 -s 192.168.143.0/24  -j DROP
# 感叹号取反，注意不要自己跳板机的 ip 也给屏蔽掉了，否则 xshell 无法连接
$ iptables -I INPUT 1 ! -s 192.168.143.0/24  -j DROP
# 主机对172.21.29.116 ip 开放22 端口权限，加端口则需要加协议
$ iptables -I INPUT 1 -s 172.21.29.116 -p tcp --dport 22 -j ACCEPT
# 禁止网段访问指定的端口
$ iptables -I INPUT 1 -s 192.168.143.0/24 -p tcp --dport 8080 -j ACCEPT
# 禁止网段访问指定的多个端口
$ iptables -I INPUT 1 -s 192.168.143.0/24 -p tcp -m multiport --dports 8080,80,9090 -j ACCEPT
# 禁止网段访问指定的端口范围
$ iptables -I INPUT 1 -s 192.168.143.0/24 -p tcp  --dports 1000:2000 -j ACCEPT
# 删除第一条规则
$ iptables -D INPUT 1
# 主机 禁止ping icmp
$ iptables -I INPUT -p icmp --icmp-type 8 -j DROP
# 通过防护墙控制连接状态
$ iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
$ iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```


### 实践：限制 ip 连接数
限制来自任何IP地址的TCP连接到端口22（SSH默认端口），如果来自同一IP的并发连接数超过100，则拒绝这些连接，并使用ICMP端口不可达消息作为响应
```bash
$ iptables -A INPUT -p tcp --dport 22 -m connlimit --connlimit-above 100 --connlimit-mask 32 -j REJECT --reject-with icmp-port-unreachable
$ iptables  -nL --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    REJECT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 #conn src/32 > 100 reject-with icmp-port-unreachable
...

```
命令参数释义：

- `-A INPUT`: 将规则添加到INPUT链中。INPUT链处理所有入站数据包。
- `-p tcp`: 指定协议为TCP。
- `--dport 22`: 目标端口是22，即SSH服务的默认端口。
- `-m connlimit`: 使用connlimit模块来限制每个客户端IP的并发连接数。
- `--connlimit-above 100`: 如果一个源IP地址发起的并发连接数超过100，则应用此规则。
- `--connlimit-mask 32`: 对于IPv4，这表示针对单个IP地址进行限制（子网掩码长度为32意味着每个独立的IP地址）。
- `-j REJECT`: 匹配条件后执行的目标动作是REJECT（拒绝）。
- `--reject-with icmp-port-unreachable`: 当拒绝数据包时，发送ICMP端口不可达的消息作为响应。

这条规则将会对尝试连接到服务器上SSH服务（端口22）的所有流量生效，若某个IP地址的并发连接数超过了100，则新的连接请求将被拒绝，并返回ICMP端口不可达的消息给发起者。这样可以有效防止单一来源的过多连接请求，有助于防范一些类型的Dos攻击或滥用行为。

### 生产配置参考
```shell
# 1. 放开22 端口访问
$ iptables -A INPUT -p tcp --dport 22 -j ACCEPT
# 2. 回环端口放开
$ iptables -A INPUT  -i lo  -j ACCEPT
$ iptables -A OUTPUT -o lo  -j ACCEPT
# 3. 放开常用端口
$ iptables -A INPUT  -p tcp -m multiport --dports 80,443 -j ACCEPT
# 4. 指定网段的访问
$ iptables -A INPUT  -s 192.168.143.0/24  -j ACCEPT
$ iptables -A INPUT  -s 10.168.143.0/24  -j ACCEPT
# 5. 连接状态
$ iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# 6. # 拒绝所有其他入站流量
$ iptables -P INPUT DROP
# 汇总
$ iptables-save
```

## iptables 常用命令参数
`iptables` 是 Linux 系统下管理防火墙规则的核心工具，其参数用于定义规则的匹配条件、操作目标和规则管理。以下是常用参数分类及详细说明：

---

### 一、规则管理参数
| 参数 | 说明 |
|------|------|
| `-A <链名>` | 在指定链的**末尾添加**规则（Append）。<br>示例：`iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT` |
| `-I <链名> [规则编号]` | 在指定链的**指定位置插入**规则（Insert），默认插入到链首。<br>示例：`iptables -I INPUT 2 -p tcp --dport 80 -j DROP` |
| `-D <链名> [规则编号]` | 删除指定链中的某条规则（Delete）。<br>示例：`iptables -D INPUT 3` |
| `-F [链名]` | **清空**指定链的所有规则（Flush），不指定链则清空所有链。<br>示例：`iptables -F INPUT` |
| `-L [链名]` | **列出**指定链的规则（List），不指定链则列出所有链。<br>示例：`iptables -L INPUT -nv` |
| `-N <链名>` | **创建**自定义链（New）。<br>示例：`iptables -N MY_CHAIN` |
| `-X <链名>` | **删除**自定义空链（Delete chain）。<br>示例：`iptables -X MY_CHAIN` |
| `-P <链名> <动作>` | 设置链的**默认策略**（Policy），如 `ACCEPT` 或 `DROP`。<br>示例：`iptables -P INPUT DROP` |

---

### 二、匹配条件参数
| 参数 | 说明 |
|------|------|
| `-s <IP/网段>` | 匹配**源IP地址**。<br>示例：`-s 192.168.1.100` 或 `-s 10.0.0.0/24` |
| `-d <IP/网段>` | 匹配**目标IP地址**。 |
| `-p <协议>` | 匹配协议类型（如 `tcp`, `udp`, `icmp`, `all`）。<br>示例：`-p tcp` |
| `--sport <端口>` | 匹配**源端口**（需配合 `-p tcp` 或 `-p udp`）。<br>示例：`--sport 8080` |
| `--dport <端口>` | 匹配**目标端口**。<br>示例：`--dport 22` |
| `-i <网卡>` | 匹配**输入网卡**（如 `eth0`）。<br>示例：`-i eth0` |
| `-o <网卡>` | 匹配**输出网卡**。 |
| `-m <模块>` | 使用扩展模块（如 `state`, `multiport`, `iprange`）。<br>示例：`-m state --state ESTABLISHED` |
| `--state <状态>` | 匹配连接状态（如 `NEW`, `ESTABLISHED`, `RELATED`）。 |

---

### 三、动作参数（`-j` 参数指定）
| 动作 | 说明 |
|------|------|
| `ACCEPT` | **允许**数据包通过。 |
| `DROP` | **丢弃**数据包（无响应）。 |
| `REJECT` | **拒绝**数据包（返回错误响应）。 |
| `LOG` | 记录日志到系统日志（如 `/var/log/messages`）。<br>示例：`-j LOG --log-prefix "Blocked: "` |
| `SNAT` | 源地址转换（用于 `nat` 表的 `POSTROUTING` 链）。<br>示例：`-j SNAT --to-source 1.2.3.4` |
| `DNAT` | 目标地址转换（用于 `nat` 表的 `PREROUTING` 链）。<br>示例：`-j DNAT --to-destination 192.168.1.100:8080` |
| `MASQUERADE` | 动态源地址转换（适用于动态 IP，如拨号网络）。<br>示例：`-j MASQUERADE` |
| `REDIRECT` | 端口重定向（常用于透明代理）。<br>示例：`-j REDIRECT --to-port 8080` |

---

### 四、其他常用参数
| 参数 | 说明 |
|------|------|
| `-t <表名>` | 指定操作的表（默认 `filter`）。<br>示例：`iptables -t nat -L` |
| `-v` | 显示详细信息（如数据包计数）。<br>示例：`iptables -L -v` |
| `-n` | 不解析域名（直接显示 IP 地址）。<br>示例：`iptables -L -n` |
| `--line-numbers` | 显示规则编号（便于删除或插入）。<br>示例：`iptables -L INPUT --line-numbers` |

---

### 五、典型示例
#### 1. 基础防火墙规则
```bash
# 允许本地回环接口
$ iptables -A INPUT -i lo -j ACCEPT

# 允许已建立的连接
$ iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 允许 SSH 连接（端口 22）
$ iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 拒绝所有其他入站流量
$ iptables -P INPUT DROP
```

#### 2. 端口转发（NAT）
```bash
# 启用 IP 转发
$ echo 1 > /proc/sys/net/ipv4/ip_forward

# 将外部 80 端口转发到内网 192.168.1.100:8080
$ iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:8080
$ iptables -t nat -A POSTROUTING -j MASQUERADE
```

#### 3. 限速规则（使用 `limit` 模块）
```bash
# 限制 ICMP 请求频率（每秒 1 个）
$ iptables -A INPUT -p icmp -m limit --limit 1/s -j ACCEPT
$ iptables -A INPUT -p icmp -j DROP
```

---

### 六、注意事项
1. **规则顺序**：规则按顺序匹配，一旦匹配成功，后续规则不再执行。
2. **默认策略**：设置默认策略为 `DROP` 前，需确保允许必要流量（如 SSH）。
3. **持久化规则**：重启后规则会丢失，需使用 `iptables-save` 和 `iptables-restore` 保存规则：
   ```bash
   iptables-save > /etc/iptables/rules.v4
   iptables-restore < /etc/iptables/rules.v4
   ```

掌握这些参数后，可以灵活配置防火墙规则，实现流量过滤、NAT 转换、日志记录等功能。

## iptables 总结
iptables 在工作中也是常用到的服务，重要但是不是很复杂，这里找了一个iptables[面试参考](https://www.jianshu.com/p/19422676b854)，自己可以写一写，加深巩固下,在生产环境层面执行前最好是现在测试环境验证下

## 参考
1. [iptables详细教程：基础、架构、清空规则、追加规则、应用实例](https://lesca.me/archives/iptables-tutorial-structures-configuratios-examples.html)


