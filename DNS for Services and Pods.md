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
### A记录
‘普通‘（非无头部信息）服务被加入到DNS后，一条以这样的格式命名的记录会被创建在DNS服务器里 'my-svc.my-namespace.svc.cluster.local. 这样就能得到服务在集群内部的IP地址。
’无头部信息‘(没有集群IP地址）的服务也同样会以这种命名格式在DNS服务器里添加一条记录，不同于普通的服务，他会得到由服务选择出来的一组Pod的IP地址，客户端要么使用标准的轮询调度方式从集合中选择需要的Pod要么就选取所有的Pods。

### SRV 记录
SRV记录是为普通或者无头部信息的服务一部分的被命名的端口而创建的。每一个命名的端口，SRV记录会有这样的命名格式‘_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster.local'. 对于一个普通的服务来说这样就能解析出端口值和CNAMe: my-svc.my-namespace.svc.cluster.local. 对于一个无头部信息的服务来说，这会导致多个结果，每一个都对应一个Pod所对应的服务，每个结果里面都会有一个端口值和Pod的CNAME:auto-generated-name.my-svc.my-namespace.svc.cluster.local.

## Pods
### A记录
如果允许，pods在创建时会在DNS服务器里面创建一条记录，命名格式为‘pod-ip-address.my-namespace.pod.cluster.local‘。举个例子，有个IP地址是1.2.3.4的Pod对象在’default'的命名空间里，DNS的域名为‘cluster.local‘这个Pod的DNS记录名称就为’1-2-3-4.default.pod.cluster.local.

### Pod的主机名称和次级域名字段
目前当一个Pod被创建时，它的主机名称在Pod的字段’metadata.name‘里。
在Pod的描述规范里面有个一个可选的‘hostname’字段，这个字段也是被用来指定Pod的hostname。一但指定了，他会以高优先级来作为Pod的hostname。比如:假设一个Pod的hostname是‘my-host’，Pod会把它的hostname设置为'my-host‘。

Pod的描述规范还有个一个可选的‘subdomain’字段，它被用来指定Pod的次级域名。比如：一个Pod的hostname是‘foo'次级域名是’bar'命名空间是‘my-namespace’，它的完整合规的域名‘FQDN’则是‘foo.bar.my-namespace.svc.cluster.local'。
举个栗子：
```
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  labels:
    name: busybox
spec:
  hostname: busybox-2
  subdomain: default-subdomain
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox
```
如果在Pod相同的命名空间和相同的次级域名下有一个无头部信息的服务，集群中的kubeDNS服务器会返回一个Pod的完整合规的主机名。比如：有这样一个Pod，主机名是‘busybox-1’，次级域名是‘default-subdomain’，在相同的命名空间下有个名为‘default-subdomain’的无头部信息的服务，Pod的完整合规域名（FQDN）是‘busybox-1.default-subdomain.my-namespace.svc.cluster.local'， DNS服务器会用这个名字提供A记录来指向Pod的IP地址。这两个Pods‘busybox1‘和’busybox2‘则会有他们自己的A记录。
Endpoint对象可以和他的IP地址一样指定’hostname’指向任一端点地址

### Pod的DNS策略
DNS策略可以以每一个Pod为基础设置。目前Kubernetes支持一下的Pod特有的DNS策略。这些策略被指定在Pod描述规范的dnsPolicy字段。

* Default: Pod会继承所在Node的解析策略
* ClusterFirst：任何不满足集群域名后缀的查询，比如‘www.kubernetes.io’， 提交到了继承自Node的上一级命名服务器。集群管理员可以有额外的域名存根和上级服务器的配置。
* ClusterFirstWithHostNet： 如果你的Pod是以hostNetwork运行，你的显示设置他的DNS策略’ClusterFirstWithHostNet‘
* None： 在kubernetes v1.9中引入的新策略。它允许Pod忽略在Kubernetes环境里面的DNS配置。从而所有个DNS配置都是从dnsConfig字段中获取

>注意： ‘Default’并不是默认的DNS策略，如果dnsPolicy没有被显示的设置，那么‘ClusterFirst’策略就会被启用。

来看看这个栗子，这个Pod的DNS解析策略被设置为`ClusterFirstWithHostNet` 因为它的hostNetWork字段被设置为`true`
```
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNetPresent
    name: busybox
  restartPolicy: Always
  hostNetwork: true
  dnsPolicy: ClusterFirstWIthHostNet
```
### Pod DNS 配置
Kubernetes v1.9引入了一个Alpha功能此功能可以让用户在Pod的DNS的配置上具有更多的控制权。 在v1.10这个功能默认会被使用。 在v1.9中如果要使用这个功能集群管理员需要设置`CustomPodDNS`apiServer和kubelet启动参数来使用这个功能，这样设置`--feature-gates=CustomPodDNS=true,...` 一但这个功能被使用，用户就能设置Pod的属性 `dnsPolicy`为`None`然后添加新的属性`dnsConfig`到Pod的描述规范里。
`dnsConfig`字段是一个可选的字段它可以和任意的`dnsPolicy`配置一起工作。然而，一但`dnsPlolicy`字段被设置为`None`, 那么`dnsConfig`字段必须要设置。

以下的属性可以个被用到`dnsConfig`字段：
* `nameservers`： Pod可以使用的DNS服务器地址。最多设置3个IP地址。 当Pod的`dnsPolicy`属性被设置为`None`，这个列表必须包含至少一个IP地址，其他情况下这个属性值可以为空。这些列出的DNS服务器会和指定的DNS策略生成的基本命名服务相集成去除重复的记录地址
* `searches`： 在Pod中查找hostname的可选DNS域名。 这是一个可选的属性。如果设置了值，这些值会和DNS策略生成的基础域名集成，去除重复的域名。Kubernetes只允许设置6个可被查找的域名。
* `options`：这是个可选的对象列表，对于那些需要名值对的Pod来填写。这些值会与DNS解析策略生成的名值对相集成然后去除重复的项目。
举个栗子
```
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
    - image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster.local
      - my.dns.search.fuffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```
Pod被创建后，test 容器会把如下的内容加入到/etc/resolv.conf文件中：
```
nameserver 1.2.3.3
search ns1.svc.cluster.local  my.dns.serch.suffix
option ndots:2 edns0
```















