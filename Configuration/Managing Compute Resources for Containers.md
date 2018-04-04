# Managing Compute Resources for Containers
当你指定一个Pod的时候，你可以可选的指定每一个容器的所需要的CPU和内存。 如果指定了资源，调度器就能更好地判断把Pods放到那个节点上去。如果容器有资源限制，
则会有专门的方式来处理资源的争用。如果想了解更多关于资源请求和限制，参阅Resource QoS

## 资源类型
CPU和内存就是常见的资源类型。每一种类型的资源都有一个基础的衡量单位。 CPU是以核数来指定，内存是以字节数来指定。
CPU和内存都是被归类为计算资源，或者资源。计算资源是可以被用数量衡量的，可以被请求，分配，消费。他们和API资源不同，API资源比如Pods，Services这些只能用
Kubernetes APIServer来读取或者修改。

## Pod和Container的资源的请求与限制
Pods里的每一个的Container都能用以下一个或多个描述规范属性
* spec.containers[].resources.limits.cpu 
* spec.containers[].resources.limits.memory
* spec.containers[].resources.requests.cpu
* spec.containers[].resources.requests.memory

尽管资源请求和资源限制只能被指定到每个容器上，我们直接说Pod的资源请求和资源限制会更方便。 一个Pod上特定类型资源的请求和资源限制其实就是在Pod中的每一个
容器上这个资源的请求和限制的总和。

## CPU的方法
CPU计算资源的请求和限制是用CPU单元衡量的。 在Kubernetes中一个CPU等于
* AWS的1个vCPU 
* GCP的1个Core
* Azure的1个vCore
* 英特尔超线程处理器的1一个Hyperthread

碎片化的资源请求和限制也是允许的。Container的描述规范是spec.containers[].resources.requests.cpu是0.5，那就意味着最多只有1个cpu的一半。表达式0.1与表达式100m是一个意思，就是要分配100 millicpu。有人说是100 millicores都是一个意思。资源请求中的小数点像0.1这样的会被代码转化为100 millicpu，对于太
小的资源请求比如小于1 millicpu是不对允许的。基于此100 millicpu可能是一个比较合适的值。

CPU总是以一个绝对数量来被请求的，不会是相对数量。 所以，0.1的资源请求对于单核，双核，或者48核的物理机器来说都是一样的。

## 内存的请求方式
内存计算资源是以字节数来衡量的。 既可以指定成

## Pods的资源请求是怎么被调度的
当你创建一个Pod时，Kubernetes资源调度器就会选择一个node来运行这个Pod。每一个节点都有一种资源类型的最大容量： 一个节点能够提供的最大CPU数量和内存大小
调度器会保证在每个节点上被调度的Container中使用的CPU和内存的总和是小于节点能提供的容量的。尽管在节点上实际的内存和CPU计算资源的使用量会很少，调度器也会阻止在节点容量检查失败的情况下继续调度Container。这是为了保护节点上的物力资源被耗尽万一后来对计算资源的需求增长了。比如：遇到每天的请求峰值。

## Pods的资源限制是怎么实现的
当kubelet在节点上启动一个容器，它将CPU和内存资源限制值传递给容器运行时，当使用Docker时：
* 









