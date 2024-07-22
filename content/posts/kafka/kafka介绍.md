---
title: "Apache kafka 介绍"
subtitle: ""
date: 2021-05-19T12:06:37+08:00
lastmod: 2022-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Kafka"]
tags: ["Kafka"]
---
## kafka 简介
Kafka 被称为下一代分布式消息系统，是非营利性组织ASF(Apache Software Foundation，简称为ASF)基金会中的一个开源项目，比如HTTP Server、Hadoop、ActiveMQ、Tomcat等开源软件都属于Apache基金会的开源软件，类似的消息系统还有RbbitMQ、ActiveMQ、ZeroMQ，最主要的优势是其具备分布式功能、并且结合zookeeper可以实现动态扩容。

![kafka-vs-others](/images/kafka/kafka-vs-others.png "kafka 竞品比较")

### 名字的由来
kafka的诞生，是为了解决linkedin的数据管道问题，起初linkedin采用了ActiveMQ来进行数据交换，大约是在2010年前后，那时的ActiveMQ还远远无法满足linkedin对数据传递系统的要求，经常由于各种缺陷而导致消息阻塞或者服务无法正常访问，为了能够解决这个问题，linkedin决定研发自己的消息传递系统，当时linkedin的首席架构师jay kreps便开始组织团队进行消息传递系统的研发；由于jay kreps非常喜欢franz kafka,并且觉得kafka这个名字很酷，因此取了个和消息传递系统完全不相干的名称kafka，该名字并没有特别的含义。

### Kafka适合什么样的场景?
它可以用于两大类别的应用:
1. 构造实时流数据管道，它可以在系统或应用之间可靠地获取数据。 (相当于message queue)
2. 构建实时流式应用程序，对这些流数据进行转换或者影响。 (就是流处理，通过kafka stream topic和topic之间内部进行变化)
### kafka 优势
1. 吞吐量高，性能好
2. 伸缩性好，支持在线水平扩展
3. 容错性和可靠性
4. 与大数据生态紧密结合，可无缝对接hadoop、stream,spark等

### 发行版本
1. Confluent Platform 
2. Cloudera kafka 偏大数据解决方案
3. hortonworks kafka 偏大数据解决方案


## 消息模型
- JMS 规范
    - 队列-点对点
    - 主题-发布订阅
    - Apache ActiveMQ
- AMQP(协议)
  - AMQP模型
    - 队列
    - 信箱
    - 绑定 
  - 特点：支持事务，数据一致性高，多用于银行，金融行业
  - Pivotal RabbitMQ
  - Spring AMQP与Spring JMS 
- MQTT

##  Kafka中的相关概念

在Kafka集群(Cluster)中，一个Kafka节点就是一个Broker，消息由Topic来承载，可以存储在1个或多个Partition中。发布消息的应用为Producer、消费消息的应用为Consumer，多个Consumer可以促成Consumer Group共同消费一个Topic中的消息。

| 概念/对象      | 简单说明                     |
| -------------- | ---------------------------- |
| Broker         | Kafka节点                    |
| Topic          | 主题，用来承载消息           |
| Partition      | 分区，用于主题分片存储       |
| Producer       | 生产者，向主题发布消息的应用 |
| Consumer       | 消费者，从主题订阅消息的应用 |
| Consumer Group | 消费者组，由多个消费者组成   |


![kafka-concepts](/images/kafka/kafka-concepts.png "kafka-concepts")

上图中包含了2个Producer（生产者），一个Topic（主题），3个Partition（分区），3个Replica（副本），3个Broker（Kafka实例或节点），一个Consumer Group（消费者组），其中包含3个Consumer（消费者）。下面我们逐一介绍这些概念。

### Producer（生产者）
生产者，顾名思义，就是生产东西的，也就是发送消息的，生产者每发送一个条消息必须有一个Topic（主题），也可以说是消息的类别，生产者源源不断的向kafka服务器发送消息。

### Topic（主题）
每一个发送到Kafka的消息都有一个主题，也可叫做一个类别，类似我们传统数据库中的表名一样，比如说发送一个主题为order的消息，那么这个order下边就会有多条关于订单的消息，只不过kafka称之为主题，都是一样的道理。
### Partition（分区）
生产者发送的消息数据Topic会被存储在分区中，这个分区的概念和ElasticSearch中分片的概念是一致的，都是想把数据分成多个块，好达到我们的负载均衡，合理的把消息分布在不同的分区上，分区是被分在不同的Broker上也就是服务器上，这样我们大量的消息就实现了负载均衡。每个Topic可以指定多个分区，但是至少指定一个分区。每个分区存储的数据都是有序的，不同分区间的数据不保证有序性。因为如果有了多个分区，消费数据的时候肯定是各个分区独立开始的，有的消费得慢，有的消费得快肯定就不能保证顺序了。那么当需要保证消息的顺序消费时，我们可以设置为一个分区，只要一个分区的时候就只能消费这个一个分区，那自然就保证有序了。
### Replica（副本）
副本就是分区中数据的备份，是Kafka为了防止数据丢失或者服务器宕机采取的保护数据完整性的措施，一般的数据存储软件都应该会有这个功能。假如我们有3个分区，由于不同分区中存放的是部分数据，所以为了全部数据的完整性，我们就必须备份所有分区。这时候我们的一份副本就包括3个分区，每个分区中有一个副本，两份副本就包含6个分区，一个分区两份副本。Kafka做了副本之后同样的会把副本分区放到不同的服务器上，保证负载均衡。讲到这我们就可以看见，这根本就是传统数据库中的主从复制的功能，没错，Kafka会找一个分区作为主分区（leader）来控制消息的读写，其他的（副本）都是从分区（follower），这样的话读写可以通过leader来控制，然后同步到副本上去，保证的数据的完整性。如果有某些服务器宕机，我们可以通过副本恢复数据，也可以暂时用副本中的数据来使用。
### Broker（实例或节点）
这个就好说了，意思就是Kafka的实例，启动一个Kafka就是一个Broker，多个Brokder构成一个Kafka集群，这就是分布式的体现，服务器多了自然吞吐率效率啥的都上来了。
### Consumer Group（消费者组）和 Consumer（消费者）
Consume消费者来读取Kafka中的消息，可以消费任何Topic的数据，多个Consume组成一个消费者组，一般的一个消费者必须有一个组（Group）名，如果没有的话会被分一个默认的组名。


## kafka 集群架构
Kafka是作为一个集群建立和运行的——一组服务器或代理管理两种类型的客户端，生产者和消费者(一般Pub/Sub术语中的发布者和订阅者)之间的通信。

根据硬件特性的不同，即使一个broker代理也足以形成一个每秒处理数万甚至数十万个事件的集群。但是为了获得高可用性和防止数据丢失，建议至少设置三个broker代理。

![Kafka-cluster-architecture](/images/kafka/Kafka-cluster-architecture.webp "Kafka-cluster-architecture")

集群架构和相关重要组件
![Kafka-cluster-architecture](/images/kafka/Kafka-cluster-architecture-2.png "Kafka-cluster-architecture-v2")

集群中的一个代理被自动选为控制器。它负责诸如故障监视之类的管理任务。为了协调集群内的代理，Kafka使用了一个单独的服务——[Apache ZooKeeper](https://zookeeper.apache.org/)。


![Kafka-cluster-architecture](/images/kafka/Kafka-cluster-architecture.png "Kafka-cluster-architecture")

上图该kafka集群A,该集群有8个实例broker节点，集群中的主题有8个分区(p0-p7),副本因子是3，也就是说每份数据存3份，每个分区都有1个leader 和2个follwer,以第一个broker为例，该broker 有3个分区，p1 分区为leader,p1分区上的所有读写请求都是由这个broker 进行处理的，p0和p2 分区是follwer,因此该broker 只负责p0和p2 从p1 中同步数据，而不处理这两个follwer 分区的读写请求。


## 环境搭建-本地伪分布式安装
一台主机部署3个broker的伪分布式集群实例演示，演示版本为 kafka_2.12-2.1.0,[下载地址](https://kafka.apache.org/downloads.html)，Kafka启动方式有Zookeeper和Kraft，两种方式只能选择其中一种启动，不能同时使用。这里选择的是zookeeper
```shell
cd /usr/local
tar -zxvf kafka_2.12-2.1.0.tgz
cd kafka_2.12-2.1.0
mkdir etc
cp config/zookeeper.properties etc/
# config/server.properties 是 kafka broker实例的配置文件，这里要搭建一个三节点的集群所以需要拷贝3份
cp config/server.properties etc/server-0.properties
cp config/server.properties etc/server-1.properties
cp config/server.properties etc/server-2.properties
```
修改broker 配置文件`server-0.properties`的配置，分别是2 处
- `listeners=PLAINTEXT://:9092` 监听端口注释去掉
- `log.dirs=/tmp/kafka-logs` 进行区分 `log.dirs=/tmp/kafka-logs-0`

修改broker 配置文件`server-1.properties`的配置，分别是3 处
- `broker.id=1` 防止和 server-0 重复
- `listeners=PLAINTEXT://:9093` 监听端口注释去掉,并更改为9093
- `log.dirs=/tmp/kafka-logs` 进行区分 `log.dirs=/tmp/kafka-logs-1`

修改broker 配置文件`server-2.properties`的配置，分别是3 处
- `broker.id=2` 防止和 server-0 重复
- `listeners=PLAINTEXT://:9094` 监听端口注释去掉,并更改为9094
- `log.dirs=/tmp/kafka-logs` 进行区分 `log.dirs=/tmp/kafka-logs-2`

检查下修改配置
```shell
root@k8s-master01:/usr/local/kafka_2.12-2.1.0/etc# cat server-*.properties |grep -e  broker.id -e listeners= -e log.dirs=|grep -v \#
broker.id=0
listeners=PLAINTEXT://:9092
log.dirs=/tmp/kafka-logs
broker.id=1
listeners=PLAINTEXT://:9093
log.dirs=/tmp/kafka-logs-1
broker.id=2
listeners=PLAINTEXT://:9094
log.dirs=/tmp/kafka-logs-2
```
启动实例

```shell
cd /usr/local/kafka_2.12-2.1.0/bin
# 先启动zookeeper
./zookeeper-server-start.sh ../etc/zookeeper.properties
# 再启动kafka 的三个实例，新开一个会话以后台进程的方式启动
./kafka-server-start.sh -daemon ../etc/server-0.properties
./kafka-server-start.sh -daemon ../etc/server-1.properties
./kafka-server-start.sh -daemon ../etc/server-2.properties
# 查看服务是否正常启动，查看端口监听
$ netstat -anp|grep -e 9092 -e 9093 -e 9094
tcp6       0      0 :::9094                 :::*                    LISTEN      34724/java          
tcp6       0      0 :::9092                 :::*                    LISTEN      33537/java          
tcp6       0      0 :::9093                 :::*                    LISTEN      34355/java
```
备注：启动Kafka本地环境需JDK 8+以上

创建主题测试部署的集群,创建一个test 名称的topic
```shell
root@k8s-master01:/usr/local/kafka_2.12-2.1.0/bin# ./kafka-topics.sh --zookeeper localhost:2181 --create --topic test --partitions 3 --replication-factor 2
Created topic "test".
root@k8s-master01:/usr/local/kafka_2.12-2.1.0/bin# ./kafka-topics.sh --zookeeper localhost:2181 --describe --topic test 
Topic:test	PartitionCount:3	ReplicationFactor:2	Configs:
	Topic: test	Partition: 0	Leader: 1	Replicas: 1,0	Isr: 1,0
	Topic: test	Partition: 1	Leader: 2	Replicas: 2,1	Isr: 2,1
	Topic: test	Partition: 2	Leader: 0	Replicas: 0,2	Isr: 0,2
```
创建好了主题topic 就可以使用生产者和消费者进行消息的发送和读取了。通过`kafka-console-consumer.sh`脚本模拟消费，会出现一个光标，等待显示消息
```shell
root@k8s-master01:/usr/local/kafka_2.12-2.1.0/bin# ./kafka-console-consumer.sh --bootstrap-server localhost:9092,localhost:9093,localhost:9094 --topic test
```
提供一个`kafka-console-producer.sh`脚本进行发送消息`message`
```shell
root@k8s-master01:/usr/local/kafka_2.12-2.1.0/bin# ./kafka-console-producer.sh --broker-list localhost:9092,localhost:9093,localhost:9094 --topic test
>message
```
### 监听器和内外部网络
总结一下kafka server.proerties 必要配置项
- broker.id
- logs.dirs
- zookeeper.connect

监听器

listeners: 指定broker 启动时本机的监听名称、端口
- listeners=PLAINTEXT://:9092  				协议：PLAINTEXT
- listeners=PLAINTEXT://192.168.1.11:9092        协议：SSL
- listeners=PLAINTEXT://hostname:9092      		协议：SASL_PLAINTEXT
- listeners=PLAINTEXT://0.0.0.0:9092      		协议：SASL_SSL

listeners: 指定broker 启动时的本机监听端口，`给服务器使用`  
advertised.listeners: 对外发布的访问IP和端口，注册到zookeeper 中，`给客户端(client)使用`

更多详细配置请参考[KAFAK 配置内外网分流，实现同时支持内网，外网，其他网络](https://www.cnblogs.com/wangcc7/p/17718020.html)

## kafka 消息模型
- 点对点
- 发布订阅
- 消息传播语义
  - 至少一次
  - 最多一次
  - 精确一次 
  ![kafka-consumer-model](/images/kafka/kafka-consumer-model.png "消费者模型")

- 分区是最小的并行单位
- 一个消费者可以消费多个分区
- 一个分区可以被多个消费组里消费者消费
- 注意：一个分区不能同时被一个消费者组里的多个消费者进行消费

### 点对点模式
![kafka-point-to-pointl](/images/kafka/kafka-point-to-point.png "点对点模式")

生产者发送一条消息到queue，只有一个消费者能收到。

### 发布订阅模式
![kafka-pub-and-sub](/images/kafka/kafka-pub-and-sub.png "发布订阅模式")

发布者发送到topic的消息，只有订阅了topic的订阅者才会收到消息。这里的订阅者也就是消费者，且每个消费者都属于不同的消费者组
### 消息传播语义
- 最多一次——消息可能会丢失，永远不重复发送
- 最少一次——消息不会丢失，但是可能会重复
- 精确一次——保障消息被传递到服务器且在服务端不重复



## Kafka API

| Producer API                     | Allows applications to send streams of data to topics in the Kafka cluster. |
| -------------------------------- | ------------------------------------------------------------ |
| Consumer API                     | Permits applications to read data streams from topics in the Kafka cluster. |
| Streams API                      | Acts as a stream processor, transforming data streams from input to output topics. |
| Connect API                      | Enables the development and running of reusable producers or consumers that connect Kafka topics to existing data system applications. |
| Admin API                        | Supports administrative operations on a Kafka cluster, like creating or deleting topics. |
| Kafka API use cases              | Real-time analytics, event sourcing, log aggregation, message queuing, and stream processing. |
| Kafka API compatible alternative | Redpanda streaming data platform.                            |

Kafka API提供了一个编程接口，允许应用程序实时生成、消费和处理记录流。它作为Kafka服务器和用户之间的通信点，以低延迟的规模处理实时数据馈送。这些属性使它成为涉及实时分析、事件溯源和许多其他数据密集型操作的用例的首选工具。

[Kafka API  架构](https://redpanda.com/guides/kafka-alternatives/kafka-api)

![Kafka-API-architecture](/images/kafka/Kafka-API-architecture.png "Kafka API 架构")

## 序列化
- 序列化与反序列化
- 常用消息格式
  - csv 适合简单的消息
  - json 1. 可读性高 2. 占用空间大
  - 序列化消息 1. avro hadoop、hive 支持好 2. protobuf 
- Avro与Schema

序列化是指将对象以二进制的方式在网络之间传输或者保存到文件中，并可以根据特定的规则进行还原

序列化的好处
1. 节省空间，提高网络传输效率
2. 跨平台
3. 跨语言

序列化和反序列化形象解释  

![kafka-serdes](/images/kafka/kafka-serdes.png "kafka-serdes")

[关于序列化和反序列参考](https://data-flair.training/blogs/kafka-serialization-and-deserialization/)


## 参考
- https://kafka1x.apachecn.org/intro.html
- https://kafka.apache.org/
- https://developer.confluent.io/quickstart/kafka-on-confluent-cloud/
- https://developer.confluent.io/what-is-apache-kafka/
- https://kafka.apache.org/downloads
- https://www.cnblogs.com/xxeleanor/p/15018271.html
- https://www.altexsoft.com/blog/apache-kafka-pros-cons/
- [kafka的发行版版本选择](https://cloud.tencent.com/developer/article/1582476)
- https://www.infoq.cn/article/apache-kafka/
- https://www.cloudduggu.com/kafka/architecture/