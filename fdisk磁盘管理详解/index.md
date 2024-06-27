# fdisk 磁盘管理详解

打算给一台服务器做逻辑卷分区，发现磁盘未做分区，且磁盘空间余量较大，本篇记录一下 fdisk 磁盘分区划分的过程

![fdisk-1](/images/linux/fdisk-1.jpg)
## 事情的起因
通过`pvcreate` 创建逻辑卷组提示报错
报错信息

{{< admonition warning >}}
Can't open /dev/sda1 exclusively. Mounted filesystem?
{{< /admonition >}}


通过 `fdisk -l` 查看该分区为boot 分区无法创建逻辑卷。fdisk -l 可以用来查看磁盘的所有分区
```shell
$ fdisk -l

Disk /dev/sda: 1000.2 GB, 1000171331584 bytes, 1953459632 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes
Disk label type: dos
Disk identifier: 0x0003eb01

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200   276834303   137367552   8e  Linux LVM
```
从上面的信息可以了解到磁盘`/dev/sda`划分了两个分区，且扇区大小为512字节，boot 分区大小为(2099199-2048)*512/1024/1024=1G,
且通过`pvdisplay`查看该`/dev/sda2 ` 分区为131G，已经全部用完。总的磁盘有1000.2 GB 可用，抛出boot /dev/sda1分区和/dev/sda2 的
131GB空间，还剩800多GB的磁盘空间未使用。闲言少叙，开始通过fdisk划分新的磁盘分区`/dev/sda3`

主分区和扩展分区

![fdisk-2](/images/linux/fdisk-2.jpg)

## 划分磁盘分区
### 首先了解什么是 fdisk 命令
fdisk 的意思是 固定磁盘(Fixed Disk) 或 格式化磁盘(Format Disk)，它是命令行下允许用户对分区进行查看、创建、调整大小、删除、移动和复制的工具。它支持 MBR、Sun、SGI、BSD 分区表，但是它不支持 GUID 分区表（GPT）。它不是为操作大分区设计的。

fdisk 允许我们在每块硬盘上创建最多四个主分区。它们中的其中一个可以作为扩展分区，并下设多个逻辑分区。1-4 扇区作为主分区被保留，逻辑分区从扇区 5 开始。

### 查看分区
fdisk 查看特定分区
```shell
fdisk -l /dev/sda
```

`fdisk`交互指令说明

|命令 |说明| 是否常用|
|---|---|---|
|a | 设置可引导标记 |  否 | 
|b| 编辑bsd磁盘标签 |  否  | 
|c| 设置DOS操作系统兼容标记 |  否  | 
|d| 删除一个分区 注：这是删除一个分区的动作  | 是  | 
|l| 显示已知的分区类型。82 为Linux swap分区，83为Linux分区 <br>  注：l是列出分区类型，以供我们设置相应分区的类型； |   是 | 
|m| m 是列出帮助信息 |  是  | 
|n| 添加一个分区 |   是 | 
|o| 建立空白DOS分区表 | 否  | 
|p| p列出分区表 |  是  | 
|q| 不保存退出 |  是  | 
|s| 新建空白SUN磁盘标签 | 否  | 
|t| 改变一个分区的系统ID | 是  | 
|u| 改变显示记录单位 |否   | 
|v| 验证分区表 | 否  | 
|w| 把分区表写入硬盘并退出 |   是 | 
|x| 扩展应用，专家功能   | 否  | 
关注一下常用命令即可

## 创建扩展分区
列出当前操作硬盘的分区情况，用 p
```shell
$ fdisk /dev/sda
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default e): p

```
新建磁盘分区，用 n,在创建的时候会提示选择是使用主分区还是逻辑分区，见扩展一。
直接下一步会提示创建分区的大小，指定分区大小时直接默认起始扇区后，在last Sector size 处直接+400GB，最后
选择 `w`进行保存，这个不要忘记。这个时候分区已经分配好了，但是还不能使用，因为没有对该分区进行格式化操作，指定分区文件系统。

### 指定分区文件系统并生效分区
选择不同文件系统的性能也不通，选择合适的文件系统对分区进行格式，在格式化文件系统前先更新分区表。重新扫描分区表需要借助`partprobe`命令。

`partprobe`命令是用于重新扫描系统上的分区表，以便内核能够识别新创建或修改的分区。当您在不重启系统的情况下对磁盘进行了分区或调整分区大小时，可以使用partprobe命令通知内核更新分区信息。

`partprobe`命令会读取分区表并向内核发送信号，让内核重新加载分区信息。这样，系统就能够识别到最新的分区信息，并使其可用于挂载和访问。

```shell
$ partprobe
$ cat /proc/partitions
major minor  #blocks  name

   8        0  976729816 sda
   8        1    1048576 sda1
   8        2  137367552 sda2
   8        3  412102655 sda3

[root@CENTOS7 temp]#  mkfs.ext3 /dev/sda3
```
注意：`partprobe`命令需要root权限才能执行。在执行partprobe之后，您可以使用相应的命令（例如`fdisk`或`lsblk`）来检查分区是否已成功更新。

格式化完分区表后那么到此就完成了，我的目标是完成硬盘分区来创建物理卷。后续可以参考[逻辑卷组管理文章](LVM逻辑卷组管理.md)
如：
```shell
pvcreate /dev/sda3
vgcreate vgdata /dev/sda3
```
### 设置开机挂载
设置开机挂载需同步至 /etc/fstab 文件中  

示例
```shell
mkdir -p /var/lib/kubelet 
lvcreate -n lvkubelet -L +35G vgdata
mkfs.xfs /dev/vgdata/lvkubelet
echo "/dev/mapper/vgdata-lvkubelet /var/lib/kubelet xfs defaults 0 0" >> /etc/fstab
mount -a
```

### 扩展一：fdisk 新增主分区和扩展分区的主要区别

1. 主分区(Primary Partition)是硬盘上的基本分区,每个硬盘最多可以创建4个主分区。

2. 扩展分区(Extended Partition)是一种特殊的主分区,它不能直接存储数据,仅可用于扩展更多的逻辑分区。每个硬盘只能创建1个扩展分区。

3. 主分区可以直接存储数据,而扩展分区内需要再创建逻辑分区(Logical Partition)才能存储数据。

4. 主分区对文件系统格式化的支持更广泛,可支持各种文件系统。扩展分区和逻辑分区仅支持部分文件系统。

5. 主分区的访问效率稍高一些。扩展分区和逻辑分区的访问会稍微低一些。

6. 如果没有空间限制,建议使用主分区。如果磁盘空间不足,可以使用扩展分区额外增加更多分区。

所以总的来说,主分区更基础和通用,扩展分区可以提供更多额外的分区数。选择时需要根据实际磁盘空间及使用需求来决定。

小结：有主选主，主分区更基础和通用


