---
title: "Ansible 权限认证报错问题记录"
subtitle: ""
date: 2020-04-17T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["Ansible"]
tags: ["Ansible"]
slug: "ansible-ssh-auth-error"
summary: "记录 Ansible 通过 SSH 连接目标主机时出现 Permission denied (UNREACHABLE) 报错的排查过程，并给出基于 ssh-keygen 与 ssh-copy-id 的免密认证解决方案。"
---

在使用 Ansible ping 多台主机时遇到 SSH Permission denied 导致 UNREACHABLE 的报错。本文记录该问题的排查过程，并通过 ssh-keygen 生成密钥、ssh-copy-id 分发公钥的方式实现免密认证，最终验证问题解决。

<!--more-->

## 一、问题描述
通过ansible命令直接ping多台机器的网络状态，提示报错

{{< admonition failure >}}
172.16.24.220 | UNREACHABLE! => {
    "changed": false, 
    "msg": "Failed to connect to the host via ssh: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).\r\n", 
    "unreachable": true
}
{{< /admonition >}}
## 二、问题处理
解决方式：单向的ssh验证

ssh-keygen一路回车,主要是用来免密通信的

ssh-copy-id 172.16.24.220 需要输入对应主机的root密码
```shell
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
7f:69:87:cf:28:fe:8b:19:55:a7:d0:c9:aa:6d:05:0c root@chm
The key's randomart image is:
+--[ RSA 2048]----+
|          E      |
|           o o . |
|            + = .|
|             = o |
|        S   o o  |
|         . + +   |
|          + B .  |
|          .B =   |
|         .+o+.o  |
+-----------------+
[root@chm log]# 
[root@chm log]# ssh-copy-id 172.16.24.220
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@172.16.24.220's password: 
 
Number of key(s) added: 1
 
Now try logging into the machine, with:   "ssh '172.16.24.220'"
and check to make sure that only the key(s) you wanted were added.
```
## 三、验证
再次验证，成功
```shell
[root@chm log]# ansible 172.16.24.220 -m ping 
172.16.24.220 | SUCCESS => {
    "changed": false, 
    "failed": false, 
    "ping": "pong"
}
```

