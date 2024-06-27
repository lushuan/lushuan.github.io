# K8s默认Service和Headless Service的区别

K8s Service有四种类型
- Service
- Headless Service
- NodePort Service
- LoadBalancer Service


Service 如果不指定则为默认类型
## Service
### Service是什么？
Service服务可以为一组具有相同功能的容器应用提供一个统一的入口地址。

### Service可以用来做什么？
我们都知道Pod在摧毁重建时ip地址是会动态变化的，这样通过客户端直接访问不合适了，这时候就可以选择使用服务来和Pod建立连接，通过标签选择器进行适配。这样就能有效的解决了Pod ip地址动态变换的问题了。

## Headless Service
headless service作为service的一种类型，它又解决了什么问题？headless service 顾名思义无头服务。

### 为什么需要无头服务？
1. 客户端想要和指定的的Pod直接通信
2. 并不是随机选择
3. 开发人员希望自己控制负载均衡的策略，不使用Service提供的默认的负载均衡的功能，或者应用程序希望知道属于同组服务的其它实例。

### Headless Service使用场景
有状态应用，例如数据库

例如主节点可以对数据库进行读写操作，而其它的两个工作节点只能读，在这里客户端就没必要指定pod服务的集群地址，直接指定数据库Pod ip地址即可，这里需要绑定dns，客户端访问dns，dns会自动返回pod IP地址列表

![server-and-Headless-Service](/images/kubernetes/blog/service-and-headless-service.png)

## 总结
- 无头服务不需要指定集群地址
- 无头服务适用有状态应用例如数据库
- 无头服务dns查询会返回pod列表，开发人员可以自定义负载均衡策略
- 普通Service可以通过负载均衡路由到不同的容器应用
