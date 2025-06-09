---
title: "Ansible 部署及快速入门"
subtitle: ""
date: 2020-06-27T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Ansible"]
tags: ["Ansible"]
---
## 1. Ansible 简介
Ansible是一个开源的IT自动化运维工具，用于配置管理、应用部署、任务执行和持续交付。其设计目标是简单易用，以帮助系统管理员更轻松地管理服务器基础设施。Ansible通过YAML（YAML Ain't Markup Language）格式的清晰易懂的文本定义配置（称为Playbook），这种格式有助于用户快速编写和阅读任务及配置信息。

以下是Ansible的主要特点：
- 简单易用：Ansible无需复杂的环境搭建和编程语言知识，采用YAML语法编写任务，具有高度可读性。部署在目标主机上不需要安装Agent，因此不会增加额外负担。
- 代理无需安装：Ansible通过SSH协议与目标主机通信，在目标主机上无需安装额外的代理程序，减轻了系统负担。
- 幂等性：Ansible任务的执行具有幂等性，即同一任务被多次执行时，结果相同且不会产生副作用。这使得Ansible在多次执行任务时，始终保持系统的稳定。
- 模块化：Ansible自带数百个可用于各种任务的模块，如文件管理、软件包安装、系统服务管理、网络设备配置等。用户还可以编写自定义模块以实现特定功能。
- 配置管理：Ansible Playbook中的变量、模板和条件处理等功能，使得配置文件可以轻松实现参数化，同时满足多种环境和主机组的需求。
- 任务编排：使用Ansible Playbook，用户可以编排一系列任务并按顺序执行。
- 社区支持与生态系统：Ansible有一个庞大且积极的社区，提供大量的模块、插件、教程和技术支持。此外，Ansible已被许多知名企业采用，持续地完善和拓展功能。

### 1.1 使用场景
Ansible作为一个功能强大的自动化运维工具，应用广泛，以下是一些典型的使用场景：

- 系统配置管理：使用Ansible，可以轻松地对目标系统进行配置，例如设置用户、组、文件权限、网络配置等。通过YAML格式的Playbook来管理配置，便于维护及版本控制。
- 软件包安装与升级：使用Ansible的软件包管理模块安装、升级或移除软件包。例如，可以针对大量服务器的软件升级、你也可以统一规划软件包的安装和版本。
- 一键部署应用与服务：通过Ansible Playbook实现应用程序和服务的快速、自动化部署。例如，部署Web服务、数据库、负载均衡器等。
- 持续集成与持续部署：将Ansible与持续集成工具（如Jenkins）结合，实现代码自动构建、测试和部署。可以大大提高软件开发和交付的效率。
- 定时任务管理：使用Ansible管理定时任务，例如创建、修改、删除Linux系统中的crontab任务。
- 网络设备管理：使用Ansible管理网络设备，包括路由器、交换机和防火墙等。可以快速地执行配置更改、故障排查和性能数据采集等任务。
- 扩展基础设施：适用于云计算环境中的基础设施管理，可以快速创建、配置和销毁虚拟机或容器，以满足不断变化的需求。
- 安全性管理：使用Ansible进行安全加固和审计，例如管理SSL证书、防火墙规则、OS安全补丁等。
- 监控与告警：通过Ansible部署和配置各种监控工具（如Nagios、Zabbix等），实现对服务器、应用程序和网络设备的监控与告警。
- 跨平台管理：Ansible支持多种操作系统，可以在Linux、Windows、MacOS等系统上实现统一的配置管理和任务执行。

综上所述，Ansible凭借其灵活性和可扩展性，可广泛应用于各种场景。根据具体需求，用户可以开发自定义模块和插件，进一步实现自动化运维的高效性。
### 1.2 工作原理
### 1.3 命令执行过程
### 1.4 ansible 管理命令
- Ansible命令行执行方式有Ad-Hoc、Ansible-playbook两种方式：
  - Ad-Hoc主要用于临时命令的执行。
  - Ansibel-playbook可以理解为Ad-Hoc的集合，通过一定的规则编排在一起。
- 两者的操作也极其简便，且提供了如with_items、failed_when、changed_when、until、ignore_errors等丰富的逻辑条件和Dry-run的Check Mode。但在Chceck Mode下并不真正执行命令，即将执行的操作不会对端服务器产生任何影响，只模拟命令的执行过程是否能正常执行。
- Ansible管理命令有：
  - ansible：这个命令是日常工作中使用率非常高的命令之一，主要用于临时一次性操作。
  - ansible-doc：Ansible模块文档说明，针对每个模块都有详细的用法说明和应用案例介绍。
  - ansible-galaxy：可以简单的理解为Github或PIP的功能，是Ansible官方一个分享role的功能平台。可以通过ansible-galaxy命令很简单的实现role的分享和安装。
  - ansible-playbook：是日常应用中使用频率最高的命令，其工作机制是，通过读取预先编写好的playbook文件实现批量管理。
  - ansible-pull：Ansible的另一种工作模式（pull模式），Ansible默认使用push模式。
  - ansible-vault：主要用于配置文件加密。
  - ansible-console：让用户可以在ansible-console虚拟出来的终端上像Shell一样使用Ansible内置的各种命令。
## 2. install 部署
- dns resolve(可选)
- install ansible

安装
```shell
$ yum install epel-release -y
$ yum install ansible -y
```
检测部署是否完成
- `rpm -ql ansible` 列出所有文件
- `rpm -qc ansible` 查看配置文件
- `ansible --help` 查看ansible帮助
- `ansible-doc -l` 查看ansible所有模块
- `ansible-doc -s yum` 查看ansible yum模块,并查看其功能

注意：
- 部分yum方式安装ansible可能会出错，可以选择采用[rpm进行安装](https://releases.ansible.com/ansible/rpm/release/)
- 采坑-rpm安装的，运行会提示缺少argparse modules，用`yum install python-argparse`安装即可

### 2.1 免密配置(可选)
```shell
$ ssh-keygen -t dsa
$ ssh-copy-id 192.168.143.102
$ ssh 192.168.143.102
Last login: Tue Jan  7 11:44:30 2025 from 192.168.143.1
$ ip a|grep 192.168.143.102
    inet 192.168.143.102/24 brd 192.168.143.255 scope global ens32
```


## 3. Ansible 基础
### 3.1 配置主机清单
```shell
root@localhost:~$ vim /etc/ansible/hosts 
...
## db-[99:101]-node.example.com
192.168.143.102
```
### 3.2 测试 ansible 模块
指定`ping`模块
```shell
$ ansible localhost -m ping
localhost | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
root@localhost:~$ ansible 192.168.143.102 -m ping
192.168.143.102 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
# 如果目标主机未设置免密登录，并设置简洁输出
root@localhost:~$ ansible 192.168.143.102 -m ping -o -u root -k
SSH password: 
192.168.143.102 | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "ping": "pong"}
```
## 4. Inventory 主机清单
1. 增加主机组
```shell
root@localhost:~$ vim /etc/ansible/hosts 
...
## db-[99:101]-node.example.com
[webserver]
host1
host2
host3
host4
```
测试组
```shell
$ ansible webserver -m ping -o -u root -k
SSH password: 
192.168.143.102 | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "ping": "pong"}
```
2. 增加用户名和密码
```shell
root@localhost:~$ vim /etc/ansible/hosts 
...
## db-[99:101]-node.example.com
[webserver2]
192.168.143.102 ansible_ssh_user='root' ansible_ssh_pass='123456'
```
测试
```shell
root@localhost:~$ ansible webserver2 -m ping -o 
192.168.143.102 | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "ping": "pong"}
```
3. 增加端口
```shell
root@localhost:~$ vim /etc/ansible/hosts 
...
## db-[99:101]-node.example.com
[webserver3]
192.168.143.102 ansible_ssh_user='root' ansible_ssh_pass='123456' ansible_ssh_port='22' 
```
测试
```shell
root@localhost:~$ ansible webserver3 -m ping -o 
192.168.143.102 | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "ping": "pong"}
```
4. 组：变量
```shell
root@localhost:~$ vim /etc/ansible/hosts 
...
## db-[99:101]-node.example.com
[webserver]
host1 ansible_ssh_port='2222' # 定制化端口
host2
host3
host4
[webserver:vars]
ansible_ssh_user='root' 
ansible_ssh_pass='123456'
```
5. 子分组
```shell
root@localhost:~$ vim /etc/ansible/hosts 
...
## db-[99:101]-node.example.com
[tomcaterver]
host1
host2
[mysqlterver]
host3
host4
[productserver:children]
```
测试
```shell
$ ansible productserver -m ping -o 
```
6. 自定义主机列表
```shell
$ ansible -i /etc/ansible/hosts webserver -m ping -o 
192.168.143.102 | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "ping": "pong"}
```

- [Ansible 主机清单参数说明](http://www.ansible.com.cn/docs/intro_inventory.html#behavioral-parameters)
## 6. Add-Hoc 点对点模式
内部指令常用模块
1. shell
2. copy
3. user
4. 软件包管理
5. 服务模块
6. 文件模块
7. 收集模块
8. fetch
9. cron
10. group
11. script
12. unarchive

`copy`模块案例
```shell
$ touch /tmp/1.txt
$ ansible-doc copy
$ ansible webserver -m copy -a 'src=/tmp/1.txt dest=/tmp/11.txt owner=root group=root mode=777'

192.168.143.102 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "checksum": "da39a3ee5e6b4b0d3255bfef95601890afd80709", 
    "dest": "/tmp/11.txt", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "d41d8cd98f00b204e9800998ecf8427e", 
    "mode": "0777", 
    "owner": "root", 
    "secontext": "unconfined_u:object_r:admin_home_t:s0", 
    "size": 0, 
    "src": "/root/.ansible/tmp/ansible-tmp-1736236277.53-3361-267624272356035/source", 
    "state": "file", 
    "uid": 0
}
```
## 7. Role 角色扮演
Role则是在ansible中，playbooks的目标组织结构。将代码或文件进行模块化，成为roles的文件目录组织结构，易读，代码可重用，层次清晰。


自动化运维面试的时候问到ansible的会比较多一些，ansible问的更多的便是playbook了。

### 1. 目录结构
Ansible 的 Role 是一种将 Playbook 中的任务、变量、模板和文件组织在一起的方法，使得 Playbook 更加模块化、可重用和易于管理。每个 Role 通常对应一个特定的功能或服务（如 web server, database 等）。以下是一个典型的 Ansible Role 目录结构：
```shell
roles/
└── myrole
    ├── defaults                # 定义默认变量，这些变量可以在外部被覆盖。
    │ └── main.yml              # 默认变量文件，通常用于设置可选配置项。
    ├── files                   # 存放需要复制到远程主机上的静态文件。
    ├── handlers                # 定义可以由任务触发的动作（如重启服务）。
    │ └── main.yml              # 处理程序文件，定义了当某些条件满足时执行的操作。
    ├── meta                    # 包含角色元数据，如依赖关系、平台兼容性等。
    │ └── main.yml              # 元数据文件，指定角色依赖和其他属性。
    ├── README.md               # 说明文档，描述角色的功能、使用方法和注意事项。
    ├── tasks                   # 角色的主要任务列表，定义要执行的具体操作。
    │ └── main.yml              # 必须存在的入口文件，包含了一系列的任务定义。其它目录和文件为可选
    ├── templates               # 存放 Jinja2 模板文件，用于生成配置文件等文本内容。
    ├── tests                   # 测试用例目录，通常包括测试 Playbook 和库存文件。
    │ ├── inventory             # 测试环境的库存文件，定义了测试目标主机。
    │ └── test.yml              # 测试 Playbook 文件，用于验证角色是否按预期工作。
    └── vars                    # 定义不可覆盖的变量，通常用于强制性的配置选项。
        └── main.yml            # 变量文件，包含了角色所需的固定配置值。
```
#### 各个目录和文件的作用

- **defaults/**: 此目录下的 `main.yml` 文件包含了该角色的默认变量，这些变量可以很容易地被用户自定义或覆盖。它们通常是可选的配置项，提供了合理的默认值。

- **files/**: 这个目录用来存放那些不需要经过模板处理就可以直接复制到远程节点上的静态文件。

- **handlers/**: 在这里定义的是处理程序，它们是一些特殊的任务，只有当其他任务通知它们时才会被执行。常见的例子是重启服务，因为这通常不应该在每次运行时都发生，而是在配置更改后才需要。

- **meta/**: 包含有关角色的元数据信息，比如它可能依赖的其他角色、适用的操作系统类型等。这对于确保角色正确安装其依赖项非常重要。

- **README.md**: 提供关于角色的详细信息，例如如何使用它、它的功能是什么以及任何重要的提示或警告。

- **tasks/**: 这是角色的核心部分，`main.yml` 文件中列出了所有要执行的任务。这是当你将角色添加到 Playbook 中时会被调用的地方。

- **templates/**: 如果你需要创建基于模板的文件（如配置文件），那么这些模板应该放在这个目录下。Ansible 使用 Jinja2 模板引擎来渲染这些模板，并根据需要插入变量值。

- **tests/**: 该目录包含了用于测试角色的资源，包括一个模拟的库存文件和一个或多个测试 Playbook。这对于开发过程中进行单元测试和集成测试非常有用。

- **vars/**: 此目录中的 `main.yml` 文件定义了角色所需的变量，这些变量应该是不可轻易覆盖的，因为它们代表了角色内部的关键配置。

通过这样的组织方式，Ansible 的 Roles 不仅易于理解，而且也促进了代码的重用性和模块化设计。
### 2. 编写任务
创建一个 Role

你可以手动创建上述目录结构，或者使用 ansible-galaxy 命令来自动创建。以下是使用 ansible-galaxy 创建 Role 的步骤：

1. 打开命令行工具。
2. 使用 ansible-galaxy role init <rolename> 来创建一个新的 Role。例如，如果你想创建一个名为 myrole 的 Role，你会运行：
```shell

root@localhost:/etc/ansible$ ansible-galaxy role init myrole^C
root@localhost:/etc/ansible$ tree roles/
roles/
└── myrole
    ├── defaults
    │ └── main.yml
    ├── files
    ├── handlers
    │ └── main.yml
    ├── meta
    │ └── main.yml
    ├── README.md
    ├── tasks
    │ └── main.yml
    ├── templates
    ├── tests
    │ ├── inventory
    │ └── test.yml
    └── vars
        └── main.yml

9 directories, 8 files
```
### 3. 准备配置文件
files 目录下是存放静态文件，templates 目录下是存放模板文件，文件中设置占位符一般是结合vars 变量和setup模块进行组合使用。
1. 在files 目录下创建一个静态文件，通过处理程序将其拷贝到远程主机
```shell
$ cd files
$ touch ansible.txt
```
2. 在templates 目录下创建一个模板文件
```shell
$ cd templates
$ vim ansible-templates.txt
the ntp_port is {{ ntp_port }}
```
### 4. 编写变量
变量结合模板文件一起使用，方便动态修改
```shell
$ cd vars
$ vim main.yml
ntp_port: 8888
```
### 5. 编写处理程序
处理程序tasks可以理解为playbook的导演
```shell
$ cd tasks
$ vim main.yml
- name: "Tasks in role demo"
  debug: msg="Simple role example"
- name: "copy file to dest host"
  copy:
    src: ansible.txt
    dest: /tmp/ansible.txt
    mode: 0644

- name: "template file to dest host"
  template:
    src: ansible-template.txt
    dest: /tmp/ansible-template.txt
    owner: root
    group: root
    mode: 0644
```
### 6. 剧本编写
`site.yaml` 可以理解为剧本，可以设置每个role的执行顺序，例如冯小刚导演的电影1942的剧本就是刘震云编剧的
```shell
$ cd /etc/ansible/role
$ vim site.yaml
---
# This playbook install base_vm
- hosts: nodes 
  #gather_facts: yes
  pre_tasks:
    - name: pre task
      shell: echo "hello" in pre_tasks
  roles:
    - role: myrole
      #when: "ansible_os_family == 'RedHat'"  # 条件判断可以将其注释掉，结合gather_facts一起使用
  tasks:
    - name: tasks in site.yaml
      debug: msg="this is task in site.yaml"
  post_tasks:
    - name: post task
      shell: echo "goodbye" in post_tasks
```
### 7. 测试
对`playbook`进行语法测试
```shell
$ ansible-playbook site.yaml --syntax-check

playbook: site.yaml
# 查看输出的细节
$ ansible-playbook -i hosts site.yml --verbose
# 查看该脚本影响哪些主机
$ ansible-playbook -i hosts site.yml --list-hosts
# 并行执行脚本
$ ansible-playbook site.yml -f 10 
# Dry run
$ ansible-playbook site.yml --check
```

## 参考
1. [Ansible 社区文档](https://docs.ansible.org.cn/ansible/latest/getting_started/index.html)
2. [Ansible中文权威指南](http://www.ansible.com.cn/docs/)
3. [自动化运维-Ansible03-Ansible常用命令和模块](https://www.cnblogs.com/maiblogs/p/16833354.html)
4. [github官网 ansible examples](https://github.com/ansible/ansible-examples)
5. [运维自动化之ANSIBLE](https://note.youdao.com/ynoteshare/index.html?id=ca2d3bbe7c37a2fd69ece59bdaad219e&type=note&_time=1736411850199)
6. [Ansible Galaxy](https://galaxy.ansible.com/ui/)

