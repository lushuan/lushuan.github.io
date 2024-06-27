# lvm 逻辑卷组管理

## 背景
部署 k8s 时需要对磁盘进行分区划分，正常是集成是直接给创建好的，现场在沟通过程中只提供了一块磁盘。
本篇文章记录一下磁盘分区划分的原理和步骤，尽量选择大白话。

![lvm-1](/images/linux/lvm-1.jpg)

## lvm 介绍
LVM——全称Logical Volume Manager，可以将多个硬盘和硬盘分区做成一个逻辑卷，并把这个逻辑卷作为一个整体来统一管理，动态对分区进行扩缩空间大小，提高磁盘管理灵活性。

在安装CentOS 7的过程中选择自动分区时，默认就是以LVM的方案安装的系统。

但是/boot分区必须独立出来，不能基于LVM创建。

## 三个概念

### PV(Physical Volume) 物理卷
> 物理卷，Physical Volume，是LVM机制的基本存储设备，通常对应一个普通分区或是整个硬盘。  
创建物理卷时，会在分区或磁盘头部创建一个用于记录LVM属性的保留区块，并把存储空间分割成默认大小为4MB的基本单元（Physical Extend，PE），从而构成物理卷。  
普通分区先转换分区类型为8e；整块硬盘，可以将所有的空间划分为一个主分区再做调整

### VG(Volume Group) 卷组
卷组，Volume Group，是由一个或多个物理卷组成的一个整体。可以动态添加、移除物理卷，创建时可以指定PE大小
### LV(Logical Volume) 逻辑卷
逻辑卷，Logical Volume，建立在卷组之上，与物理卷没有直接关系。格式化后，即可挂载使用。

## 三者关系

![lvm](/images/linux/lvm.png)

## lvm特点
LVM最大的特点就是可以对磁盘进行动态管理。因为逻辑卷的大小是可以动态调整的，而且不会丢失现有的数据。我们如果新增加了硬盘，其也不会改变现有上层的逻辑卷。作为一个动态磁盘管理机制，逻辑卷技术大大提高了磁盘管理的灵活性！

## 工作原理
1. 物理磁盘被格式化为PV，空间被划分为一个个的PE
1. 不同的PV加入到同一个VG中，不同PV的PE全部进入到了VG的PE池内
1. LV基于PE创建，大小为PE的整数倍，组成LV的PE可能来自不同的物理磁盘
1. LV现在就直接可以格式化后挂载使用了
1. LV的扩充缩减实际上就是增加或减少组成该LV的PE数量，其过程不会丢失原始数据

## 讲个故事
上面工作原理可以做一个类比，集成提供的挂载磁盘可以理解为一堆小麦，pv的作用是将这一堆小麦打磨成面粉，
不管挂载多少个磁盘pv都可以做成面粉，一堆一堆的面粉(pv格式化的各个磁盘)可以加入到一个VG中，
这个VG可以理解为一个供销社将所有的面粉进行了纳管，当然也可以不同的面粉划分给不同的供销社,
意思就是四块磁盘A、B、C、D 通过打面机pv做成了四份面粉，A、C份面粉卖给了大脚供销社VG-DJ,C、D两份面粉卖给了小脚供销社
VG-XJ,现在亚洲舞王尼古拉斯赵四想要吃馒头，就选择了去大脚供销社去购买面粉，打算购买50斤面粉。这50斤面粉供销提供的lv,
lv 一小份的面粉是来自于vg供销社而不是pv 面粉制造机器。下面会给出示例从小麦生产到赵四购买面粉。

## 操作
![lvm-2](/images/linux/lvm-2.jpg)

### 分区操作
1、 查看磁盘  

通过`fdisk -l`可以看出集成提供了一块600多G的磁盘 `/dev/xvdc`
```shell
$ fdisk -l
....
Disk /dev/xvdc: 644.2 GB, 644245094400 bytes, 1258291200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
2、 创建物理卷  

pv 将小麦加工成面粉，原理是物理磁盘被格式化为PV，空间被划分为一个个的PE
```shell
$ pvcreate /dev/xvdc
  Physical volume "/dev/xvdc" successfully created
```
3、 创建逻辑卷组  

将面粉运输至大脚供销社vgdata,vgdata 是自定义的逻辑卷组名
```shell
$ vgcreate vgdata /dev/xvdc
  Volume group "vgdata" successfully created
$ vgs
  VG     #PV #LV #SN Attr   VSize    VFree   
  centos   1   3   0 wz--n-  <99.00g       0 
  datavg   1   1   0 wz--n- <400.00g    4.00m
  vgdata   1   0   0 wz--n- <600.00g <600.00g
```
4、创建逻辑卷  

可以看的出来逻辑卷的创建是来自于逻辑卷组vg,后面的扩容也是从逻辑卷组进行扩容，这个步骤可以比喻为赵四去大脚超市
购买了35斤的面粉
```yaml
mkdir -p /var/lib/kubelet
lvcreate -n lvkubelet -L +35G vgdata
mkfs.xfs /dev/vgdata/lvkubelet
echo "/dev/mapper/vgdata-lvkubelet /var/lib/kubelet xfs defaults 0 0" >> /etc/fstab
mount -a
```
查看逻辑卷是否挂载成功
```shell
$ df -h|grep kubelet
/dev/mapper/vgdata-lvkubelet    35G   35M   35G   1% /var/lib/kubelet
```

## 相关操作命令
### 创建逻辑卷
Linux 系统使用逻辑卷来模拟物理分区，并在其中保存文件系统。

| 选 项 | 长选项名         | 描 述                           |
|-----|--------------|-------------------------------|
| -c  | --chunksize  | 指定快照逻辑卷的单位大小                  |
| -C  | --contiguous | 设置或重置连续分配策略                   |
| -i  | --stripes    | 指定条带数                         |
| -I  | --stripesize | 指定每个条带的大小                     |
| -l  | --extents    | 指定分配给新逻辑卷的逻辑区段数，或者要用的逻辑区段的百分比 |
| -L  | --size       | 指定分配给新逻辑卷的硬盘大小                |
| -M  | --persistent | 让次设备号一直有效                     |
| -n  | --name       | 指定新逻辑卷的名称                     |
| -p  | --permission | 为逻辑卷设置读/写权限                   |
| -r  | --readahead  | 设置预读扇区数                       |
| -R  | --regionsize | 指定将镜像分成多大的区                   |
| -s  | snapshot     | 创建快照逻辑卷                       |
| -Z  | --zero       | 将新逻辑卷的前1 KB数据设置为零             |
| -m  | --mirrors    | 创建逻辑卷镜像                       |
| 空   | --minor      | 指定设备的次设备号                     |

参数很多，其实常用的就几个
### 指定逻辑卷大小
如逻辑卷组 Vol1
```shell
lvcreate -n lvtest -L +32G Vol1
```

## 创建文件系统
此时只有逻辑卷没有文件系统，需要给逻辑卷指定文件系统

这里的文件系统指定的是ext4，公司生产环境中使用的是 xfs 文件系统如`mkfs.xfs /dev/vgdata/lvkubelet`，以下示例使用的是ext4文件系统，不同文件系统的差别请自行了解，这里不做说明
```shell
sudo mkfs.ext4 /dev/Vol1/lvtest 
mke2fs 1.41.12 (17-May-2010) 
Filesystem label= 
OS type: Linux 
Block size=4096 (log=2) 
Fragment size=4096 (log=2) 
Stride=0 blocks, Stripe width=0 blocks 
131376 inodes, 525312 blocks 
26265 blocks (5.00%) reserved for the super user 
First data block=0 
Maximum filesystem blocks=541065216 
17 block groups 
32768 blocks per group, 32768 fragments per group 
7728 inodes per group 
Superblock backups stored on blocks: 
 32768, 98304, 163840, 229376, 294912 
Writing inode tables: done 
Creating journal (16384 blocks): done 
Writing superblocks and filesystem accounting information: done 
This filesystem will be automatically checked every 28 mounts or 
180 days, whichever comes first.Use tune2fs -c or -i to override.
```
### 目录挂载
在创建了文件系统后需要将该逻辑卷挂载到指定的虚拟目录下，就跟他是物理分区一样
```shell
sudo mount /dev/Vol1/lvtest /mnt/my_partition
```
命令解释是将逻辑卷挂载到指定虚拟目录上

设置挂载永久生效，否则机器重启后会丢失掉

注意修改名称
```shell
echo "/dev/mapper/$vgname-lvdocker /var/lib/docker xfs defaults 0 0" >> /etc/fstab
```

### 修改LVM
| 命 令      | 功 能       |
|----------|-----------|
| vgchange | 激活和禁用卷组   |
| vgremove | 删除卷组      |
| vgextend | 将物理卷加到卷组中 |
| vgreduce | 从卷组中删除物理卷 |
| lvextend | 增加逻辑卷的大小  |
| lvreduce | 减小逻辑卷的大小  |

通过以上命令可以完全控制你的LVM环境

如扩容

扩容前先取消挂载
```shell
umount /mnt/my_partition
# 对/dev/Vol1/lvtest 增加1G
lvextend -L +1G /dev/Vol1/lvtest -r
# 重新挂载设备，并查看挂载状态
mount -a
df -h
```

## LVM 命令汇总

|功能|pv命令|vg命令|lv命令|
|---|---|---|---|
|扫描|pvscan|vgscan|lvscan|
|Create创建|pvcreate|vgcreate|lvcreate|
|Display显示|pvdisplay|vgdisplay|lvdisplay|
|Remove移除|pvremove|vgremove|lvremove|
|Extend扩展|-|vgextend|lvextend|
|Reduce减少|-|vgreduce|lvreduce|

关于收缩 LVM 逻辑卷，可以参考上面的命令lvreduce。这会涉及到数据安全性问题，谨慎使用

## 小结
就是三步走，创建物理卷，创建卷组，创建逻辑卷，然后格式化并挂载使用。扩容的话，没有挂载使用，就正常扩，然后格式化挂载，已经挂载使用了，扩容后在线调整生效。

