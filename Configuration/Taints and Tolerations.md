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











































