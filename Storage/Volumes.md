# 存储卷
在容器磁盘的文件是短暂存在的， 这对于在容器中运行一些有意义的应用来说是会导致问题的。第一，当容器崩溃时，kubelet回去重启这个容器，但是里面的文件将会丢失
容器会重新以一个新的状态启动；第二，当在Pod中有一组容器在运行，这些容器间是有文件共享的。 Kubernetes 的存储卷抽象用来解决这些问题。

在这里获取Pod相关信息

## 背景
Docker也有存储卷的概念，只是Docker对他的管理稍显松散。在Docker里面存储卷就是一个在磁盘上的简单路径后者是在另一个容器中。 Docker对他不提供生命周期的管理
而且在此时之前只支持本地磁盘存储卷。 现在Docker开始支持存储卷驱动，但是这个驱动的功能有限。每一个容器只能有一个存储驱动并且不支持传入参数。

从另一方面讲，Kubernetes的存储卷则和使用它的Pod一样具有显示的生命周期。其结果就是存储卷的生存期会比Pod中的容器生存期更长并且卷里面存储的数据会在容器重启
过程中被保留下来。 当然，如果Pod停止并且退出，存储卷也会停止退出。更重要的是Kubernetes可以支持在一个Pod中使用不同类型的存储卷，且数量可以是同时使用任意
数量的存储卷。

究其本质，存储卷就是一个路径，亦或者包含有数据的路劲，这个路劲是可以被Pod中的容器访问到的。那么这个路径是怎么来的呢？ 他是由其介质负责提供，介质内容则由
这个存储卷的类型决定。

Pod使用存储卷时通过.spec.volumes字段指定，和.spec.containers.volumeMounts字段。

在容器中的进程看到的文件系统是一个由他们自己的Docker镜像和存储卷共同组成的文件系统。 Docker镜像是这个文件系统的根，所有其他挂载进来的存储卷则是这个镜像
的某一个特定的目录上。存储卷是不能挂载到另一个存储卷上的亦或者是另一个存储卷的硬链接。


## 存储卷类型
Kubernetes 支持如下的存储卷类型
* awsElasticBlockStore
  awsElasticBlockStore存储卷是将Amazon Web Services（AWS）EBS 存储卷挂载到Kubernetes 集群中的Pod上，不像emptyDir存储卷会在Pod不使用时清理它使用的
  存储卷，EBS存储卷会保存卷内容被切很少会卸载卷。这就意味着EBS存储卷可以预埋数据然后会这些数据移交给Pods让数据在Pods之间共享。

这种awsElasticBlockStore数据卷的使用会有一些限制：
* 支撑Pod运行的Node必须是AWS EC2实例
* 这些实例必须是和EBS存储卷在同一个地域的同一个可见区域
* EBS存储卷只支持一个EC2实例挂载一个存储卷

### AWS EBS配置例示
```
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  contianers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-ebs
      name: test-volume
  volumes:
  - name: test-volume
    awsElasticBlockStore:
      volumeId: <volume-id>
      fsType: ext4
```












