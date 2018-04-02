# Ingress
是一种API对象用来从外部访问集群当中服务，通常是通过HTTP协议。 Ingress可以提供负载均衡，SSL终端和基于名字的虚拟主机。

## 术语
为了阅读这篇文章的便利，让读者更能清楚的区别一些使用的在有些情况和地方可以互换的专门用语产生歧义。在这里做一些澄清。

* 节点：Kubernetes集群当中的一个物理或者虚拟的机器
* 集群： 被Kubernetes管理起来的计算单元，他们一般是被防火墙隔离与互联网之外的。
* 边缘路由器：用来为你的集群加强防火墙策略的路由器。一般是云服务供应商提供的或者是硬件设备的某部分来管理入口。
* 集群网络：一组物理的或者逻辑上的连接，它根据Kubernetes网络模型来辅助集群内部的通信。这方面的例子有Overlays的flannel，SDN的OVS
* 服务：Kubernetes的服务是使用标签选择器标记的一组Pods。除非申明，Services是具有虚拟IP地址只能在集群内部路由


## 什么是Ingress
通常情况下，服务和pods都有IP地址并只能被集群网络路由。所有终止于边缘路由器的流量要么被丢弃要么被转发。概念上来讲可以这么表示：
```
  internet
     |
 ----------
 [ Services]
```
Ingress是一个允许入站到集群内部服务的连接规则集合
```
    internet
       |
   [Ingress]
 ---|-----|---
   [Services]
```
它可以被配置成让Service具有被外部访问的URL，负载均衡，SSL终端，基于名字的虚拟主机，或更多。用户通过POST发送Ingress资源到APIServer来请求ingress。
ingress控制器负责实现Ingress，尽管你会配置你自己的边缘路由器或额外的前端来以高可用的方式处理流量的分发但通常APIServer都会自带一个负载均衡器。

### 前提条件
在你使用Ingress资源之前，有些事情你的懂得。Ingress只是个beta版本的资源，他并不在Kubernetes1.1版本前可用。简单的创建资源不会起作用，Ingree的需要一个
控制器。

GCE 谷歌的Kubernetes引擎在主节点上配置了一个ingress控制器。你可以在一个pod里面部署任意数量的自定义ingress控制器。你必须给每一个ingress加上一个合适
的类注释。

使用之前请阅读beta版本控制器的限制。 在非GCE/谷歌Kubernetes引擎环境中，ingress引擎是以Pod的形式被部署的

## Ingress资源
一个ingree最小资源如下：
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  spec:
    rules:
    - http:
        paths:
        - path: /testpath
          backend:
            serviceName: test
            servicePort: 80
```
如果不配置ingress控制器，用这个资源文件去以POST形式发送到APIServer上时不会用任何的影响

第1-6行： 和其他的Kubernetes配置一样，一个Ingress需要apiVersion，kind和metadata字段。 

第7-9行： Ingress规范规定了所有需要去创建一个负载均衡器或者代理服务器的配置信息。 最重要的是，它包含一个满足所有集群入站请求的规则列表。目前ingress
规则只支持http。

第10-11行： 每一个http规则都包含如下的信息：一个主机域名（比如：foo.bar.com，这个例子中是默认的*），路劲列表（比如：/testpath）每一个都和后端的
Services关联（test：80）。在负载均衡器指向流量到后端前这两个主机名和路径必须符合指定的请求者。

第12-14行：后端正如service doc中描述的是一个service：port组合。Ingress的流量通常直接被发送到符合的endpoints上。

全局参数：为了尽可能的简化这个例子，这个ingress描述没有指定全局参数。可以配置一个默认的全局后端以防在没有任何符合描述规范中的path的请求被发送到后端的
ingress控制器。

## Ingress控制器
为了让ingress资源工作集群中必须有个正在运行的控制器。这不像其他运行在kube-controlle-管理器中的类型的控制器，他们在集群被创建是就自动运行了。你自己
选择一个能符合你集群的ingress控制器的实现。 我们目前支持和维护GCE和nginx控制器。

## 预备工作
以下的文档描述了一个以Ingress资源暴露的跨平台功能。 理想情况下，所有的Ingress控制器都应该实现这个规范，但是我们的目的不是这个。 我们目前支持和维护
GCE和nginx控制器。请阅读控制器规范以了解每一个的概括

## Ingress的类型
### 单一服务Ingress
Kubernetes中一个已有的概念，你可以对外暴露一个单一的服务，然后你也可以通过Ingress做到，只需要通过一个没有任何规则和默认的后端
```
apiVersion: extension/v1beta1
kind: Ingress
metadata:
  name: test-ingress
sepc:
  backend:
    serviceName: testsvc
    servicePort: 80
```


### 简单的分裂
之前我们讨论过，Kubernetes中的Pod的IP地址只能在集群网络中可见， 所以我们要需要在边缘处借助什么工具来接受ingress的流量并代理到正确的endpoints上去。
通常这个组件是一个高可用的负载均衡器。一个Ingress就能让你所使用的负载均衡器的数量最小化。
```
foo.bar.com -> 178.91.123.132 -> /foo  s1:80
                                 /bar  s2:80
```
实现它我们只需要这样的资源文件：
``` 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          serviePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```
一但S1和S2被创建了，Ingress控制器将会为Ingress准备特定的负载均衡器实现。当所有的准备结束了你就能在ngress描述的最后一列看到负载均衡器的地址信息了

### 基于名称的虚拟主机
基于名字的虚拟主机对为相同的IP地址生成多个主机名。
```
foo.bar.com ---|                 |-> foo.bar.com s1:80
               | 178.91.123.132  |
bar.foo.com ---|                 |-> bar.foo.com s2:80
```
以下的Ingress对象描述了负载均衡器怎样基于主机头部信息路由请求。
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: s1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: s2
          servicePort: 80
```
默认的后端：一个没有规则的Ingress，就像之前的部分描述的，他会把所有的流量都发送到默认的后端上去。你也可以用同样的技术告诉负载均衡器怎么去找你的404页面通过一定的规则和默认的后端。

## TLS
你可以提供TLS的私钥和证书给Secret对象来为Ingress设置安全性。
















































































