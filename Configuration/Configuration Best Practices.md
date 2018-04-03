## 配置最佳实践
这篇文章归纳强调了贯穿整个用户文档和例子的最佳实践。
这是个实时的文档，如果你认为有一些实践没有被列入这篇文章但是对于用户是非常有用的，请不要犹豫给我们开issue或者PR，我们会将他们加入到这篇最佳实践当中
来。

## 基本的配置注意事项
* 当你定义描述规范时，指定最新的稳定API版本
* 在将配置规范推送到集群之前，应当用版本控制软件管理他们。这样会让你快速的回滚到上一个可用的配置规范定义。这样做的目的也是为了重建集群或恢复集群
* 使用YAML格式来编辑你的配置规范文档而不是用JSON。尽管JSON可以在大多数的场景中作为交互媒介，但是YAML更加面向用户。
* 把相关的对象定义放到同一个文件当中。管理一个文件比管理一堆要方便。 
* kubectl可以接收文件夹路劲作为输入参数
* 不要指定没有用的默认值，简单的，最小化的配置信息最不容易出错
* 为对象的描述添加注释，这样有利于我们去优化去自省

## ‘不加保护’的Pods 与 ReplicaSets， Demployment 和Jobs
如果可以避免不要使用不加任何保护的Pods（是指Pods没有关联到ReplicaSet或者Deployment)， 如果节点发生故障，‘不加任何保护’的Pods将不会被重新创建。

Deployment在除了显示申明不要重建的策略下，因为它即会去创建ReplicaSet来保证在集群中始终有期望数量的Pods存在，也指定了一个替换Pods的策略，总是首选
的方式去直接创建Pods。Job 也是一个不错的选择。

## Services
* 在创建Service相关的后端工作单元之前，在这些工作单元需要访问这个Service之前创建Service.当Kubernetes启动容器时，它会提供环境变量指向所有的运行的
Services，比如：有个‘foo'的Service，所有的容器都会在他们的初始环境里面得到这些变量：
```
FOO_SERVICE_HOST=<the host the Service is running on>
FOO_SERVICE_PORT=<the port the Service is running on>
```
如果使用代码和Service通讯，不要使用环境变量；使用DNS name of the Service来替换。 Service的环境变量方式只是给很难改动源代码使用DNS方式的遗留软件使用的，这种方式不灵活。
* 不要为Pod指定hostPort除非他绝对的需要。 当你绑定一个Pod到hostPort，这会限制Pod被安排的地方的数量。 因为每一个<hostIP, hostPort, protocol>组合必须是唯一的，如果没有显示的指定hostIP和protocol，Kubernetes就会默认使用0.0.0.0为hostIP以及TCP作为默认的protocol。
如果只是为了debug方便，有可以使用apiserver proxy或者kubectl port-forward。
* 避免使用hostNetwork， 它也会降低Pod能够被安排的地方的数量。

## 使用Labels
定义并且使用标签来标识应用或者Deployment的语义，比如：{app: myapp, tier: fronted, phase:test, deployment: v3}. 你可以使用标签来为其他资源选择合适的Pods； 比如： 一个Service选择所有的含有标签tire：frontend的Pods

Service可以使用它的选择器来跨多个Deployment。 Deployment可以很方便的更新service而不需要停止它。
Deployment可以描述一个对象的期望状态，如果改变了的规范描述值被应用到了Deployment，部署控制器就会把实际值更改为这个期望的状态。

* 你可以为了debug目的来操作label。 因为Kubernetes控制器（比如：ReplicaSet)和Service是通过使用选择器选择标签来找到Pods，从Pod上删除相关的标签则意味着这个pod将不会被控制器选中或者不会参与一个服务的响应。如果你删除了一个已经存的Pod的标签， 它的控制器则会创建新的Pod来代替它。 这个功能用来在隔离区环境bug一个之前能很好工做的Pod。
















