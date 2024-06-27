# Prometheus 监控k8s 组件指标梳理

## 监控策略
|目标 |服务发现模式 |监控方法 |数据源|
|---|---|---|---|
|从集群各节点kubelet组件中获取节点kubelet的基本运行状态的监控指标|node |白盒监控|kubelet|
|从集群各节点kubelet内置的cAdvisor中获取，节点中运行的容器的监控指标|node |白盒监控|kubelet|
|从部署到各个节点的Node Exporter中采集主机资源相关的运行资源|node |白盒监控|node exporter|
|对于内置了Promthues支持的应用，需要从Pod实例中采集其自定义监控指标|pod |白盒监控|custom pod|
|获取API Server组件的访问地址，并从中获取Kubernetes集群相关的运行监控指标|endpoints |白盒监控|api server|
|获取集群中Service的访问地址，并通过Blackbox Exporter获取网络探测指标|service |黑盒监控|blackbox exporter|
|获取集群中Ingress的访问信息，并通过Blackbox Exporter获取网络探测指标|ingress |黑盒监控|blackbox exporter|

## k8s 插件采集指标说明
包含夜莺指标做横向对比参考
### Cadvisor 指标说明

`cpu 指标`

| 指标名                | 含义                                  | prometheus metrics或计算方式                                 | 说明                                 |
| :-------------------- | :------------------------------------ | :----------------------------------------------------------- | :----------------------------------- |
| cpu.util              | 容器cpu使用占其申请的百分比           | sum (rate (container_cpu_usage_seconds_total[1m])) by( container) /( sum (container_spec_cpu_quota) by(container) /100000) * 100 | 0-100的范围                          |
| cpu.idle              | 容器cpu空闲占其申请的百分比           | 100 - cpu.util                                               | 0-100的范围                          |
| cpu.user              | 容器cpu用户态使用占其申请的百分比     | sum (rate (container_cpu_user_seconds_total[1m])) by( container) /( sum (container_spec_cpu_quota) by(container) /100000) * 100 | 0-100的范围                          |
| cpu.sys               | 容器cpu内核态使用占其申请的百分比     | sum (rate (container_cpu_sys_seconds_total[1m])) by( container) /( sum (container_spec_cpu_quota) by(container) /100000) * 100 | 0-100的范围                          |
| cpu.cores.occupy      | 容器cpu使用占用机器几个核             | rate(container_cpu_usage_seconds_total[1m])                  | 0到机器核数上限,结果为1就是占用1个核 |
| cpu.spec.quota        | 容器的CPU配额                         | container_spec_cpu_quota                                     | 为容器指定的CPU个数*100000           |
| cpu.throttled.util    | 容器CPU执行周期受到限制的百分比       | 未录入                                                       | 0-100的范围                          |
| cpu.periods           | 容器生命周期中度过的cpu周期总数       | counter型无需计算                                            | 使用rate/increase 查看               |
| cpu.throttled.periods | 容器生命周期中度过的受限的cpu周期总数 | counter型无需计算                                            | 使用rate/increase 查看               |
| cpu.throttled.time    | 容器被节流的总时间 )                  | counter型无需计算                                            | 单位(纳秒                            |

`mem 指标`

| 夜莺指标名                   | 含义                                                 | prometheus metrics或计算方式                                 | 说明                                           |
| :--------------------------- | :--------------------------------------------------- | :----------------------------------------------------------- | :--------------------------------------------- |
| mem.bytes.total              | 容器的内存限制                                       | 无需计算                                                     | 单位byte 对应pod yaml中resources.limits.memory |
| mem.bytes.used               | 当前内存使用情况，包括所有内存，无论何时访问         | container_memory_rss + container_memory_cache + kernel memory | 单位byte                                       |
| mem.bytes.used.percent       | 容器内存使用率                                       | container_memory_usage_bytes/container_spec_memory_limit_bytes *100 | 范围0-100                                      |
| mem.bytes.workingset         | 容器真实使用的内存量，也是limit限制时的 oom 判断依据 | container_memory_max_usage_bytes > container_memory_usage_bytes >= container_memory_working_set_bytes > container_memory_rss | 单位byte                                       |
| mem.bytes.workingset.percent | 容器真实使用的内存量百分比                           | container_memory_working_set_bytes/container_spec_memory_limit_bytes *100 | 范围0-100                                      |
| mem.bytes.cached             | 容器cache内存量                                      | container_memory_cache                                       | 单位byte                                       |
| mem.bytes.rss                | 容器rss内存量                                        | container_memory_rss                                         | 单位byte                                       |
| mem.bytes.swap               | 容器cache内存量                                      | container_memory_swap                                        | 单位byte                                       |

`filesystem && disk.io 指标`

| 夜莺指标名              | 含义                       | prometheus metrics或计算方式                           | 说明         |
| :---------------------- | :------------------------- | :----------------------------------------------------- | :----------- |
| disk.bytes.total        | 容器可以使用的文件系统总量 | container_fs_limit_bytes                               | (单位：字节) |
| disk.bytes.used         | 容器已经使用的文件系统总量 | container_fs_usage_bytes                               | (单位：字节) |
| disk.bytes.used.percent | 容器文件系统使用百分比     | container_fs_usage_bytes/container_fs_limit_bytes *100 | 范围0-100    |
| disk.io.read.bytes      | 容器io.read qps            | rate(container_fs_reads_bytes_total)[1m]               | (单位：bps)  |
| disk.io.write.bytes     | 容器io.write qps           | rate(container_fs_write_bytes_total)[1m]               | (单位：bps)  |

`network 指标`

| 夜莺指标名                                                   | 含义                       | prometheus metrics或计算方式                               | 说明            |
| :----------------------------------------------------------- | :------------------------- | :--------------------------------------------------------- | :-------------- |
| net.in.bytes                                                 | 容器网络接收数据总数       | rate(container_network_receive_bytes_total)[1m]            | (单位：bytes/s) |
| net.out.bytes                                                | 容器网络积传输数据总数）   | rate(container_network_transmit_bytes_total)[1m]           | (单位：bytes/s) |
| net.in.pps                                                   | 容器网络接收数据包pps      | rate(container_network_receive_packets_total)[1m]          | (单位：p/s)     |
| net.out.pps                                                  | 容器网络发送数据包pps      | rate(container_network_transmit_packets_total)[1m]         | (单位：p/s)     |
| net.in.errs                                                  | 容器网络接收数据错误数     | rate(container_network_receive_errors_total)[1m]           | (单位：bytes/s) |
| net.out.errs                                                 | 容器网络发送数据错误数     | rate(container_network_transmit_errors_total)[1m]          | (单位：bytes/s) |
| net.in.dropped                                               | 容器网络接收数据包drop pps | rate(container_network_receive_packets_dropped_total)[1m]  | (单位：p/s)     |
| net.out.dropped                                              | 容器网络发送数据包drop pps | rate(container_network_transmit_packets_dropped_total)[1m] | (单位：p/s)     |
| container_network_{tcp,udp}_usage_total 默认不采集是因为 --disable_metrics=tcp, udp ,因为开启cpu压力大 |                            |                                                            |                 |

`system 指标`

| 夜莺指标名            | 含义                           | prometheus metrics或计算方式 | 说明       |
| :-------------------- | :----------------------------- | :--------------------------- | :--------- |
| sys.ps.process.count  | 容器中running进程个数          | container_processes          | (单位：个) |
| sys.ps.thread.count   | 容器中进程running线程个数      | container_threads            | (单位：个) |
| sys.fd.count.used     | 容器中打开文件描述符个数       | container_file_descriptors   | (单位：个) |
| sys.fd.soft.ulimits   | 容器中root process Soft ulimit | container_ulimits_soft       | (单位：个) |
| sys.socket.count.used | 容器中打开套接字个数           | container_sockets            | (单位：个) |
| sys.task.state        | 容器中task 状态分布            | container_tasks_state        | (单位：个) |

### kube-apiserver metrics 指标说明

|                         指标名                         |  类型   |                          含义                          |                             说明                             |
| :----------------------------------------------------: | :-----: | :----------------------------------------------------: | :----------------------------------------------------------: |
|                apiserver_request_total                 | counter |                对APIServer不同请求的计数。                 |  |
|         apiserver_request_duration_seconds_sum         |  gauge  |                    统计APIServer客户端对APIServer的访问时延。对APIServer不同请求的时延分布。                   |  |
|        apiserver_request_duration_seconds_count        |  gauge  |                     请求延迟记录数                     | 计算平均延迟:`apiserver_request_duration_seconds_sum`/`apiserver_request_duration_seconds_count` |
|apiserver_admission_controller_admission_duration_seconds_bucket|Gauge|准入控制器（Admission Controller）的处理延时。标签包括准入控制器名字、操作（CREATE、UPDATE、CONNECT等）、API资源、操作类型（validate或admit）和请求是否被拒绝（true或false）。|
|apiserver_admission_webhook_admission_duration_seconds_bucket|Gauge|准入Webhook（Admission Webhook）的处理延时。标签包括准入控制器名字、操作（CREATE、UPDATE、CONNECT等）、API资源、操作类型（validate或admit）和请求是否被拒绝（true或false）。|
|apiserver_admission_webhook_admission_duration_seconds_count|Counter|准入Webhook（Admission Webhook）的处理请求统计。标签包括准入控制器名字、操作（CREATE、UPDATE、CONNECT等）、API资源、操作类型（validate或admit）和请求是否被拒绝（true或false）。|
|              apiserver_response_sizes_sum              | counter |                   请求响应大小记录和                   |                                                              |
|             apiserver_response_sizes_count             | counter |                   请求响应大小记录数                   |                                                              |
|                authentication_attempts                 | counter |                       认证尝试数                       |                                                              |
|          authentication_duration_seconds_sum           | counter |                     认证耗时记录和                     |                                                              |
|         authentication_duration_seconds_count          | counter |                     认证耗时记录数                     |                                                              |
|          apiserver_tls_handshake_errors_total          | counter |                    tls握手失败计数                     |                                                              |
|  apiserver_client_certificate_expiration_seconds_sum   |  gauge  |                    证书过期时间总数                    |                                                              |
| apiserver_client_certificate_expiration_seconds_count  |  gauge  |                  证书过期时间记录个数                  |                                                              |
| apiserver_client_certificate_expiration_seconds_bucket |  gauge  |                    证书过期时间分布                    |                                                              |
|          apiserver_current_inflight_requests           |  gauge  | APIServer当前处理的请求数。包括ReadOnly和Mutating两种 |                                                              |
|           apiserver_current_inqueue_requests           |  gauge  |     是一个表向量， 记录最近排队请求数量的高水位线      | [apiserver请求限流](https://link.segmentfault.com/?enc=ddkdbN8V09qjT6ExKNHObQ%3D%3D.XxjHTjyhClA01FEr3RrstvUkqWEhABp9yRahO%2FJ6p%2BmJLvjhHiGTcsQpyByt9gQ6jMa1g7p8WJGvNRMyPlDTCTshRDXGOr5Egl6RT5Ctrwg%3D) |
|    apiserver_flowcontrol_current_executing_requests    |  gauge  |     记录包含执行中（不在队列中等待）请求的瞬时数量     |             APF api的QOS APIPriorityAndFairness              |
|     apiserver_flowcontrol_current_inqueue_requests     |  gauge  |        记录包含排队中的（未执行）请求的瞬时数量        |                                                              |
|                  workqueue_adds_total                  | counter |                       wq 入队数                        |                                                              |
|                workqueue_retries_total                 | counter |                       wq retry数                       |                                                              |
|      workqueue_longest_running_processor_seconds       |  gauge  |                    wq中最长运行时间                    |                                                              |
|          workqueue_queue_duration_seconds_sum          |  gauge  |                   wq中等待延迟记录和                   |                                                              |
|         workqueue_queue_duration_seconds_count         |  gauge  |                   wq中等待延迟记录数                   |                                                              |
|          workqueue_work_duration_seconds_sum           |  gauge  |                   wq中处理延迟记录和                   |                                                              |
|         workqueue_work_duration_seconds_count          |  gauge  |                   wq中处理延迟记录数                   |  |
|up |	Gauge |	服务可用性，1 表示服务可用，0 表示服务不可用|

**关键指标**：

> \$interval 表示间隔时间，例如 5m
> \$quantile 0 ≤ 值 ≤ 1，用于计算当前样本数据值的分布情况，当值为 0.5 时，即表示找到当前样本数据中的中位数



| 名称 | PromQL | 说明 |                                                                                                                                                               
| ---- | ---- | ---- |
| API QPS  | sum (irate(apiserver_request_total[$interval]))                                                                                                                             | APIServer总QPS        |
| 读请求成功率   | sum(irate(apiserver_request_total{code=\~"20.*",verb=\~"GET&#124;LIST"}[\$interval]))/sum(irate(apiserver_request_total{verb=~"GET&#124;LIST"}[\$interval])) | APIServer读请求成功率。      |
| 写请求成功率   | sum(irate(apiserver_request_total{code=\~"20.*",verb!\~"GET&#124;LIST&#124;WATCH&#124;CONNECT"}[\$interval]))/sum(irate(apiserver_request_total{verb!\~"GET&#124;LIST&#124;WATCH&#124;CONNECT"}[\$interval])) | APIServer写请求成功率。      |
| 在处理读请求数量 | sum(apiserver_current_inflight_requests{request_kind="readOnly"})                                                                                                           | 所有 APIServer 当前在处理读请求数量; 不加 sum() 显示每个 apiserver 在处理的读请求数量|
| 在处理写请求数量 | sum(apiserver_current_inflight_requests{request_kind="mutating"})                                                                                                           | APIServer当前在处理写请求数量（老版本 k8s requestKind） |
| GET读请求时延  | histogram_quantile(\$quantile, sum(irate(apiserver_request_duration_seconds_bucket{verb="GET",resource!="",subresource!\~"log&#124;proxy"}[\$interval])) by (pod, verb, resource, subresource, scope, le)) | 展示GET请求的响应时间，维度包括APIServer Pod、Verb(GET)、Resources（例如Configmaps、Pods、Leases等）、Scope（范围例如Namespace级别、Cluster级别）。      |
| LIST读请求时延 | histogram_quantile(\$quantile, sum(irate(apiserver_request_duration_seconds_bucket{verb="LIST"}[\$interval])) by (pod_name, verb, resource, scope, le)) | 展示LIST请求的响应时间，维度包括APIServer Pod、Verb(GET)、Resources（例如Configmaps、Pods、Leases等）、Scope（范围例如Namespace级别、Cluster级别）。     |
| 写请求时延     | histogram_quantile(\$quantile, sum(irate(apiserver_request_duration_seconds_bucket{verb!\~"GET&#124;WATCH&#124;LIST&#124;CONNECT"}[\$interval])) by (cluster, pod_name, verb, resource, scope, le))  | 展示Mutating请求的响应时间，维度包括APIServer Pod、Verb(GET)、Resources（例如Configmaps、Pods、Leases等）、Scope（范围例如Namespace级别、Cluster级别）。 |





### etcd metrics 指标说明

|               指标名                | 类型  |        含义        | 说明 |
| :---------------------------------: | :---: | :----------------: | :---: |
|     etcd_db_total_size_in_bytes     | gauge |   db物理文件大小   |      |
|         etcd_object_counts          | gauge | etcd对象按种类计数 |      |
|  etcd_request_duration_seconds_sum  | gauge | etcd请求延迟记录和 |      |
| etcd_request_duration_seconds_count | gauge | etcd请求延迟记录数 |      |

### kube-scheduler 指标说明

|                        指标名                         |  类型   |               含义               |   说明   |
| :---------------------------------------------------: | :-----: | :------------------------------: | :------: |
|     scheduler_e2e_scheduling_duration_seconds_sum     |  gauge  |       端到端调度延迟记录和       |          |
|    scheduler_e2e_scheduling_duration_seconds_count    |  gauge  |       端到端调度延迟记录数       |          |
|     scheduler_pod_scheduling_duration_seconds_sum     |  gauge  |          调度延迟记录和          | 分析次数 |
|    scheduler_pod_scheduling_duration_seconds_count    |  gauge  |          调度延迟记录数          |          |
|                scheduler_pending_pods                 |  gauge  |      调度队列pending pod数       |          |
|          scheduler_queue_incoming_pods_total          | counter |        进入调度队列pod数         |          |
|  scheduler_scheduling_algorithm_duration_seconds_sum  |  gauge  |        调度算法延迟记录和        |          |
| scheduler_scheduling_algorithm_duration_seconds_count |  gauge  |        调度算法延迟记录数        |          |
|         scheduler_pod_scheduling_attempts_sum         |  gauge  | 成功调度一个pod 的尝试次数记录和 |          |
|        scheduler_pod_scheduling_attempts_count        |  gauge  | 成功调度一个pod 的尝试次数记录数 |          |

### coredns 指标说明

|                   指标名                   |  类型   |        含义        |            说明            |
| :----------------------------------------: | :-----: | :----------------: | :------------------------: |
|         coredns_dns_requests_total         | counter |     解析请求数     | A记录，AAAA记录，other记录 |
|        coredns_dns_responses_total         | counter |     解析响应数     | NOERROR，NXDOMAIN，REFUSED |
|           coredns_cache_entries            |  gauge  |     缓存记录数     |         成功或失败         |
|          coredns_cache_hits_total          | counter |     缓存命中数     |         成功或失败         |
|         coredns_cache_misses_total         | counter |    缓存未命中数    |         成功或失败         |
|  coredns_dns_request_duration_seconds_sum  |  gauge  |   解析延迟记录和   |                            |
| coredns_dns_request_duration_seconds_count |  gauge  |   解析延迟记录数   |                            |
|    coredns_dns_response_size_bytes_sum     |  gauge  | 解析响应大小记录和 |                            |
|   coredns_dns_response_size_bytes_count    |  gauge  | 解析响应大小记录数 |                            |



### kube-stats-metrics 指标说明

`pod metrics`

|                      指标名                       |  类型   |                             含义                             |
| :-----------------------------------------------: | :-----: | :----------------------------------------------------------: |
|               kube_pod_status_phase               |  gauge  |   pod状态统计:Pending，Succeeded，Failed，Running，Unknown   |
|         kube_pod_container_status_waiting         | counter |             pod处于waiting状态，值为1代表waiting             |
|     kube_pod_container_status_waiting_reason      |  gauge  | pod处于waiting状态原因：ContainerCreating，CrashLoopBackOff pod启动崩溃,再次启动然后再次崩溃，CreateContainerConfigError，ErrImagePull，ImagePullBackOff，CreateContainerError，InvalidImageName |
|       kube_pod_container_status_terminated        |  gauge  |          pod处于terminated状态，值为1代表terminated          |
|    kube_pod_container_status_terminated_reason    |  gauge  | pod处于terminated状态原因：OOMKilled，Completed，Error，ContainerCannotRun，DeadlineExceeded，Evicted |
|     kube_pod_container_status_restarts_total      | counter |                     pod中的容器重启次数                      |
|  kube_pod_container_resource_requests_cpu_cores   |  gauge  |                       pod容器cpu limit                       |
| kube_pod_container_resource_requests_memory_bytes |  gauge  |                 pod容器mem limit(单位:字节)                  |

`deployment metrics`

|                   指标名                    | 类型  |         含义          |
| :-----------------------------------------: | :---: | :-------------------: |
|       kube_deployment_status_replicas       | gauge |    dep中的pod num     |
|  kube_deployment_status_replicas_available  | gauge |  dep中的 可用pod num  |
| kube_deployment_status_replicas_unavailable | gauge | dep中的 不可用pod num |

`daemonSet metrics`

|                     指标名                     | 类型  |           含义           |
| :--------------------------------------------: | :---: | :----------------------: |
|     kube_daemonset_status_number_available     | gauge |        ds 可用数         |
|    kube_daemonset_status_number_unavailable    | gauge |       ds 不可用数        |
|       kube_daemonset_status_number_ready       | gauge |        ds ready数        |
|   kube_daemonset_status_number_misscheduled    | gauge | 未经过调度运行ds的节点数 |
| kube_daemonset_status_current_number_scheduled | gauge |     ds目前运行节点数     |
| kube_daemonset_status_desired_number_scheduled | gauge |    应该运行ds的节点数    |

`daemonSet metrics`

|                  指标名                  | 类型  |      含义      |
| :--------------------------------------: | :---: | :------------: |
|     kube_statefulset_status_replicas     | gauge |   ss副本总数   |
| kube_statefulset_status_replicas_current | gauge |  ss当前副本数  |
| kube_statefulset_status_replicas_updated | gauge | ss已更新副本数 |
|        kube_statefulset_replicas         | gauge |  ss目标副本数  |

`Job metrics`

|          指标名           | 类型  |       含义        |
| :-----------------------: | :---: | :---------------: |
|  kube_job_status_active   | gauge | job running pod数 |
| kube_job_status_succeeded | gauge |  job 成功 pod数   |
|  kube_job_status_failed   | gauge |  job 失败 pod数   |
|     kube_job_complete     | gauge |   job 是否完成    |
|      kube_job_failed      | gauge |   job 是否失败    |

`CronJob metrics`

|                 指标名                 | 类型  |       含义        |
| :------------------------------------: | :---: | :---------------: |
|       kube_cronjob_status_active       | gauge | job running pod数 |
|       kube_cronjob_spec_suspend        | gauge | =1代表 job 被挂起 |
|    kube_cronjob_next_schedule_time     | gauge | job 下次调度时间  |
| kube_cronjob_status_last_schedule_time | gauge | job 下次调度时间  |

`PersistentVolume metrics`

|                指标名                | 类型  |                        含义                        |
| :----------------------------------: | :---: | :------------------------------------------------: |
| kube_persistentvolume_capacity_bytes | gauge |                     pv申请大小                     |
|  kube_persistentvolume_status_phase  | gauge | pv状态:Pending，Available，Bound，Released，Failed |

`PersistentVolumeClaim metrics`

|                           指标名                           | 类型  |             含义             |
| :--------------------------------------------------------: | :---: | :--------------------------: |
| kube_persistentvolumeclaim_resource_requests_storage_bytes | gauge |       pvc request大小        |
|          kube_persistentvolumeclaim_status_phase           | gauge | pvc状态:Lost，Bound，Pending |

### node metrics 指标说明

|                  指标名                   | 类型  |                             含义                             |
| :---------------------------------------: | :---: | :----------------------------------------------------------: |
|        kube_node_status_condition         | gauge | condition:NetworkUnavailable，MemoryPressure，DiskPressure，PIDPressure，Ready |
|  kube_node_status_allocatable_cpu_cores   | gauge |                     节点可以分配cpu核数                      |
| kube_node_status_allocatable_memory_bytes | gauge |               节点可以分配内存总量(单位：字节)               |
|           kube_node_spec_taint            | gauge |                         节点污点情况                         |
|  kube_node_status_capacity_memory_bytes   | gauge |                   节点内存总量(单位：字节)                   |
|    kube_node_status_capacity_cpu_cores    | gauge |                         节点cpu核数                          |
|      kube_node_status_capacity_pods       | gauge |                     节点可运行的pod总数                      |

