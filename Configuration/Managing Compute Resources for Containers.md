# Managing Compute Resources for Containers
当你指定一个Pod的时候，你可以可选的指定每一个容器的所需要的CPU和内存。 如果指定了资源，调度器就能更好地判断把Pods放到那个节点上去。如果容器有资源限制，
则会有专门的方式来处理资源的争用。如果想了解更多关于资源请求和限制，参阅Resource QoS

## 资源类型
CPU和内存就是常见的资源类型。每一种类型的资源都有一个基础的衡量单位。 CPU是以核数来指定，内存是以字节数来指定。
CPU和内存都是被归类为计算资源，或者资源。计算资源是可以被用数量衡量的，可以被请求，分配，消费。他们和API资源不同，API资源比如Pods，Services这些只能用
Kubernetes APIServer来读取或者修改。

## Pod和Container的资源的请求与限制
每一个Pods的Container能
