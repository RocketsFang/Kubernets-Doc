# DNS for Services and Pods

这篇翻译主要内容是DNS对Kubernetes的支持概览

## 简介

Kubernetes DNS在集群内部调度了一个DNS Pod和Service， 并设置了Kubelets告诉每一个Containers来使用DNS服务来解析DNS名称。

### 什么东西可以得到DNS名称

包括DNS服务器每一个在集群中的Service都会有一个DNS名称。 默认情况下一个客户端的Pod会去查找这个DNS列表，它包括Pod的命名空间和集群的默认域名。可以用这个
例子来解释：
假设在Kubernetes命名空间‘bar'中有一个名为’foo'的服务。同样一个运行在命名空间’bar‘的Pod可以简单的做一个DNS查找’foo‘来获取这个服务。如果一个运行在命名
空间’quux‘的Pod则要通过DNS查找’foo.bar'来获取这个相同的服务。
接下来的部分将详细介绍支持的记录和设计。

## Services
### DNS记录
‘普通‘（非无头部信息）服务被加入到DNS后，一条以这样的格式命名的记录会被创建在DNS服务器里 'my-svc.my-namespace.svc.cluster.local. 这样就能得到服务在集群内部的IP地址。







