# CronJob
CronJob是在Job的基础上管理时间，换句话说： 
* 在一个特定的时间点上
* 重复的在一个特定的时间点

一个CronJob就像是Crontab文件中的一行指令。 它在指定的时间表中周期性的执行Job，使用了Cron的格式被编写。
可以参阅Running automated task with cron jobs 去获得如何去创建并且如何利用cron job，还提供了一个CronJob的spec例子以供学习之用。

## Cron Job的局限性
Cron Job会在一个计划周期内将近船舰一个job对象。 这里我们说将近是因为在有些情况下会会有两个job 对象被创建，亦或者没有job对象被创建。我们只能尝试着
去防止此类的情况发生，但是并不能够保证一定。因此job的必须是幂等的。
如果startingDeadlingSeconds属性被设置到了一个很大的值或者只是默热值，并且concurrencyPolicy属性被设置为Allow，那么job就会至少被运行一次。
如果CronJob控制器没有运行或者坏了好长时间比如是在job开始执行的时间到开始执行的时间加上startingDeadlineSeconds,那么job就不会被执行。如果这个时间
周期跨越了多个开始执行的时间而且concurrencyPolicy被设置为不能并行执行。 比如，一个cron job被设置为从08：30：00开始执行它的startingDeadlineSeconds
被设置为10，如果Cron Job控制器恰好发生故障时间段是08：29：00到08：42：00， 这个job就不会被触发，给startingDeadlineSeconds一个合理的较大的时间总比
不执行它要合理。

CronJob只会去创建符合他计划表设置的job，然后job接着负责管理具体执行工作的pod






































































