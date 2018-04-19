## 什么是节点？
在Kubernetes中一个节点就是一个工作机器，之前被称之为minion。 他可能是一个物理机器可能是一个虚拟机器，这样看集群是什么样的。他是被主模块管理并且配置有
Pod所需要的服务，包括Docker，kubelet，kube-proxy。

## 节点状态
节点的状态包括一下的信息：
* 地址
* 阶段 -- 被废弃
* 状况
* 能力
* 基本信息

我们依次来看看这些属性

### 地址
这个属性很大程度上依赖于你所使用的云提供商或者裸机的配置
* HostName: 它是由节点内核提供。 可以通过kubelet的--hostname-override参数来覆盖
* ExternalIP: 这个节点的IP地址是可以被路由到外部网络的
* InternalIP: 只能够在集群内部被路由

### 阶段
被废弃，已不再使用

### 状况
这个属性描述了所有运行的节点的状态
OutOfDisk           True-> 如果没有足够的磁盘空间来调度新的Pod， 否则为False
 Ready              True-> 如果节点是健康的并能够可以调度Pods。False表示不能接受新的Pods。Unkonwn节点控制器在40秒内接收不到节点的信息
 MemoryPressure     True-> 如果节点内存有压力，如可用内存很少；否则为False
 DiskPressure       True-> 如果节点的可用盘空间有压力，如空间很少；否则为False
 NetworkUnavailable True-> 如果节点的网络没有被正确配置；否则是False
 ConfigOk           True-> 如果kubelet的配置是正确的；否则是False
 
 节点的状况是一JSON方式展示的， 比如一个健康的节点描述是这样的
 ```
 "conditions": [
 {
   "type": "Ready"
   "status": "True"
 }
 ]
 ```
 如果状况中的Ready条件是Unknown或者False超过了pod-eviction-timeout设置值，这是属性是传给kube-controller-manager的有节点控制器来调度节点上Pods被
 删除的工作。 默认的驱逐时间是5分钟。 有些情况下当节点是连不上时， apiserver是不能和节点上的kubelet通信的，知道和kubelet的通信恢复了才能决定删除节点上的Pods。 同时被调度安排删除的Pods可能会继续在分区节点上运行。
 
 在Kubernetes1.5之前， 节点控制器会在APIServer上强制删除这些不能访问的Pods。 然而到了1.5以及以后的版本， 节点控制器不会强制去删除这些不能访问的Pods直到得到确认他们已经在集群中停止运行了。 这就会看到那些不能被访问的节点上运行的Pods的状况是Terminating或者Unknown。在Kubernetes不能由于节点故障自动处理底层基础设施的情况下需要管理员来手工删除节点。 删除节点会导致在这个节点上的所有Pods都被从APIServer上面删除，释放他们的名称资源。
Kubernets v1.8引入了alpha功能会根据条件自动创建节点污点。要想使用这个行为，加入参数--feature-gate=..., TaintNodesByCondition=true 到APIServer，控制器管理者和调度器中去。 当TaintNodesByCondition被使用了，调度器会在调度时忽略节点的状态而只是关注节点的污点和Pod的容忍度。
现在用户可以从旧的调度模型和新的更灵活的调度模型中做选择。 Pod没有任何的容忍度就会按照旧的调度模型被调度。而Pod可以容忍节点上的污点，那么他就可以被调度到那个节点之上。

> 注意，由于在污点的创建和状态的获取之间存在非常短的延迟，通常是小于一秒钟，所以完全有可能会存在这样的现象成功调度的Pods的数量有小的增加但是又会被kubelet拒绝掉了。

### 容量
描述了节点上可用的资源，比如：CPU,内存和可调度的最大Pod数量。

### 基本信息
一般基本的节点信息，比如内核版本kubernetes版本(kubelet和kube-proxy)，Docker版本，操作系统版本。这些信息都是kubelet从节点上搜集的

## 管理
不像Pods和Services，节点不是Kubernetes自来创建的：他是云服务提供商或者在你的虚拟机池子里就存在的。换句话说就是当Kubernetes创建了一个节点，仅仅只是创建了一个代表这个物理节点的对象。创建完之后，Kubernetes就会去检查这个节点对象是否有效。 比如：用如下的配置创建一个节点对象：
```
{
"kind": "Node",
"apiVersion": "v1",
"metadata":{
  "name": "10.240.79.157",
  "label":{
    "name": "my-first-k8s-node"
  }
}
}
```
Kubernetes就回去创建一个内部的节点对象，并且基于metadata.name这个字段去验证这个节点的健康状况。如果这个节点对象是有效的，比如上面所有的服务都可用，他就有资格被用来调度Pod；否则他就会被忽略不会去参与集群的任何工作指导这个对象变得有效。Kubernetes会一直保留这个无效的节点对象直到客户端显示的去删除
它，在此期间Kubernetes会不停的重复健康检查动作。
目前，有三种模块会和Kubernetes节点接口交互：节点控制器，kubelet，kubectl

### 节点控制器
节点控制器是一个Kubernetes控制节点各方面操作的主要模块。
他在节点的生命周期里面扮演不同的角色。
第一件事情就是为节点分配一个ICDR段。
第二件事情是维护它自己内部的一个节点列表并持续更新他，这个列表是和云服务供应商能提供的计算机器相对应的。 一但运行一个云环境，不管什么时候只要有一个节点不工作，节点控制器就会询问供应商这个节点对应的计算机器是否可用。 不能用节点控制器就回去删除这个节点对象

第三件事情就是监控节点的健康状况。节点控制其负责更新NodesStatus的NodeReady属性，比如：节点控制器由于检测节点的心跳而认为此节点变得不能通信了就要把它置成ConditionUnknown，如果节点还是不能恢复通信就会驱逐所有在上面运行的Pods当然是使用优雅的方式。默认的ConditionUnknown状态持续的时间是40秒，5分钟之后开始驱逐上面的节点。
节点控制器会在--node-monitor-period参数设置的时间间隔去检查节点的状态的

在Kubernetes1.4中，我们更新了节点控制器在大规模节点遇到问题时访问主节点时的场景。 从Kuberenetes1.4开始，节点控制器在是否要驱逐节点上的Pods时会看集群中节点的状态。

在大多数情况下，节点控制器会限制驱逐节点的比率到--node-eviction-rate参数设置的值，默认是每逢中0.1就是说他不会在10秒钟内从多余一个节点上驱逐Pods。

节点驱逐行为在一个指定的区域内的节点变得不可用时会有变化。 节点控制器会检查这个区域内同一时间段节点不可用的比率（NodeReady属性为ConditionUnknown或ConditionFalse），如果不可用的节点比率至少有--unhealthy-zone-threshold（默认是0.55）指定的值时，驱逐Pod的比率会减少；如果集群很小（比如小于或等于--large-cluster-size-threshold属性指定的值时）默认是50，那么驱逐行为会停止。



































