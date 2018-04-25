# Concepts Underlying the Cloud Controller Manager
## Cloud Controller Manager
起初云服务控制器管理器(CCM，与二进制安装模块不同)是设计被用来集成云服务特定代码但又和Kubernetes核心代码彼此之间独立。云服务控制器管理器和其他主要的模块
一起运行，比如：Kubernetes控制器管理器，APIServer，调度器。它也可以以Kubernetes插件方式运行于Kubernetes之上。

云服务控制器管理器是以插件方式设计的这样一来新的云服务提供商就能提供一个Kubernetes插件从而运行在Kubernetes里。 
这篇文档将讨论云服务控制器管理器背后的概念以及它的详细功能描述。我们先看看没有云服务管理器控制器的架构图：

## 设计
在以前的图例中，Kubernetes和云服务供应商是通过各种模块集成的：
* kubelet
* Kubernetes控制器管理器
* Kubernetes API Server

CCM着眼大局在之前的三个模块的基础之上创建了一个唯一的与云服务的集成点。 更新后的CCM是这样的

## CCM的模块
CCM将Kubernetes控制器管理器的功能分解开来，让他们在分离的进程中运行。 特别的是，他分解了Kubernetes控制器管理中和云服务有依赖的功能，如:
* 节点管理器
* 卷管理器
* 路由管理器
* 服务管理器

在1.9版本中， CCM之运行了以下的控制器：
* 节点管理器
* 路由管理器
* 服务管理器

另外，他还会去运行一个称为持久卷标签管理器。这个管理器负责给由GCP和AWS云服务供应商创建的持久卷设置地域和区域标签。
>注意 卷控制器是故意不被纳入CCM的。 由于复杂度和抽取供应商卷逻辑的工作量而不被CCM包含。  

最初使用CCM来支持卷是利用Flex卷以可插拔的方式支持。然而，考虑到投入后来用CSI替换了Flex。
正由于这些变数，我们决定用一个短期不支持的方式作为一个过渡方法指导CSI可以发布为止。

## CCM的功能
CCM继承了Kubernetes中和云服务供应商相关模块的功能。 这部分我们就来看看这个模块的结构。

1， Kuberenetes控制器管理器
CCM的功能主要来自于KCM，之前我们提到过，CCM运行下面的控制器：
* 节点控制器
* 路由控制器
* 服务控制器
* PersistentVolumeLabels控制器

### 节点控制器
节点控制器
节点控制器负责在云服务提供商提供里运行的集群中获得节点信息来初始化节点对象：
1， 用云服务提供商特定的地区/地域标签初始化节点
2， 用于服务商特定的实例信息来初始化节点对象，比如：类型、大小
3， 获取节点所在的网络地址和主机名称
4， 如果节点对象没有回应，检查云服务提供商提供的物理节点，如果物理节点不可用则删除此节点对象。

### 路由控制器
路由控制器负责在云环境中配置适合的路由规则以保证不同节点之间能够相互通信。 路由控制器只是用于谷歌的GCE集群。

### 服务控制器
服务控制器负责监控服务的创建、更新、删除事件。 在现有的Kubernetes中的服务状态基础之上，它配置云负载均衡器来反映Kubernetes中的服务的状态。 另外，它还保证服务的后端状态是实时更新的。

### PersistentVolumeLabels 控制器
当AWS EBS/GCE PD卷被创建出来后， 他负责为AWS EBS/GCE PD卷上添加标签。他把这个手工的工作给自动化了。
这些标记对于调度器来说很重要因为这些卷会被限制在他们所在的地区/区域里面运行。 使用这些卷的Pod则被调度到相同的地区/区域。
PersistentVolumeLabels控制器是特别为CCM实现的； 就是说在没有CCM之前是没有PersistentVolumeLabels控制器的。 在将Kubernetes API Server里面的PV打标签的逻辑移到CCM时，创建了这个控制器。 在KCM中没有这个概念。

2， Kubelet
节点控制器中包含有kubelet的与云服务提供商有依赖的功能。 在介绍CCM之前， kubelet负责初始化节点中与云服务提供商相关的详细信息包括IP地址，地区/地域标签及实例类型等的信息。 CCM的介绍中已经把这个初始化操作从kubelet中移动到了CCM中。
在这个新的模型中，kubelet初始化的节点是没有云服务提供商的特定信息的。 然后， 他会添加污点到这些新的节点上以使他们不能被立即调度Pod知道CCM添加了与云服务提供商相关的信息到这些节点对象里面。 然后kubelet回去删除这个污点。

3， Kubernetes API Server
PersistentVolumeLabel控制器把依赖云服务提供商的代码从Kubernetes API Server中移动到了CCM中。

## 插件机制
云服务控制器管理器使用GO语言预留了接口，这样特定的供应商就能提供自己的实现从而与Kubernetes集成。特别提一下，它使用了这些云服务供应商接口。


## 授权
这一节主要是把访问CCM各个对象并完成特定操作所需要的权限。

### 节点控制器
节点控制器只和节点对象打交道。 他需要所有的权限来get，list，create，update，patch，watch，delete节点对象
v1/Node:
* Get
* List
* Create
* Update
* Patch
* Watch 
* Delete

### 路由控制器
路由控制器监听节点对象的创建然后配置合适的路由规则。 他需要对节点对象有get 操作

### 服务控制器
服务控制器监听Service对象的创建、更新、删除事件，然后给这些Service对象绑定后端端点。 为了能够访问Services对象，他需要list和watch访问权限。 为了能够更新Service对象，他需要patch和update访问操作。 为了能够给Service绑定后端实现，他需要create、list、get、watch、update访问操作
v1/Service
* List
* Get
* Watch
* Patch
* Update

### PersistentVolumeLabels 控制器
PersistentVolumeLabels控制器监听PersistentVolume的创建事件然后更新这些卷。 这个控制器需要get和update访问权限
v1/PersistentVolume
* Get
* List
* Watch
* Update

### 其他
CCM的核心实现需要访问权限来创建事件， 









































