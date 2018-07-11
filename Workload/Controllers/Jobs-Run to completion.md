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
* 通常只有一个pod是启动的，除非pod失败了
* 只要pod成功终止了，job就完成

2， 含有固定完成数量的并行的job
* 指定一个正数给.spec.completions字段
* 当有一个成功的pod，job就完成，范围值是1到.spec.completions
* 还没有实现：每一个pod传入一个不同的索引值，范围是1到.spec.completions

3， 包含work queue的并行job： 不指定.spec.completions, 使用默认的.spec.parallelism。pod之间相互配合或者有个外部的服务来协调他们
*  每一个pod都具有相互感知的能力进而知道其他pod的进度，从而是job完成
*  如果任何一个pod成功终结，则不会有新的pod被创建出来
*  一旦至少有一个pod成功终结，则所有的pod终结，进而job完成
*  一旦任何pod成功终结，则其他的pod不会继续处理任何工作或者输出结果。这些pod都应该准备终结

对于一个非并行的job，我们可以同时给.spec.completion 和 .spec.parallelism 不设定值。这样他们会使用默认的值1

对于固定完成数量的并行job，.sepc.completions应当被设置为期望的完成值。对于.spec.parallelism可以设定值，或者使用默认值1

对于work queue job， .sepec.completions不能设定值，.spec.parallelism设置为正数

请参考job pattern章节获得更多的如何使用不同的job知识。

### 控制并行
并行执行的数量.spec.parallelism的值为非负数。如果不设置它则默认值1会被使用，如果设定为0，则job会立即暂停知道这个值增加。

真正并行的job（任何时刻运行的pod数量）数量会或多或少于.spec.parallelism请求的数量，其原因如下：
* 对于固定完成数量的job，真正并行运行的pod数量不会超过剩余要去完成的工作pod数量。 大于.spec.parallelism指定的数量将会被忽略
* 对于work queue job， 如果有pod成功完成则不会有性的pod被启动， 然而正在运行的pod会等他们完成工作
* 如果控制器没有及时的反馈
* 如果由于硬件原因控制器没有创建出新的pod，那么运行的pod会少于请求的数量
* 鉴于在同一个job中创建pod失败的经历，控制器会节制创建新的pod
* 当pod需要优雅的推出，他需要是时间来退出

### 控制pod和container的失败
会有很多原因导致Pod中的container失败，比如container中进程不正常退出，由于内存资源过大而container被cgroup终止等等各种原因。如果此类事件发生，我们
给.spec.template.spec.restartPolicy="OnFailure"，那么Pod就始终存在于node上，container会被重新执行。那么你的程序则需要处理此类场景当container
需要在本地重启，或者指定.sepc.template.spec.restartPolicy="Never"， 参阅pods-state来获得更多的restartPolicy信息。
由于各种原因整个pod也会失败，比如当一个pod被分配到一个node上，而这个node正好在被升级或者重启后者删除等等；或者container的.spec.tempalte.spec.
restartPolicy="Never"。当pod失败了job控制器会启动一个新的pod。因此，你的程序需要考虑当程序在一个新的pod中被重启，特别是程序需要处理临时文件，文
件锁没有完成的输出。
注意，尽管你指定了.spec.parallelism=1 和 .spec.completions=1 和 .spec.tempate.spec.restartPolicy="Never"，同样的程序可能会被启动两次。
如果你指定了.spec.parallelism 和 .spec.completions的值都大于1，那么会同时有多个pod运行，因此，你的pod必须的容忍不一致。

#### Pod失败补偿策略
有这样的情况，当由于配置原因导致程序逻辑错误我们的程序在重试过后希望他能失败，为达到这个目的设置.spec.backoffLimit 让程序在失败退出前做尝试。它的
默认值是6，Job控制器会根据失败的job来创建新的job这个新job的重试延迟是指数递增的但是不会超过6分钟。如果没有新的失败Pod产生，那么Job在做下一次状态
侦测是会重置失败补偿次数。

## Job终止和清理
当Job完成后，将不会再有新的Pod被创建出来，但是已有的pod也不会被清理掉。保留这些Pod可以为我们后来做日志分析。Job执行完后job对象也会被保留下来供我们
查看它的状态。他们的删除与否完全交给用户来决定。可以使用kubectl delete明恋来删除Job 比如: kubectl delete jobs/pi，或者  kubectl delete -f 
./job.yaml), 当你使用kubectl删除job时，由他创建的pods也会一并被删除掉。
默认，Job的执行不会被外界干扰除非Pod失败了，就像上面.spec.backoffLimit 描述的那样。另一个终止Job的方法是设置一个活跃最终时间，可以用Job
的这个字段.spec.activeDeadlineSeconds来设置一个秒值。
activeDeadlineSeconds适用于Job的存在时间，和有多少个Pod被创建没有关系。一旦一个Job的activeDeadlineSeconds到了，这个Job和它创建的Pod都会被终结
掉。 结果就是Job的状态信息reason:DeadlineExceedd

注意Job的.space.activeDeadlineSeconds要优先于.spec.backoffLimit。因此，Job重试一个或多个Pods但他不会去部署一个Pod一旦Job的存活最终时间.spec
.activeDeadlineSeconds到了,而backoffLimit的重试次数没有到。
举个栗子：
```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-timeout
spec:
  backoffLimit: 5
  activeDeadlineSeconds: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle","print bpi(2000)"]
      restartPolicy: Never
```

## Job 模式
Job对象可以被用来支持实现稳定的并行执行的pods，他们并不是被设计成常见的科学计算中的需要彼此密切通信的并行过程。它不支持可以并行运行的一系列独立的但
又相关的工作单元。他们跟多的用于邮件发送，渲染框架，扫描一个区间的NoSql键值等等。
在复杂的系统中，可能会有多个不同系列的工作单元。这里我们只考虑一个工作单元，用户想把他们一起做完用job批处理。
这里列出一些不同的并行处理计算的模式，他们有各自的优缺点。权衡结果如下：
* 每一个Job对象负责一个工作单元与一个job对象负责所有的工作单元。后者跟适用于大量的工作单元。要是使用前者的话会对用户系统造成负担
* 用相同数量的pod来完成相同数量的工作单元，用一个pod来完成多个工作单元。 前者很明显只需要少量的对已有的code和containers的改动。 后者更适用于大量的
工作单元，和之前的原因一样
* 多个方法共同用一个工作队列，这个的需要一个队列服务，要对已有的程序做修改来世他们利用这个工作队列。 其他方法会更适用于一个已经被容器话的应用

权衡列表

Pattern | Single Job Object | Fewer Pods than work items? | Use app unmodified? | Works in Kube 1.1?
---- | --- | --- | --- | --- 
Job Template Expansion |   |   | ✓ | ✓ 
Queue with Pod per Work Item | ✓ |  | sometime | ✓
Queue with Variable Pod Count | ✓ | ✓ |  | ✓
Single Job with Static Work assignment | ✓ |  | ✓ |

当你用.spec.completion指定完成， 那么每一个由Job控制器创建的pod就都还有相同的spec。这就意味着所有的pod都将包含有相同的命令行，相同的存储，相同的
环境变量。这些模式就是用不同的方法来安排pod去做不同的事情。

## 高级用法
### 定制你自己的pod选择器
通常情况下，当我们创建一个Job对象时，我们不需要指定.spec.selector，当job被创建时系统默认的逻辑会自动添加这个字段。它会选择一个不会与其他Job重复的
值。 
然而，在一些个场景中，我们可能需要覆盖这个自动生成的值。我们可以使用.spec.selector。
当我们选择这样去做的时候，一定要小心，如果这个标签选择器不是唯一的，那么它将会命中不相关的pods，这些不相关的pods会被删除掉，或者Job会用这些个pods
来计算工作量的完成，或者这个Job亦或者这个Job和被影响到的Job都不会创建pods或者执行pod来完成工作量。 这个不唯一的标签选择器还会导致其他的控制器比如：
ReplicationController和他的pods不可预见的问题。Kubernets不会去检查阻止用户这么去做，如果我们自定义了.spec.selector

这有个例子可供想使用这个功能的用户参考
假设Job old已经看是执行。 当你期望已经存在的pods继续执行，而将要被创建的剩余的pod去使用不同的pod模板，还想要更改job的名字。 我们不能够通过更新Job
来达到这个目的，因为这些字段是不可更新的。 因此， 我们要删除job old而保持它的pods继续执行，使用kubectl delete jobs/old --cascade=false， 在删除之前我们的标记一下应该用什么selector。
```
kind: Job
metadata:
  name: old
  ...
spec:
  selector:
    matchLabels:
      job-uid: a8f23sdfa09dfsjlkdsf
    ...
```
当你创建新的job并且使用新的名字new我们的显示的指定相同的标签选择器。 因为已经存在的pods具有标签job-uid=a8f23sdfa09dfsjlkdsf， 他们也会受控于job new。你需要为这个新的job指定manualSelector: true 因为你不需要系统自动为你生成标签选择器
```
kind: Job
metadata:
  name: new
  ...
spec:
  manualSelector: true
  selector:
    matchLabels:
      job-uid: a8f23sdfa09dfsjlkdsf
  ...
```
这个新的job会有一个新的uid 不同于a8f23sdfa09dfsjlkdsf。 设置manualSelector: true 就是要告诉系统我们知道我们在所生么，并且请忽略这个

## 可选项
### 裸Pods
当pod正在运行的node被重启或者失败时，pod会被终止掉并且不会不重启。 然而，Job将会创建新的pods来覆盖这个被终结的pods。 鉴于这个原因，我们推荐使用job而不是裸pod尽管我们的程序只需要一个pod。

### 复制控制器
Job是可以说是Replication Controller的补偿，一个复制控制器管理的pods是不被期望终结的比如像 web servers，而Job管理的pod是被期望可以终结的比如：批处理jobs。
正如在Pod生命周期部分论述的，Job是Pod被设置了RestartPolicy为OnFailure或者Never的Pod，如果RestartPolicy不被设置，默认的值则是Always。

### 单一Job启动控制器Pod
另一个模式是说一个单一的Job创建的pod，而这个pod又去创建了其他的pods，而这个由Job直接创建的pod扮演了这些间接创建的pod的控制器。这样虽然有了灵活性但是它太复杂不易和kubernetes集成。
这个使用场景可以是一个Job创建了一个Pod这个Pod运行一个脚本这个脚本启动了一个Spark主控制器，运行一个spark设备，然后清理。
还有个高级的场景是总体的进程保证Job对象去完成工作量，也会完全控制去创建什么样的pods及分配什么样的工作给他们。




























































































