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


















