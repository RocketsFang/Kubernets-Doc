#Running automated tasks with cron jobs
我们可以使用CronJob去创建一个基于时间计划表的Job，这些自动创建的job就好像Linux或者Unix系统里面的Cron任务

Cron Job在完成周期性，重复性任务方面很有用处，比如就好像备份或者发送邮件，Cron Job也可以用来安排特定时间的个体任务，比如你想在环境空闲期安排一个job。

注意：CronJob资源是在batch/2alpha1 API Group在Kubernetes 1.8是被不推荐使用了。取而代之是batch/v1beta1，这个API默认是被激活的，在本片论述中的例
都是使用了batch/v1beta1 API版本。

Cron Job自生具有局限性和特质。 比如，在特定的情况下，一个Cron Job会创建多个jobs，因此jobs因该是幂等的。

## 准备开始
* 准备一个Kubernetes集群，配置kubectl命令行工具去操作这个集群。
* Katacoda
* Play with Kuberetes
使用kubectl version来核对版本信息
* 我们的需要一个Kubernetes 版本大于1.8的集群。 对于之前的版本的显示的激活batch/v2alapa1 通过使用--runtime-config=batch/2alpha1来告诉API Server
然后重启API server和控制管理器组件。

## 创建一个Cron Job
Cron Job需要一个config 文件，这个例子配置了.spec文件来每分钟输出目前的时间和问候语句
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      tempate:
        containers:
        - name: hello
          image: busybox
          args:
          - /bin/sh
          - -c
          - date; echo hello from the kubernetes cluster
        restartPolicy: onFailure
```
使用如下的命令去创建一个cron job
```
kubectl create -f ./cronjob.yml
cronjob "hello" created
```

可选的，我们也可以使用如下的kubectl run命令去创建一个cron job而不用大段的配置信息
```
kubectl run hello --schedule="*/1 * * * *" --restart=OnFailure --image=busybox -- /bin/sh -c "date; echo hello from Kubernetes cluster"
```
创建完cron 对象后，使用这个命令去获取它的状态
```
kubectl get cronjob hello
```
正如你看到的，cron job还没有被安排执行任何的job。 我们也可以监控job会在每一分钟的时候被创建
```
kubectl get jobs --watch
```
现在我们能够看到job被‘hello’cron job运行。 我们可以通过这个命令来查看被cron job所安排执行的job
```
kubectl get cronjob hello
```
这时我们就能看到job在指定的时间（last-schedule字段）被cron job ‘hello’成功的安排创建。目前没有活跃的job是说job已经被执行或者失败了。
现在，找到上一个job执行时创建的pods并且查看pods 的输出信息。我们可以看到job名字和pod名字是不一样的。
```
pods=$(kubectl get pods --show-all --selector=job-name=hello-4111706356 --output=jsonpath={.items..metadata.name})
kubectl logs $pods
```

## 删除Cron Job
当不再需要Cron Job对象时，可以使用kubectl delete cronjob
```
kubectl delete cronjob hello
```
删除cron job对象时会连带它创建的job和pods一起被删除，这是也会停止创建新的job。 可以参阅 garbage collection章节

## 创建一个Cron Job规范文件
和其他Kubernetes配置一样，cron job也会需要apiVersion， kind和metadata属性。 有关基本信息可以参阅 deploying applications和使用kubectl to
manage resource文档
注意：对所有的cron job的修改，特别是他的.spec，只会对下面的生效

### 计划
.spec.schedule是一个.spec 必须的字段。它只接受cron格式的值，比如：0 *  *  * * 哦人@hourly， 作为他的job被创建和执行的时间。

### Job 模板
.spec.jobTemplate是job的tempate也是一个必须的字段。他是完全和job的模板一致的除了他没有apiVersion和kind字段。





















































