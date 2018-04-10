# 为节点指派Pods
你可以限制Pod只能被分配到特定的节点上或者更倾向于在某个节点上运行。 这里有几种实现的方法，这些方法都是用了标签选择器来做出选择。 一般来说没有必要做这
个限制，因为调度器会自动的做出合理的替换（比如：在节点之间散布Pods，而不是在一个资源不足的节点上调度Pod）。然后也会有这样的需求用户想对Pod的分布获得
更多的控制权，比如：确保Pod是在一个挂有固态硬盘的节点上终止生命周期，或者让来自两个一起工作的Pod能被分配在有相同可见性的区域。
这里有这些例子的[所有文件](https://github.com/kubernetes/website/tree/master/docs/user-guide/node-selection)

### 节点选择器
nodeSelector是最简单的一种限制方法。它是Pod详细描述规范的一部分，是包含若干名值键的图数据结构。 想要Pod被运行在一个特定的节点上，节点上必须有表示的
名值键对的标签。最常见的用法是一个名值键对儿。

让我们仔细看一个例子怎么使用nodeSelector的

#### 步骤0：前提条件
这个例子假设你已经具有了基本的Kubernetes知识。

#### 步骤1：把标签贴到节点上
执行kubectl get nodes获取所有可用的节点。选出你想要贴标签的节点然后执行kubectl label nodes <node-name> <label-key>=<label-value>将标签贴到节点
上比如想把标签‘disktyp=ssd’贴到节点‘kubernetes-foo-node-1.c.a-robinson.internal‘上去， 我们可以执行
kubectl label nodes kubernetes-foo-node-1.c.a-robinson.internal disktype=sd

如果遇到‘invalid command’错误信息，你可能是在使用老版本的kubectl 它不支持label命令。

之后可以使用kubectl get nodes --show-label 命令来检查节点上是否有这个标签。

#### 步骤2：在Pod的详细描述规范中加入nodeSelector属性
不管你手头有怎么样的Pod的配置文件，尽管加入nodeSelector属性。加入你有如下的配置文件
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
  spec:
    containers:
    - name: nginx
      image: nginx
```
接下来添加nodeSelector
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```
当使用这个yaml文件去创建Pod时，调度器会把这个Pod调度到我们添加了标签的节点上。 我们可以通过kubectl get pods -o wide 命令来查看Pod所分配的那个节点
信息。

### 过渡选项： 内建节点标签
除了我们自己贴标签，节点还自带了一套标准的标签。 从Kubernetes1.4开始有：
* kubernetes.io/hostname
* failure-domain.beta.kubernetes.io/zone
* failure-domain.beta.kubernetes.io/region
* beta.kubernetes.io/instance-type
* beta.kubernetes.io/os
* beta.kubernetes.io/arch

> 注意：这些标签的值是依赖于云服务提供商，并不能保证他们的可用性。比如：kubernetes.io/hostname属性值可能在一些环境中就是节点的名字而在另外的环
境就是其他的值了。

### 亲和性与反亲和性
nodeSelector通过特别的标签将Pod限制在节点上。 目前处于beta阶段的亲和性与反亲和性功能极大的扩展了限制的类型。 关键的改进是：
1， 语法更加灵活，不仅仅只是‘AND‘的绝对符合
2， 你只需要表明一下规则是‘软性的’，‘偏爱的’而不是硬性的需要，所以如果调度器不能够被满足，Pod是能被调度到某个节点上。
3， 可以针对在节点上运行的其他的Pods的标签，而不是针对节点的标签，这就变成了可以和那个Pods或者不可以和那个Pods共存的规则。

亲和性功能由两类亲和性组成， ‘节点亲和性’和‘Pod间的亲和性/反亲和性’。节点的亲和性就好像一个存在的nodeSelector，而Pod间的亲和性和反亲和性是通过Pod
标签而不是节点标签进行限制。nodeSelector将继续和以往一样工作但最终会被废弃，因为节点的亲和性可以表达nodeSelector的意思。

#### 节点亲和性
节点亲和性在Kubernetes v1.2以alpha功能的方式被引入。他在概念上和nodeSelector类似， 它允许你以节点标签为基准限制Pod应该被调度到哪个有资格的节点上。
目前有两种节点亲和性，叫做requiredDuringSchedulingIgnoredDuringExecution和preferredDuringSchedulingIgnoredDuringExecution. 可以把他们认为分
别是软性限制和硬性限制，就好像之前的规则里规定的Pod在被调度到节点之前必须符合这以规则（就像nodeSelector，但是他的表达式更为丰富），后一个则是偏好某
个规则它会尝试着完全满足但是不会保证一定会。 ‘IgnoredDuringExecution’正如它的部分名字表达的，可nodeSelector的工作方式一样。 如果节点的标签在
运行时发生了变化亲和规则不能在满足，Pod还是会继续在这个节点上运行。 节点亲和性是通过affinity下的nodeAffinity属性来指定的
```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            value:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  container:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```
这个节点亲和性规则描述了Pod只能被调度到有键名是kubernetes.io/e2e-az-name并且键值或者是e2e-az1或者是e2e-az2的标签的节点上。 另外， 在这些满足的
节点中，则更偏好有键名是another-node-label-key键值是another-node-label-value的节点。
这里使用到了运算符In。 新的节点语法支持这些运算符： In, NotIn，Exists，DoesNotExist，Gt， Lt。并没有显示的节点反亲和性，但是运算符“NotIn和
DoesNotExist可以做到这个行为。

如果你即指定了nodeSelector也指定了nodeAffinity，则在所有候选的节点上只有这两点都满足了才能在上面调度Pod。
如果在nodeSelectorTerms上指定了多个matchExpressions，则调度器只有在满足所有的表达式的节点上才能调度Pod

如果你删除或者更改了调度Pod所在的节点的标签，Pod不会被删除。 换句话说亲和性选择还将继续工作。

### Pod间的亲和性和反亲和性（beta特性）
>注意：Pod间的亲和性以及反亲和性需要大量的处理器，而这又会显著拖慢大集群的调度时间。 我们不建议在大于几百的节点集群中使用这个特性。

Pod间的亲和性是在PodSpec的affinity属性的podAffinity属性上指定。 Pod间的反亲和性是在PodSpec的affinity属性的podAntiAffinity属性中指定。

来看个例子：
```
apiVerison: v1
kind: Pod
metadata:
  name: with-pdo-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          mantchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDUringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
```
Pod上的这个亲和性描述规范定义了一个Pod亲和性规则以及一个Pod反秦和亲规则。 podAfffinity是requiredDuringSchedulingIgnoredDuringExecution 而podAnti
Afffinity则定义了preferredDuringScheduleingIgnoredDuringExecution。pod的亲和性规则定义Pod能够在这样的节点上调度，这个节点已经存在有键名security
键值S1的Pod并且和这个节点在一个区域。

原则上，topologyKey可以是任何合法的标签键。 然而处于性能和安全的原因， 对topologyKey有这样几个限制：
1， 对亲和性和反亲和性的requiredDuringSchedulingIgnoredDuringExecution，topologyKey不能为空。
2， 对于Pod的requiredDuringSchedulingIgnoredDuringExecution属性来说， 准入控制器LmitPodHardAntiAffinityTopology将限制topologyKey为kubernetes.
io/hostname. 如果想让它对用户拓扑可见，我们的修改准入控制器亦或者简单的不使用此准入控制器。
3， 对于Pod的反亲和性属性preferredDuringSchedulingIgnoredDuringExecution，不为topologyKey设置值那么它会被设置成‘所有的拓扑’（所有的拓扑是指：
kubernetes.io/hostname, failure-domain.beta.kubernetes.io/zone 和 failure-domain.beta.kubernetes.io/region)

除了labelSelecotr和topologyKey，你还可以有选择性的指定一个labelSelector应该比较且满足的命名空间列表。如果没有设置这个属性则默认和定义了亲和性和反亲
和性的这个Pod的命名空间一致。如果指定了这属性但是没有设置值Kubernetes会认为是所有的命名空间。

对于亲和性和反亲和性的属性requiredDuringSchedulingIgnoredDuringExecution设置的所有matchExpressions在都被满足的情况之下才会在这个节点上执行调度。

### 更多的实用性用例
在更高级别的集合中Pod间的亲和性和反亲和性将会更加的有用比如 ReplicaSets，Statefulsets，Deployments等等。他们能和容易的将一套工作单元协同定位到相同
的拓扑中去，比如相同的节点上。

#### 总是协同定位到相同的节点上
在一个三节点的集群中， 一个web应用程序使用了内存缓存数据库redis，我们要求web服务器要和这个缓存尽可能的协同定位。 这是一个简单的redis部署描述文件它包含
三个副本和标签选择器app=store。 Deployment包含一个PodAntiAffinity配置来保证所有个的副本不会被调度到同一个节点上。
```
apiversion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```
以下的web服务器的部署yaml片段包含podAntiAffinity和podAffinity配置。 这个告诉调度器所有它的副本应该和有app=store标签的Pod协同定位。 这也保证了每一个
web服务器的副本不会被协同定位到同一个节点上面。
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matcheExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
       containers:
       - name: web-app
         image: nginx:1.12-alpine
```
创建完上面的两个Deployment后，在我们的三个节点上将会出现这样的分布：
    node-1            node-2          node-3
  webserver-1       webserver-2     webserver-3
    cache-1           cache-2         cache-3

正如你所看到的，web服务器的三个副本自动的和缓存副本如我们期望的那样自动的协同定位了

最佳实践是对这样的有状态的工作单元如redis用反亲和性的规则实现高可用性的部署来保证分布。

#### 从不协同分布在同一个节点上
高可用性的数据库StatefulSet是包含一个主节点三个副本，一种部署偏好是不能把他们协同定位到同一个节点上。
node-1          node-2            node-3             node-4
DB-MASTER       DB-REPLICA-1      DB-REPLICA-2       DB-REPLICA-3




















