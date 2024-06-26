# K8s删除Namesapce一直处于Terminating状态处理

k8s namespace 无法删除记录

名字空间yaml 文件
```Yaml
kind: Namespace
apiVersion: v1
metadata:
  name: kube-logging
```
删除卡住
```shell
$ sudo kubectl delete -f kube-logging.yaml --wait=true
namespace "kube-logging" deleted # 长久的等待
```

强制删除（无效果）
``` shell
sudo kubectl delete ns kube-logging  --grace-period=0 --force
```
- 排查思路
查看命名空间下的所有资源
```shell
kubectl api-resources -o name --verbs=list --namespaced | xargs -n 1 kubectl get --show-kind --ignore-not-found -n kube-logging
```
输出结果该命名空间下无关联的相关资源


### 解决方式
一行命令解决，注意替换两处待删命名空间字样
``` shell
kubectl get namespace "待删命名空间" -o json \
  | tr -d "\n" | sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/" \
  | kubectl replace --raw /api/v1/namespaces/待删命名空间/finalize -f -
```

### 总结
- 创建namesapce时注入finalizers会导致namespace无法
- 可以通过edit ns 的方式将finalizers 置空或者上面的一行命令解决的方式，本质是一致的

### 补充 K8s Finalizers
Finalizers 字段属于 Kubernetes GC 垃圾收集器，是一种删除拦截机制，能够让控制器实现异步的删除前（Pre-delete）回调。其存在于任何一个资源对象的 Meta[1] 中，在 k8s 源码中声明为 []string，该 Slice 的内容为需要执行的拦截器名称。

对带有 Finalizer 的对象的第一个删除请求会为其 metadata.deletionTimestamp 设置一个值，但不会真的删除对象。一旦此值被设置，finalizers 列表中的值就只能被移除。

当 metadata.deletionTimestamp 字段被设置时，负责监测该对象的各个控制器会通过轮询对该对象的更新请求来执行它们所要处理的所有 Finalizer。当所有 Finalizer 都被执行过，资源被删除。

metadata.deletionGracePeriodSeconds 的取值控制对更新的轮询周期

 当 finalizers 字段为空时，k8s 认为删除已完成。

## 参考
1. [k8s 如何让 ns 无法删除](https://blog.csdn.net/shenyuanhaojie/article/details/126249072)
2. [kubernetes_Namespace无法删除的解决方法](https://blog.csdn.net/Wqr_18390921824/article/details/124208510?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-1-124208510-blog-115963541.235%5Ev38%5Epc_relevant_sort_base2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-1-124208510-blog-115963541.235%5Ev38%5Epc_relevant_sort_base2&utm_relevant_index=2)
3. [K8S从懵圈到熟练 - 我们为什么会删除不了集群的命名空间？](http://www.taodudu.cc/news/show-1094934.html?action=onClick)
4. [k8s问题解决 - 删除命名空间长时间处于terminating状态](https://www.cnblogs.com/hellxz/p/17450248.html)
5. [熟悉又陌生的 k8s 字段：finalizers](http://www.taodudu.cc/news/show-2899643.html?action=onClick)
