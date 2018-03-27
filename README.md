# Services

Kubernetes Pods不是永生的。他们可以被创造也有销毁的时候，他们不能自生。ReplicationControllers就是专门用来自动地创建和销毁Pod的（比如:扩容或者进行交替升级的时候）
但当Pod一担获得自己的ip地址时，尽管这些IP地址随着时间的推移并不总是那么的稳定。这就会导致一个问题：在Kubernetes集群中一些Pod（我们姑且称之为后端）要提供特定
的功能给其他Pod（我们称之为前端），这些前段Pod怎么能够发现并且跟踪这些后端的Pod？

我们进入Service的讨论

Kubernetes中的Service是一种抽象，它定义了一组逻辑上的Pod以及被其他我们称之为Micro-Service的对象怎么去访问这些Pod的策略。由Service所指定的这组Pods通常是由
Label Selector所指定的（本文我们也会谈谈什么时候我们不需要选择器指定的Service）。

请看这个例子，设想一下有这样一个处理图像的后端，它由3个Pod副本。这些副本之间是可以互换的-即前段不需要知道具体是由那个Pod提供的处理功能。不可否认的是每次正真
处理图像的Pod是不同的，这些是对前端透明的它不需要知道而且也不需要维护一个这样的后端的列表。Service抽象在这里就发挥了作用它解耦了彼此间的依赖。

对于Kubernetes的系统应用，Kubernetes提供了一个简单的Endpoints API设计，它能够在Pod组发生变化时自动进行更新；对于非Kubernetes应用，Kubernetes则提供了虚拟
IP指向Service而后者将请求重定向到具体的应用。

# 定义Services

Kubernetes中的Service简单来说和Pod一样就是一个Rest对象，和所有的Rest对象一样，Service的定义能够被POST转发到APIServer借以创建一个新的对象。比如，你有一组Pod
他们每个Pod都对外暴露9376端口，并且他们也都定义了一个‘app=MyApp’的标签。
```
king: Service
apiVersion: v1
matadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
 ```

这段描述将会创建一个名称为“my-service”的Service对象，他会将目标TPC端口为9376的访问重定向到有“app=MyApp”标签的Pod。这个Service也被指定了一个IP地址（有
 时被称之为集群IP地址），service代理会使用这个地址。Service选择器会不断的检查并且所有的访问结果都会被POST到一样被称为“My-Service"的Endpoints对象。
 
 我们需要注意的是Service能够将请求Port映射到人一个targetPort定义上。默认的targetPort将会被赋予和port值一样的值。更有趣儿的是targetPort可以是字符串，
 是后端Pods的port的名字。在每一个后端的pod上正真被赋值的port会有不同。这给部署和使用提供了很大的伸缩性。比如,你可以在Pods的下一次升级中更形pods所暴露
 的端口而不需要去修改客户端（前端的端口）。
 
 Kubernetes的Services可以支持TCP和UDP两种协议，默认的协议是TPC协议。
 
 # 没有选择器的Services
 
 通常Services是对访问Kubernetes Pods的抽象，他们也可以被用来抽象其他的后端。比如：
 * 你可能使用你自己的测试数据库，而在生产环境则使用外部数据库集群
 * 你可能要使用另一个namespace里面的Servcie或者另一个集群里面的Servcie
 * 你可能要迁移部分的工作任务到Kubernetes集群中，剩下的工作任务则继续保留在Kubernetes集群之外

如果满足上面任何一个场景你都能定义一个没有选择器的Service
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```
由于这是一个没有selector的service相关的Endpoints对象将不会被创建。但是你可以手工的将他映射到你定义的endpoints上去
```
kind: Endpoints
apiVersion: v1
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 1.2.3.4
    ports:
      - port: 9376
```

这里要注意：Endpoints的ip地址可能不会是回环地址（127.0.0.0/8），链路本地地址（169.254.0.0/16），链路本地多播地址（224.0.0.0/24）。

访问一个有选择器的Service和访问一个没有选择器的Service方法是一样的。访问会被路由到关联的endpoints对象上去。

一个定义有ExternalName service是一个没有选择器的service中的特例，他没有定义任何的端口和Endpoint，然而，它被用来访问集群外的外部Service的一种途径。
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

当寻找主机my-service.prod.svc.cluster时，集群的DNS服务将会返回CNAME记录中的my.database.example.com值。这种使用服务的方式和其他方法一样只不过它是在DNS
层上就进行了转发并没有代理和请求的转发。一会儿你会决定把你的数据库迁移到你的集群中吗？然后你就能启动它的Pods，定义合适的选择器或者endpoints然后更改
service的类型。

# 虚拟IPs和服务代理
每个Kubernetes的Node都会运行一个kube-proxy。kube-proxy负责为Service实现虚拟IP形式，当然ExternalName除外。在Kubernetes1.0中，Services是一个在IP层上的TCP&UDP 4层结构，代理完完全全的在用户命名空间里。到了Kubernetes1.1，Ingress API的引入则将其变为建立在HTTP之上的7层结构，iptables代理模式也被实现。而这一功能从Kubernetes1.2开始成为了默认的配置。直到Kubernets1.8-beta版本， IPVS代理被引入。 下面我们来看看这些Services的实现：
## proxy-mode： userspace
这种模式下，kube-proxy监控Kubernetes 主节点上Services对象和Endpoints对象的加入或者删除。每一个Service他都会在Node节点上随机的挑选一个端口并将其开放。任何到这个代理端口上的链接都将会被代理到Service的后端Pods上（也可以说是Endpoints上）。具体挑选哪一个Pod则要看Service的SessionAffinity定义。最后，加入iptables的控制规则来将流量导入到Service的ClusterIP（虚拟的IP地址）和port上后者转发这个流量到代理端口。默认情况下，对于后端的Pod选择是轮询的方式。

https://d33wubrfki0l68.cloudfront.net/b8e1022c2dd815d8dd36b1bc4f0cc3ad870a924f/1dd12/images/docs/services-userspace-overview.svg

## proxy-mode iptables
在这个模式下，kube-proxy监控Kubernetes主节点上Service和Endpoint对象的创建以及删除。为每一个Service kube-proxy都会去建立一个iptable流量规则来捕获到这个Service的ClusterIP和Port的访问，并将其重定向到支撑Service的后端Endpoints组。每一个Endpoints对象，kube-proxy也都会为其安装iptables流量控制规则来选择后端的Pod。默认情况下，都端的选择是随机的。

显然，iptables代理不需要在用户空间和内核空间之间切换，和用户空间代理模式相比他更加快更加稳定。然而它也不像用户空间代理那样，iptables代理在被选择的Pod不能处理请求时自动重试另一个Pod，所以它和readiness探针一起配合工作。
 
 ## proxy-mode: IPVS
 Notes: 此功能在v1.9时还只是beta版本
 
 在这个模式下，kube-proxy监控Kubernetes Services和Endpoints对象的变动，通过调用netlink套接字来创建响应的ipvs规则并且和Kubernetes Services和Endpoints周期性的同步这些ipvs规则，以此来保证ipvs状态与期望值保持一致。一但Service被访问，流量将会去重定向到后端Pods。
 
 和iptables模式相同，ipvs是以netfilter框架提供的hook功能为基础，不同的是他用哈希表为底层的数据结构运行在内核空间。这将意味着ipvs重定向流量更快，在同步代理规则上拥有更优异的性能优势。而且，ipvs为负载均衡算法提供更多的选项，比如：
 * rr: round-robin 轮询调度
 * lc: least connection 最小连接数调度
 * dh: destingation hashing 目标地址散列调度
 * sh: source hashing  源地址散列调度
 * sed: shortest expected delay 最短期望延迟调度
 * nq: never queue  永不排队调度
 
 注意： 在运行kube-proxy进程之前，ipvs模式默认节点机器系统是安装了IPVS内核模块的。如果在启动kube-proxy进程后Kubernetes将以ipvs代理模式进行工作，但kube-proxy检查后如果发现在节点机器上没有安装IPVS模块那么kube-proxy就会进入到iptable代理模式。

在以上任何一个代理模型中，对Service的IP+port访问的任何流量都会被代理到一个恰当的后端Endpoints，这些对于客户端都是透明的他们不需要知道Services或者Pods的具体信息。基于用户IP的上下文亲和性的设置可以通过设置service.spec.sessionAffinity属性，只要将他设置成客户端的IP地址即可默认它是'None'，也可以通过属性service.spec.sessionAffinityConfig.clientIP.timeoutSeconds设置用户上下文最大关联时间（默认时间是10800秒），它要和service.spec.sessionAffinity配合使用。
 
 
# Multi-Port Services

接下来我们了解一下绑定多端口的服务设置

实际工作中我们会遇到一个Servces需要暴露多个端口。下面这个例子展示了怎么在一个Servcie对象定义中绑定多个端口。当选择绑定多个端口时你的指定所有的端口名字，这样一来Endpoints就不会搞混淆了。比如：
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  - name: TCP
    protocol: TCP
    port: 443
    targetPort: 9377
```
 
 # 选择你自己的IP地址
 
 在创建Service时我们可以指定我们的集群IP地址，只要设置spec.clusterIP属性即可，假设你想覆盖一个已有的DNS记录，或者遗留的IT系统已经在使用了一个特定的IP地址而且修改这个IP地址会涉及复杂的后续更改。这样我们就可以选择一个合法的IP地址来作为自定义的clusterIP来使用，当然这个地址必须是service-cluster-ip-rang区间。如果所选的ip地址不合法apiserver会返回一个422 HTTP状态码来提示错误。
 
 ## 为什么不用轮询调度方式的DNS
 
 我们总是有这样的困扰为什么我们要选择虚拟IP地址来做这些实现而不是利用标准的轮询调度的DNS方式。这是因为：
 
 * 很长一段时间里DNS的库没有很好的遵守DNS的扩展用户授权协议(TTLs)并且对域名解析结果做了缓存
 * 许多的应用都对DNS域名解析只做一次解析然后缓存结果
 * 假设应用和DNS库对现状做了很好的解决，也会很难管理客户端的应用重复的去做域名解析工作。
 
 我们尝试着去把用户从这个困境中解脱出来。换句话说，如果有足够多的用户支持用这个方式，我们也可以选用这种替换方案

 
# 服务发现

kubernetes支持两种基本的服务发现机制 - 环境变量和DNS

## 环境变量
当一个Pod运行在物理节点上时，kubelet就会往活动的Service里面设置一系列的环境变量，它既支持Docker的links特性，也支持简单的{SVCNAME}_SERVICE_HOST和{SVCNAME}_SERVICE_PORT 变量.

比如，服务“redia-master"对外暴露了TCP端口6379，IP地址是10.0.0.11生成了如下的环境变量：
```
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDRSSS=10.0.0.11
```

这样意味着会有顺序问题-任何要被Pod访问的Service一定要先于Pod自生创建，不然这些环境变量将不会被得到。DNS则不会有这样的限制。
 
## DNS
相对于环境变量另一个选项是安装集群附加项-DNS服务器。DNS服务器监控Kubernetes APIServer，如果有新的Services加入就会为他们每一个创建一组DNS记录，如果DNS在整个集群内被激活，那将意味着所有的Pods都可以通过Services的自动名称解析功能被发现。

比如， 在Kubernetes的命名空间”my-ns”，有一个名字为“My-Service"的Service，那就会有一个名为my-service.my-ns的DNS记录被创建。在“my-ns”命名空间的Pods就可以个简单的通过Service “my-service”来找到这个Pods。在其他的命名空间中的Pods命名规则必须类似于“my-service.my-ns”。这样一来通过名称解析的结果就是找到相应的集群IP地址。

Kubernetes也支持对定义的ports通过DNS SRV来发现。如果“My-Service.my-ns" Service定义了port的名字为“http“，协议类型为TCP，就可以通过DNS SRV查找”_http._tcp.my-service.my-ns"来查找这个端口值。

Kubernetes的DNS 服务器是唯一一个能够访问ExternalName Service的方式。

## 无头部信息的服务
有时候我们可能不需要负载均衡或者一个单独的Servcie IP地址，这样一来你就可以创建一个“无头部信息”的服务，这个服务中没有集群IP可以通过设置spec.clusterIP属性为None来实现。

这样一来对于Dev就能用自己的方法去发现Endpoint从而降低与Kubernetes框架的耦合度，应用本身仍然可以使用自我注册的模式并将自己容易地集成到其他的服务发现系统中。

像这样的Services 没有集群IP地址，kube-proxy并不会去处理这些服务，对他们也没有复杂均衡也没有代理。DNS是如何自动的配置自己这依赖service是否定义了选择器。

## 定义了选择器的Service

对于定义有选择器的无头部信息的服务，Endpoints控制器会在APIServer上创建Endpoints记录，并且修改DNS配置信息返回一个Service的后端Pods记录。

## 没有定义选择器的Service

没有定义选择器的无头部信息的服务，Endpoints控制器则不会去创建Endpoints记录。但是DNS系统仍然可以这样被查找和配置：

* 对于ExternalName类型的服务通过CNAME记录
* 同名的Endpoints，其他被发现的类型也一样

# 发布服务 - 服务类型

一般对于像前端这类的应用我们都会把它暴露成一个外部IP地址。

Kubernetes的ServiceType属性值用来设置服务的暴露类型，默认是ClusterIP集群IP。

对应的Type类型值，以及他们行为：
* ClusterIP： 只在集群内部可见的IP形式暴露。 这样一来这个服务只能在集群内部可见，这是这个Type的默认值
* NodePort：以节点的IP在一个静态的端口上（NodePort定义）暴露服务。一个集群内部以ClusterIP的服务也会被创建，他会被路由到NodePort 服务。在集群外部我们可以<NodeIP>:<NodePort>的地址去访问这个NodePort服务。
* LoadBalancer：使用云供应商提供的负载均衡器对外暴露服务。NodePort和ClusterIP服务也会被创建同时他们会被路由到LoadBalancer的服务上去。
* ExternalName：将服务映射到externalName字段值（比如：foo.bar.example.com)，CNAME记录的值。 其中没有代理生成，这个功能需要kube-dns v1.7或更高的版本
  
## Type - NodePort

如果你设置了Type值为"NodePort", Kubernetes主节点会以标记好的端口区间配置分配一个端口（30000-32767），这样每一个节点都会代理这个端口。这个端口可以在 spec.ports[*].nodePort查看到

如果要自定一个端口值，可以在nodePort字段中指定一个值，这样Kubernetes系统会用这个你指定的端口值，当然我们要自己保证这个端口不被占用并且是在node port的区间内的，不然APIServer将会返回一个事务失败的状态码

这个方式给了Dev很大的自由度来设置他们自己的负载均衡器，这样就能配置出一个Kubernetes不支持的环境，或者就是以暴露更多的节点IP地址为目的的

这个服务的设置可以在<NodeIP>:spec.ports[*].nodePort和spce.clusterIP:spec.ports[*].port查看到。
  
## Type - LoadBalancer

如果云服务提供商可以个提供负载均衡功能，我们可以通过配置type字段来设置LoadBalancer来为你的Service获得一个负载均衡能力。实际的负载均衡配置是一个异步的，配置过程的状态信息发布在Services的这个字段中。比如：
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyAp
  Ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
  clusterIP: 10.0.171.239
  loadBalanceIP: 78.11.24.19
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 146.148.47.155
```

来自这个外部负载均衡器分发的流量就会被代理给后端的Pods，实际的工作逻辑依赖于云服务提供商的配置。一些提供商允许指定loadBalancerIP地址。这样一来，负载均衡器就会以用户指定的IP地址工作。否则一个临时的负载均衡IP地址就会分配给这个负载均衡器。如果指定了负载均衡IP地址但是云服务提供商不支持这个功能那么配置将会忽略。

Azure服务的注意事项：要使用用户自定义的loadBalancerIP值，得实现创建出这个公共的IP地址资源，它必须是和集群在同一个资源组中。分配这个指定的IP地址给loadBalancerIP。通过在配置文件的securityGroupName字段中检查配置非否生效。

## 内部的负载均衡器

一个混合的环境中有时候需要在同一个VPC内部路由流量。

在一个水平拆分的DNS环境中，你可能需要既能将你的Pod提供的service路由到外部也需要在内部路由。

这就要基于具体的云服务提供商在Service定义中加入以下的注释。

## 外部IPs

如果有外部IPs需要被路由到一个或多个集群节点，Kubernetes的服务可以被暴露给这些外部IPs。进入集群的流量中有externalIP会根据服务的端口，路由到其中一个服务端点。Kubernetes不会管理externalIPs，这些是集群管理者来负责的。

在ServiceSpec中，extrenalIPs可以和任何一个ServiceType一同使用。在以下的例子中，“my-service”可以被客户端在80.11.12.10：80上访问。
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: myApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  externalIPs:
  - 80.11.12.20
```

# 不足

使用用户空间代理virtual IPs，这个只适合于规模中等的容量，不适合于十分庞大拥有成千上万服务的集群。请浏览[Deisgn]

  







































