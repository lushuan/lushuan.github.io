# Linux 时间校正

Linux服务器运行久时，系统时间就会存在一定的误差，一般情况下可以使用date命令进行时间设置，但在做数据库集群分片等操作时对多台机器的时间差是有要求的，此时就需要使用ntpdate进行时间同步

## 手动修改
手动设置时间，需要使用 sudo 权限用户
```shell
# 当前时区具体的时间
date -s "2022-12-31 23:59:59"
# 写入修改后的时间到硬件时钟（RTC）中，将确保设置的时间在下次启动时保持不变
hwclock --systohc
```
手动修改时间会有误差，建议使用下面的时间服务器进行同步更新

##  通过时间服务器校正
```shell
sudo yum install ntpdate -y
```

###  同步时间
```shell
sudo ntpdate -b time1.aliyun.com
```

###  配置启动ntpd

```shell
$ vi /etc/ntp.conf
...
# For more information about this file, see the man pages
# ntp.conf(5), ntp_acc(5), ntp_auth(5), ntp_clock(5), ntp_misc(5), ntp_mon(5).

driftfile /var/lib/ntp/drift

# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default nomodify notrap nopeer noquery

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1 
restrict ::1

# Hosts on local network are less restricted.
#restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server time1.aliyun.com

#broadcast 192.168.1.255 autokey        # broadcast server
#broadcastclient                        # broadcast client
#broadcast 224.0.1.1 autokey            # multicast server
#multicastclient 224.0.1.1              # multicast client
#manycastserver 239.255.254.254         # manycast server
#manycastclient 239.255.254.254 autokey # manycast client

# Enable public key cryptography.
#crypto

includefile /etc/ntp/crypto/pw

# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography. 
keys /etc/ntp/keys

# Specify the key identifiers which are trusted.
#trustedkey 4 8 42

# Specify the key identifier to use with the ntpdc utility.
#requestkey 8

# Specify the key identifier to use with the ntpq utility.
#controlkey 8

# Enable writing of statistics records.
#statistics clockstats cryptostats loopstats peerstats

# Disable the monitoring facility to prevent amplification attacks using ntpdc
# monlist command when default restrict does not include the noquery flag. See
# CVE-2013-5211 for more details.
# Note: Monitoring will not be disabled with the limited restriction flag.
```

###  设置开机自启
```shell
sudo systemctl status ntpd;
sudo systemctl start ntpd;
sudo systemctl enable ntpd;
```

### ntp 常用服务器
#### 国内
- cn.pool.ntp.org  中国开源免费NTP服务器
- ntp1.aliyun.com 阿里云NTP服务器
- ntp2.aliyun.com 阿里云NTP服务器
- time1.aliyun.com 阿里云NTP服务器
- time2.aliyun.com 阿里云NTP服务器

  
#### 国外
- time1.apple.com 苹果NTP服务器
- time2.apple.com 苹果NTP服务器
- time3.apple.com 苹果NTP服务器
- time4.apple.com 苹果NTP服务器
- time5.apple.com 苹果NTP服务器
- time1.google.com 谷歌NTP服务器
- time2.google.com 谷歌NTP服务器
- time3.google.com 谷歌NTP服务器
- time4.google.com 谷歌NTP服务器
- pool.ntp.org 开源免费NTP服务器





