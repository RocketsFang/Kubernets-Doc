# Jobs - Run to Completion
一个工作单元会创建一个或多个pods并且保证特定数量的他们能够被正确终止。 随着pods的成功执行完成，工作单元job追踪工作内容成功的完成。一旦当特定数量的
成功完成，job本身也就随之终结，同时删除Job将意味着由job创建出来的pod也会被清理掉

一个简单的例子为了正确地完成一个pod我们创建一个job对象。 job对象将会去创建一个新的pod如果第一个pod执行失败或者被删除了（比如：由于一个节点硬件的
故障或者节点被重启）

job同时也能被用来同时运行多个pod

## Running an exmple Job
这是个job的配置，它会计算圆周率派的大小到2000位然后打印结果。它会用将近10秒来完成这个工作
```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```
执行以下命令来创建这个job例子
```
kubectl create -f job.yaml
```
检查这个job的状态，运行以下命令
```
kubectl describe jobs/pi
```

查看执行成功后的job的pods，使用以下命令
```
kubectl get pods
```
列出机器可读的所有特定job的pods，使用如下命令
```
pods=$(kubectl get pods --selector=job-name=pi --output=jsonpath={.items..metadata.name})
echo $pods
```
这里的选择器和job的选择器是一样的，--output=jsonpath选项使用了特定的表达式来从返回的pods中列出pod的名字

## Writing a Job Spec
和其他的Kubernetes配置一样，job规范也需要apiVersion kind 和metadata 字段
job也需要一个.spec部分

#### Pod模板
.spec.template是.spec的唯一一个必须的字段
.spec.template是一个pod 的模板。 它和pod的模式一样，除了他是嵌套的而且没有apiVersion和kind字段
除了这些pod必须的字段外，job中的pod模板必须有一个合适的标签和一个合适的重启策略

这个唯一的RestartPolicy只能为Never或者OnFailure

#### Pod selector
.spec.selector是一个可选的字段，在大多数的场景中不用指定它。

#### Parallel Jobs
共有三种主要类型的job

1， 非并行的job
  ** 

2， 含有固定完成数量的并行的job

3， 包含work queue的并行job： 不指定.spec.completions, 使用默认的.spec.parallelism。pod之间相互配合或者有个外部的服务来协调他们

























