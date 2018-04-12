# 污点和容忍
节点亲和性是描述了Pod能够吸引一组节点能力的属性（或者是倾向或者是硬性的需要）。污点则恰好相反它是让节点排斥一组Pods

污点和容忍组合工作来使Pods不被在不合适的节点上调度。 一个节点上可以有多个污点；这表明这个节点不会接受Pods如果Pods详细描述规范中不接受这些污点。容忍适用
于Pods，它规定了Pods如果容忍了污点是可以被调度到有这样污点的节点之上的。

## 概念
通过kubectl 命令来给node加入污点（taints）
```
kubectl taint nodes node1 key=value:NoSchedule
```
它会去往节点node1上放置污点, 污点的定义是包含有键名key，键值value的名值对儿它将会影响到NoSchedule属性。 这意味着除非Pod有容忍污点的定义不然Pod不会被
调度到这个节点node1上去。
用下面这个命令来删除上面命令添加的污点。
```
kubectl taint nodes node1 key:NoSchedule-
```
通过在Pod的PodSpec中添加容忍，以下的容忍‘满足’在上面创建的污点，因此包含有任何容忍的Pod都会被调度到node1上面去：
```
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

```
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```
对污点的容忍包括键值相同和影响相同：
* Exists操作符(在没有value值设定的情况下)
* Equal操作符则需要和指定的value值一致才能生效
Equal是默认的运算符，如果没有指定运算符
> 注意：这里有两个特别的用例
* 空键名搭配Exists操作符将会容忍所有个的键名，键值和影响，换句话说就是容忍一切
```
tolerations:
- operator: "Exists"
```
* 空effects值将会对所有满足键名key的污点做容忍处理
```
tolerations:
- key: "key"
  operator: "Exists"
```
上面的例子是对effect的NoSchedule做了演示，可选的还有PreferNoSchedule, 这是NoSchedule的偏向或‘软’要求版本 - 就是说Kubernetes将会尝试避免调度Pod到
不容忍污点的节点上去，这不是强制的要求。 最后一种可选是effect的NoExecute，一会儿我们在讨论。

在一个节点上可以配置多个污点，同时在一个Pod上配置多个容忍。Kubernetes会像过滤器一样按照先处理所有节点上的污点，忽略Pod对他可以容忍的；剩下的没有被忽略的污点意味着影响着Pod的调度。一般情况是：
* 如果至少有一个没有被忽略的污点节点，Pod的要求是NoSchedule，那么Kuebernetes不会再这个节点上调度Pod
* 如果没有可以被忽略的污点影响到NoSchedule 但是至少有一个可以被忽略的污点影响到PreferNoSchedule，那么Kubernetes将会试着不去在这个节点上调度Pod
* 如果至少有一个没有被忽略的污点节点它影响到NoExecute那么即使已经有了Pod在这个节点上运行了他会被驱逐出去，如果没有运行的Pod也就不会再在上面调度新的
Pod。 
比如，你在节点上配置了这样的污点
```
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```
Pod上配置了这样的容忍规则
```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```
这个例子，Pod将不会被调度到这个节点node1上去，因为有第三个污点没有被容忍。 但是在污点被加入前已经运行的Pod将不会受到影响，他们会继续运行，因为只有第三个污点没有被容忍，这个污点值影响调度不影响已经运行的。

通常，有NoExecute污点被加入到节点上去，没有容忍这个污点的Pod将立即会被驱逐掉， 容忍的Pod永远不会被驱逐出去。然而NoExecute影响有一个可选的属性tolerationSeconds它会决定在污点被加入后能容忍的Pod还能再节点上停留多久。比如：
```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```
他的意思是在节点上有运行的Pod在这个节点上配置污点并且这个污点能被Pod容忍，那么这个Pod会在3600秒后被驱逐出这个节点，如果这个污点在3600之内被删除了这个Pod还会继续运行在这个节点上。

## 例子
污点和容忍是一个灵活的方式来处理要使Pod避开调度在一些节点上和组织Pod继续在节点上继续运行的需求。这里列出一些用例：
* 专注某个节点： 如果你想给一组用户专门分配一组节点给他们使用，然后给他们的Pod添加容忍（这个也可以通过创建一个用户自定义的准入控制器实现），这样一来这些Pod就能使用这些被污染的节点以及其他没有被污染的节点。如果你只想把这些节点配置成这些用户的专属节点，你还的额外加入相似的标签到这些节点上(e.g. dedicated=groupName），并且准入控制器相应的加入节点亲和性配置保证Pod只能被调度到有标签dedicated=groupName的节点。

* 特殊配置的节点： 如果在集群中有一小部分有特殊硬件配置的节点（e.g.  GPU）,我们期望没有特殊硬件需求的Pod就不要在这些特殊硬件配置的节点上被调度，可以省下资源给后来的有需求的Pods。这可以通过污染这些特殊硬件配置的节点。
```
kubectl taint nodes nodename special=true:noSchedule  或者
kubectl taint nodes nodename special=true:PreferNoSchedule
```
要让Pod被调度到这些节点上时只要给Pod的描述规范里面加入容忍就能让Pods使用这些硬件资源了。还是这个需求，其实最容的做法是加入一个用户定义的准入控制器。比如：推荐使用扩展资源来表示这些特殊资源，用扩展资源名字污染这些特殊配置的节点， 然后运行ExtendedResourceToleration准入控制器。 现在，由于节点被污染了凡是不能容忍这写污染的Pod都不会被调度到这些节点上。 但是当你发送一个Pod去请求扩展资源，这样ExtendedResourceToleration准入控制器就会自动把容忍加到Pod里面，这些Pod就会被调度到有特殊配置的节点上。这样一来我们就能保证这些特殊资源节点是专门为有这样请求的Pod独享的，而且不用你手工添加容忍到你的Pod配置规范里面去。

* 基于污点的驱逐机制（alpha featrue）： Pod的预定义驱逐机制，一但节点出现问题这个预定义机制就能立即被触发。

### 基于污点的驱逐机制
之前我们提到过NoExecute污点影响，这个污点会对已经运行的Pod按照如下的逻辑进行：
* 不能容忍污点的Pod立即被驱逐出节点
* Pod中的容忍描述规范中指定了容忍污点但是没有指定容忍时间tolerationSeconds，那么Pods会一直存在下去
* Pod中的容忍描述规范中指定了容忍污点也指定了容忍时间，那么着Pods会在时间到期后被驱逐出节点。

另外，Kubernete1.6中引入了alpha功能节点问题代理。换句话说就是，节点控制器会在定义的条件符合情况下自动加污点给节点，







































