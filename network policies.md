# 网络策略
所谓的网络策略就是一组Pod怎们相互进行通信或怎么与其他的网络终端通信
NetworkPolicy 资源属性就是利用标签来选择Pods并且定义了怎样让流量在选定的Pod之间流通规则

# 前提条件
网络策略是网络插件实现的， 所以你必须使用网络解决方案来支持NetworkPolicy资源属性，没有网络策略控制器而只是简单的创建一个网络策略资源是没有任何作用的。

# 孤立的Pods和非孤立的Pods
默认所有的Pods都不是孤立的，他们都可以接受来自任何的流量。
Pods会在有NetworkPolicy资源属性定义时变得孤立。一但有这个属性选择了特定的Pods，那个被选中的Pod就不会接受不被NetworkPolicy允许的流量（其他没有被选中的Pod则不受影响，继续保持默认的行为）

# NetworkPolicy 资源属性
参考[NetworkPolicy](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#networkpolicy-v1-networking)获得更详细的信息。
这里有个NetworkPolicy的例子
```
apiVersion: networking.k8s.io/v1
kind NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
      cidr: 172.17.0.0/16
      except:
      - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
       port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```
如果没有选择相应的网络解决方案来支持网络策略，那么仅仅让APIServer去按照这个描述规范去创建相应的资源不会有任何的作用。

必填字段：和其他的Kubernets配置资源文件一样， NetworkPolicy资源文件需要apiVersion，kind和metadata字段。其他的基本信息请参考这些文档

spec：NetworkPolicy 规范定义了在特定命名空间内创建一个网络策略所需要的所有信息。 podSelector： NetworkPolicy包含一个podSelector他用来选择要应用网络策略的一组pods。比如：策略回去选择所有role是db的pods，如果指定他的podSelector为role=db。 默认会有一个空的podSelector他会选择所有的pods。

policyType : 每一个NetworkPolicy都会包含policyTypes列表里面指定或者只有Ingress，或者只有Egress，或者两个都有。policyType字段指定了是否有指定的进站策略被应用到选定的一组Pods上来，或者出站策略被应用到选定的一组Pods上来，亦或者既有进站策略也有出站策略。如果不指定policyType则进站策略会被设定，而出站策略只有存在出站策略的时候才会被启用。

ingress：每个NetworkPolicy可能会包括一个优良的进站列表。 每一个都要求既满足‘from'也要满足’ports'部分。这个例子只有一个单独的规则，他要求从三个中的任意一个源在单独的端口上满足流量就能入站。 第一个是ipBlock， 第二个是通过命名空间选择器， 第三个是同过Pod选择器。


# 默认的策略
默认情况下，在命名空间中如果没有策略存在，那么所有个的入站和出站操作都会被允许。以下的例子就会然你更改那个命名空间中的默认的行为

## 默认阻止所有的入站流量
你可以为一个命名空间创建一个默认的孤立策略，可以创建一个NetworkPolicy用它来对所有的Pods进行入站访问
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```
这可以保证所有没有被其他NetworkPolicy作用的Pods仍然处于孤立的状态。 这个策略不会改变默认的出站孤立行为

## 默认允许所有的入站流量
如果你想让入站流量去访问一个命名空间中的Pods（尽管已经有了策略被应用与一些Pods，这些Pod已经被孤立），你仍然可以创建一个策略显示的允许那个命名空间的所有流量
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metdata:
  name: allow-all
spec:
  podSelector: {}
  ingress:
  - {}
```

## 默认阻止所有的出站流量
你可以为一个命名空间创建一个默认的出站孤立策略，通过创建NetworkPolicy应用到所有的Pods并阻止从这些Pods上发出的出站流量。
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Egress
```
这会保证没有被其他NetworkPolicy选中的Pods不会有流量从他们发出。 这个策略不会更改默认的入站孤立流量

## 默认允许所有的出站流量
如果你想让一个命名空间中的所有源自于Pods的出站流量发出（尽管这些Pods上已经被应用了策略，已经是孤立Pods了），你仍然可以创建一个Policy来显示第允许那个命名空间上的所有出站流量
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```

## 默认阻止所有的入站和出站流量
你可以个为一个命名间创建一个NetworkPolicy，并用他来创建一个默认的策略来阻止所有的进站和出站流量。
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```







































