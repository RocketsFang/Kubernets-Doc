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
* 这个属性的值spec.contaires[].resources.requests.cpu 会被转换成相应系统的core值，这个属性值有可能是碎片化的，并且是1024的倍数。这个值比较大时或者是2会被用到docker run命令的--cpu-shares标志值
* 这个属性值 spec.containers[].resources.limits.cpu会被转换成它的millicore值并且是100的倍数。结果值为一个container在100毫秒的间隔中能使用的CPU总时间。Container在这个间隔中不能多于这个CPU的共享时间。
> 默认的CPU配额是100毫秒， CPU最小的配额是1毫秒
* 这个属性spec.containers[].resources.limits.memory 被转换成整数，将会被用在docker run命令的--memory参数。
如果container使用的内存超过了它的限制，这个容器会被停止。如果是能重启的，kublet将会和其他运行时失败一样会重启容器。
如果容器请求内存超过了它的内存限制，不管什么时候一但节点内存耗尽这个容器的Pod将会被不会再被kubelet管理。
Container可能或不可能被允许超过CPU的限制，但是发生了超额使用CPU容器不会被停止。

怎么确定Container由于资源限制不能被调度或者被停止，参看Troubleshooting章节

## 监控计算资源的使用
Pod的资源使用是Pod状态报告的一部分。
如果在集群里面部署了可选的监控程序，Pod的资源使用则可以从监控系统中获得。

## 问题诊断
### 我的Pod被挂起事件信息显示 failedScheduling
如果Kubenetes调度器发现没有任何节点上可以再增加Pod时， Pod就不会再被调度直到节点上有可用的资源。 调度器每次会对失败的调度记录一条错误的信息。
```
kubectl describe pod frontend | grep -A 3 Events
events:
 FirstSeen   LastSeen  Count   From           Suobject                 PathReason               Message
 36s            5s       6      scheduler         FailedScheduling     Failed for reason PodExceedsFreeCPU and possiblely others
```
在之前的例子里面，名为‘frontend’的Pod由于在节点上CPU资源不足而被调度失败。如果是在节点上内存不足也会有类似的错误信息。一般来说，如果一个Pod被挂起并报出类似的错误信息，可以试试以下的方法：
* 添加更多的节点
* 停止没有被使用的Pod，把资源释放给挂起的Pod
* 检查Pod的资源描述规范中的资源请求没有大过节点的限制。比如所有的节点都有cpu:1的容量，而Pod则被描述为可以请求cpu：1.1 这样一来这个Pod将不会被调度。

可以通过kubectl describe nodes命令来查看
```
kubectl describe nodes e2e-test-minion-group-4lw4
```
可以配置资源配额功能来限制可以被消费的资源总数。 如果配合命名空间，则能防止所有的资源被一个team占尽。

### 我的Container被终止了
你的Container可能会由于资源缺乏而被终止。可以通过kubectl describe pod来检查特定的Pod中的Container是否是由于资源限制而被终止的。
























