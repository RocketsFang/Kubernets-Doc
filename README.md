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



 
 
 
 
 
 
 
 
 
 
 
 
 
 
