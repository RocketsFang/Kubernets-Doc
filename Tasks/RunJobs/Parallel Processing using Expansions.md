#Parallel Processing using Expansions
这个例子，我们将会去运行多个Kubernetes Jobs，这些jobs使用一个通用的模板。如果你想了解基本的非并行的job可以参阅jobs那一章节

## 基本模板的扩展
首先，下载这个模板并且命名为job-tmpl.yaml
```
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
      spec:
        containers:
        - name: c
          image: busybox
          command: ["sh","-c","echo Processing item $ITEM && sleep 5"]
        restartPolicy: Never
```
和Pod模板不同我们的job模板不是一个Kubernetes API类型，它就是个yaml格式展示的Job对象，在使用它之前有些占位符需要被填写。 $ITEM不是Kubernetes语法
在这个例子中，容器所要做的事情就是echo一个字符串并且休眠一会儿。在正真的场景中，所要做的事情会是重要的计算，比如渲染一个电影，处理数据库的特定区域的
数据行。$ITEM参数就会被指定比如是电影的帧数或者数据库行数。
这个job和他的pod模板会有一个标签jobgroup=jobexample。这个标签没有什么特殊的。它只是辅助我们去操作所有的job。我们同样设置了相同的标签到pod的模板中，
来方便我们去和job一起操作pods。当这个job被创建后，系统会为它的pod添加跟多的标签用以区分各个pods。 这个jobgroup不是kubernetes特殊的标签，你可以替换
你喜欢的标签。

## 模板的多个参数
在第一个例子中，我们的实例模板中包含有一个参数，并且这个参数是被用作标签的，而这个标签纸是被限制的。
本节的例子会有点复杂他用的jinja2模板来生成对象，我们会用在线的python脚本在转换这个模板为一个文件。
首先，复制粘贴这个模板到job.yaml.jinja2 































































