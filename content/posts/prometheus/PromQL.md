---
title: "PromQL"
subtitle: ""
date: 2022-08-29T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Prometheus"]
tags: ["Prometheus"]
---
![PromQL](/images/prometheus/promql/promql.png "PromQL")
## 介绍
PromQL 是 Prometheus 内置的数据查询语言，其提供对时间序列数据丰富的查询，聚合以及逻辑运算能力的支持。
并且被广泛应用在 Prometheus 的日常应用当中，包括对数据查询、可视化、告警处理。可以这么说，PromQL 是 Prometheus 所有应用场景的基础，理解和掌握 PromQL 是我们使用 Prometheus 必备的技能。

### PromQL 简介
Prometheus Query Language (PromQL) 是一个专为Prometheus监控系统设计的强大查询语言，
它允许用户对收集的时间序列数据进行高效、灵活的查询和分析。PromQL的设计哲学在于提供简洁而强大的语法，以支持复杂的数据检索和实时监控场景
#### Prometheus和PromQL的关系
Prometheus是一个开源的系统监控和警报工具包，广泛用于云原生环境中。它通过收集和存储时间序列数据，支持实时监控和警报。
PromQL作为Prometheus的核心组件，允许用户通过强大的查询语言对这些数据进行检索和分析。无论是简单的数据查看还是复杂的性能分析，
PromQL都能够提供必要的工具来满足用户的需求。
#### PromQL的设计哲学
PromQL的设计哲学围绕着几个关键点：灵活性、表现力和性能。它旨在提供足够的灵活性，以支持从简单到复杂的各种查询需求，
同时保持查询表达式的简洁性。此外，PromQL经过优化以支持高效的数据处理和检索，这对于实时监控系统来说至关重要。

## PromQL基础概念
### 数据类型和结构
在PromQL中，主要操作以下几种数据类型：
#### 即时向量（Instant Vector）
即时向量是一个时间点上的一组时间序列，每个时间序列具有一个唯一的标签集合和一个数值。它通常用于表示某一瞬间的系统状态。

假设我们有一个监控系统的CPU使用率的时间序列，其查询表达式可能如下：
```
cpu_usage{host="server01"}
```
该查询返回“server01”主机上最新的CPU使用率数据。
#### 区间向量（Range Vector）
区间向量是在一段时间范围内的一组时间序列，它可以用来分析时间序列的变化趋势或计算时间序列的移动平均等。

要查询过去5分钟内`server01`主机的CPU使用率数据：
```
cpu_usage{host="server01"}[5m]
```
#### 标量（Scalar）
标量是一个简单的数值类型，它不带有时间戳，通常用于数学计算或与时间序列数据的比较。

假设我们想要将`server01`主机的CPU使用率与一个固定阈值进行比较：
```
cpu_usage{host="server01"} > 80
```
这里“80”就是一个标量值。
#### 字符串（String）
字符串类型在PromQL中用得较少，主要用于标签值的展示。

### 时间序列（Time Series）和指标（Metrics）
#### 时间序列

在时间序列中的每一个点称为一个样本（sample），样本由以下三部分组成：

- 指标(metric)：metric name 和描述当前样本特征的 labelsets
- 时间戳(timestamp)：一个精确到毫秒的时间戳
- 样本值(value)： 一个 float64 的浮点型数据表示当前样本的值

示例
```
<--------------- metric ---------------------><-timestamp -><-value->
http_request_total{status="200", method="GET"}@1434417560938 => 94355
http_request_total{status="200", method="GET"}@1434417561287 => 94334

http_request_total{status="404", method="GET"}@1434417560938 => 38473
http_request_total{status="404", method="GET"}@1434417561287 => 38544

http_request_total{status="200", method="POST"}@1434417560938 => 4748
http_request_total{status="200", method="POST"}@1434417561287 => 4785
```
#### 指标类型
从存储上来讲所有的监控指标 metric 都是相同的，但是在不同的场景下这些 metric 又有一些细微的差异。 例如，在 Node Exporter 返回的样本中指标 node_load1 反应的是当前系统的负载状态，随着时间的变化这个指标返回的样本数据是在不断变化的。
而指标 node_cpu_seconds_total 所获取到的样本数据却不同，它是一个持续增大的值，因为其反应的是 CPU 的累计使用时间，从理论上讲只要系统不关机，这个值是会一直变大。

为了能够帮助用户理解和区分这些不同监控指标之间的差异，Prometheus 定义了 4 种不同的指标类型：
- Counter（计数器）
- Gauge（仪表盘）
- Histogram（直方图）
- Summary（摘要）

在 node-exporter 返回的样本数据中，其注释中也包含了该样本的类型。例如：
```
# HELP node_cpu_seconds_total Seconds the cpus spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="cpu0",mode="idle"} 362812.7890625
```
##### Counter
Counter (只增不减的计数器) 类型的指标其工作方式和计数器一样，只增不减。常见的监控指标，如 http_requests_total、node_cpu_seconds_total 都是 Counter 类型的监控指标。

Counter 是一个简单但又强大的工具，例如我们可以在应用程序中记录某些事件发生的次数，通过以时间序列的形式存储这些数据，我们可以轻松的了解该事件产生的速率变化。PromQL 内置的聚合操作和函数可以让用户对这些数据进行进一步的分析，例如，通过 rate() 函数获取 HTTP 请求量的增长率：
```
rate(prometheus_http_requests_total[5m])
or
rate(http_requests_total[5m])
```
##### Gauge
与 Counter 不同，Gauge（可增可减的仪表盘）类型的指标侧重于反应系统的当前状态。因此这类指标的样本数据可增可减。常见指标如：node_memory_MemFree_bytes（主机当前空闲的内存大小）、node_memory_MemAvailable_bytes（可用内存大小）都是 Gauge 类型的监控指标。
通过 Gauge 指标，用户可以直接查看系统的当前状态：
```
node_memory_MemFree_bytes
```
##### Histogram 和 Summary
除了 Counter 和 Gauge 类型的监控指标以外，Prometheus 还定义了 Histogram 和 Summary 的指标类型。Histogram 和 Summary 主用用于统计和分析样本的分布情况。

在大多数情况下人们都倾向于使用某些量化指标的平均值，例如 CPU 的平均使用率、页面的平均响应时间，这种方式也有很明显的问题，以系统 API 调用的平均响应时间为例：如果大多数 API 请求都维持在 100ms 的响应时间范围内，而个别请求的响应时间需要 5s，那么就会导致某些 WEB 页面的响应时间落到中位数上，而这种现象被称为长尾问题。

为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，统计延迟在 0~10ms 之间的请求数有多少而 10~20ms 之间的请求数又有多少。通过这种方式可以快速分析系统慢的原因。Histogram 和 Summary 都是为了能够解决这样的问题存在的，通过 Histogram 和Summary 类型的监控指标，我们可以快速了解监控样本的分布情况。

例如，指标 prometheus_tsdb_wal_fsync_duration_seconds 的指标类型为 Summary。它记录了 Prometheus Server 中 wal_fsync 的处理时间，通过访问 Prometheus Server 的 /metrics 地址，可以获取到以下监控样本数据：
```
# HELP prometheus_tsdb_wal_fsync_duration_seconds Duration of WAL fsync.
# TYPE prometheus_tsdb_wal_fsync_duration_seconds summary
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.5"} 0.012352463
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.9"} 0.014458005
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.99"} 0.017316173
prometheus_tsdb_wal_fsync_duration_seconds_sum 2.888716127000002
prometheus_tsdb_wal_fsync_duration_seconds_count 216
```
从上面的样本中可以得知当前 Prometheus Server 进行 wal_fsync 操作的总次数为 216 次，耗时 2.888716127000002s。其中中位数（quantile=0.5）的耗时为 0.012352463，9 分位数（quantile=0.9）的耗时为 0.014458005s。

在 Prometheus Server 自身返回的样本数据中，我们还能找到类型为 Histogram 的监控指标prometheus_tsdb_compaction_chunk_range_seconds_bucket：
```
# HELP prometheus_tsdb_compaction_chunk_range_seconds Final time range of chunks on their first compaction
# TYPE prometheus_tsdb_compaction_chunk_range_seconds histogram
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="100"} 71
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="400"} 71
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="1600"} 71
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="6400"} 71
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="25600"} 405
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="102400"} 25690
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="409600"} 71863
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="1.6384e+06"} 115928
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="6.5536e+06"} 2.5687892e+07
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="2.62144e+07"} 2.5687896e+07
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="+Inf"} 2.5687896e+07
prometheus_tsdb_compaction_chunk_range_seconds_sum 4.7728699529576e+13
prometheus_tsdb_compaction_chunk_range_seconds_count 2.5687896e+07
```
与 Summary 类型的指标相似之处在于 Histogram 类型的样本同样会反应当前指标的记录的总数(以 _count 作为后缀)以及其值的总量（以 _sum 作为后缀）。
不同在于 Histogram 指标直接反应了在不同区间内样本的个数，区间通过标签 le 进行定义。

## 查询
### 简单查询和表达式
PromQL Web UI 的 Graph 选项卡提供了简单的用于查询数据的入口
输入 up，然后点击 Execute，就能查到监控正常的 Target
通过标签选择器过滤出 job 为 kubernetes-pods 的监控
```
up{job="kubernetes-pods"}
```

注意此时是 up{job="node-exporter"}属于绝对匹配，PromQL 也支持如下表达式：
- !=：不等于；
- =~：表示等于符合正则表达式的指标；
-  !~：和=~类似，=~表示正则匹配，!~表示正则不匹配。
如果想要查看主机监控的指标有哪些，可以输入 node，会提示所有主机监控的指标：

![prom-1](/images/prometheus/promql/promql-1.png "查询")

假 如 想 要 查 询 Kubernetes 集 群 中 每 个 宿 主 机 的 磁 盘 总 量 ， 可 以 使 用node_filesystem_size_bytes

查询指定分区大小 node_filesystem_size_bytes{mountpoint="/"}：

或者是查询分区不是/boot，且磁盘是/dev/开头的分区大小（结果不再展示）：

```
node_filesystem_size_bytes{device=~"/dev/.*", mountpoint!="/boot"}
```
查询主机 `192.168.1.11` 在最近 5 分钟可用的磁盘空间变化：
```
node_filesystem_size_bytes{mountpoint="/",instance="192.168.1.11:9100"}[5m]
```
目前支持的范围单位如下：
- s：秒
- m：分钟
- h：小时
- d：天
- w：周
- y：年

查询 10 分钟之前磁盘可用空间，只需要指定 offset 参数即可
```
node_filesystem_avail_bytes{instance="192.168.1.11:9100", mountpoint="/", device="/dev/mapper/centos-root"} offset 10m
```
查询 10 分钟之前，5 分钟区间的磁盘可用空间的变化
```
node_filesystem_avail_bytes{instance="192.168.1.11:9100", mountpoint="/", device="/dev/mapper/centos-root"}[5m] offset 10m
```
### 使用标签选择器（Label Selectors）过滤数据
多个标签通过逗号进行分隔
```
kubelet_http_requests_total{method="GET",path =~ "metrics.*"}
```
查看主机过去5分钟已使用内存
```
node_memory_Active_bytes[5m]
```
查询结果
![promql-5](/images/prometheus/promql/promql-5.png "Label Selectors")

查看`10.10.192.220`主机一小时前的使用内存
```
node_memory_Active_bytes{instance="10.10.192.220:9100"} offset 1h
```

### 高级查询
使用PromQL除了能够方便的按照查询和过滤时间序列以外，PromQL还支持丰富的操作符，用户可以使用这些操作符对进一步的对事件序列进行二次加工。这些操作符包括：数学运算符，逻辑运算符，布尔运算符等等。

#### 数学运算
例如，我们可以通过指标node_memory_MemFree_bytes获取当前主机可用的内存空间大小，其样本单位为Bytes。这是如果客户端要求使用MB作为单位响应数据，那只需要将查询到的时间序列的样本值进行单位换算即可

```
node_memory_MemFree_bytes / 1024 / 1024
```

PromQL支持的所有数学运算符如下所示：
1. {{< raw >}}+ (加法) {{< /raw >}}
2. {{< raw >}}- (减法)  {{< /raw >}}
3. {{< raw >}}* (乘法)  {{< /raw >}}
4. {{< raw >}}/ (除法)  {{< /raw >}}
5. {{< raw >}}% (求余)  {{< /raw >}}
6. {{< raw >}}^ (幂运算) {{< /raw >}}


**使用布尔运算过滤时间序列**  
查看free 内存值大于512M的主机
```
(node_memory_MemFree_bytes / 1024 / 1024) > 512
```

目前，Prometheus支持以下布尔运算符如下：
1. != (不相等)
2. \> (大于)
3. < (小于)
4. \>= (大于等于)
5. <= (小于等于)


**使用集合运算符**  
使用瞬时向量表达式能够获取到一个包含多个时间序列的集合，我们称为瞬时向量。 通过集合运算，可以在两个瞬时向量与瞬时向量之间进行相应的集合操作。

如时钟未同步告警
```
min_over_time(node_timex_sync_status[1m]) == 0 and node_timex_maxerror_seconds >= 16
```
目前，Prometheus支持以下集合运算符：
1. and (并且)
2. or (或者)
3. unless (排除)

**操作符优先级**  
对于复杂类型的表达式，需要了解运算操作的运行优先级

例如，查询主机的CPU使用率，可以使用表达式：
```
100-(avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by(instance,job)* 100) > 80
```
其中irate是PromQL中的内置函数，用于计算区间向量中时间序列每秒的即时增长率。

#### 内置函数
Prometheus还提供了下列内置的聚合操作符，这些操作符作用域瞬时向量。可以将瞬时表达式返回的样本数据进行聚合，形成一个新的时间序列。

- sum (求和)  
- min (最小值)  
- max (最大值)  
- avg (平均值)  
- stddev (标准差)  
- stdvar (标准差异)  
- count (计数)  
- count_values (对value进行计数)  
- bottomk (后n条时序)  
- topk (前n条时序)  
- quantile (分布统计) 

使用聚合操作的语法如下：
```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
``` 
其中只有count_values, quantile, topk, bottomk支持参数(parameter)

without用于从计算结果中移除列举的标签，而保留其它标签。

by则正好相反，结果向量中只保留列出的标签，其余标签则移除。通过without和by可以按照样本的问题对数据进行聚合。

例如：
```
sum(http_requests_total) without (instance)
```
等价于
```
sum(http_requests_total) by (code,handler,job,method)
```
如果只需要计算整个应用的HTTP请求总量，可以直接使用表达式：
```
sum(http_requests_total)
```

count_values用于时间序列中每一个样本值出现的次数。count_values会为每一个唯一的样本值输出一个时间序列，并且每一个时间序列包含一个额外的标签。

例如：
```
count_values("count", http_requests_total)
```

topk和bottomk则用于对样本值进行排序，返回当前样本值前n位，或者后n位的时间序列。

获取HTTP请求数前5位的时序样本数据，可以使用表达式：
```
topk(5, http_requests_total)
```
quantile用于计算当前样本数据值的分布情况quantile(φ, express)其中0 ≤ φ ≤ 1。

例如，当φ为0.5时，即表示找到当前样本数据中的中位数：
```
quantile(0.5, http_requests_total)
```
