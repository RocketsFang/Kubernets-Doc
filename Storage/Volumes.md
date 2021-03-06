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

## azureDisk
A azureDisk是用来将微软公司的Azure Data　Disk挂载到Pod

## azureFile
azureFile使用来将微软的文件存储（smb2.1和3.0）挂载到pod中

## cephfs
cephfs存储是用来将一个已经存在的CephFS存储挂载到你的Pod中，他不想emptyDir，当pod被移出后存储也会被移出；相反cephfs会保留里面的内容并且他很少会被卸载掉。这就意味着CephFS存储可以预埋一些数据，然后这些数据可以被移交给Pods。CephFS可以被同时挂在多个写程序。

## configMap
configMap资源它为我们提供了一种注入配置数据到Pod中的方法，存在于configMap中的对象可以被以configMap存储类型的方式被引用然后被运行于容器中的程序所消费。
当你要引用这个存在于configMap中的对象是，你只要指定这个对象的名字即可。我们可以指定一个路径来指向ConfigMap中的特定条目。比如我们挂在log-config ConfigMap到一个Pod中去。我们可以使用以下的yaml：
```
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: long_level
            path: log_level
```
log-config ConfigMap被挂载为一个存储，所有存储在log_level条目中的内容都会被挂载到Pod中的/etc/config/log_level目录中。注意这里的路径起源于存储的mountPath和以log_level为键名的path


## 利用subPath
有时，在一个容器当中去共享一个存储是十分有用的。volumeMounts.subPath属性可以在引用的存储中指定一个特定的子目录而不是使用它的根目录。
这儿有个LAMP配置它使用了单独的共享的存储。HTML文件被放到这个存储的html文件夹，数据库将会被存放到相同存储mysql文件夹。
```
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
  containers:
  - name: mysql
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "rootpassword"
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: site-data
      subPath: mysql
  - name: php
    image: php:7.0-apache
    volumeMounts:
    - mountPath: /var/www/html
      name: site-data
      subPath: html
  volumes:
  - name: site-data
    persistentVolumeClaim:
      claimName: my-lamp-site-data
```

### subPath和扩展环境变量的结合使用
功能状态： Kubernetes v1.11 alpha
subPath目录的名可以用Download API环境变量指定。 在使用这个功能之前你必须的启动VolumeSubpathEnvExpension。
这个例子中，Pod会使用subPath在主机存储/ar/log/pods中创建一个子目录pod1，这个是来自Download API中pod的名字。主机目录/var/log/pods/pod1将会被挂载到容器中的/logs
```
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox
    command: [ "sh", "-c", "while[ true ]; do echo "hello"; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs
      subPath: ${POD_NAME}
   restartPolicy: Never
   volumes:
   - name: workdir1
     hostPath:
       path: /var/log/pods
```

### downloadAPI
downloadAPI 存储是被用来使downloadAPI 数据能够对应用可见，他会挂载一个目录并且把请求数据以普通文件方式写入到这个目录中。

### emptyDir
emptyDir会在Pod分配给Node之前就会被创建出来，然后会和Pod一样一直存在于相同的Node上。正如其名那样，它的初始状态是空的，尽管这个存储在每个容器中会被挂载到相同或者不同的路径中，但这些容器就可以读写emptyDir存储中的相同文件。一但由于各种原因Pod被移出Node，那个在emptDir中的数据就会被永远的删除。
**注意**一个容器的崩溃不会导致Pod被移出Node，所以在容器崩溃期间emptyDir中数据是安全的。
emptyDir的一些用法：
1, 附属磁盘空间，比如一次基于磁盘的合并排序
2, 对崩溃恢复的一次长时间计算的检查点
3, 在webserver容器提供数据时用来存放内容管理容器抓来的所有文件

默认，emptyDir存储可以是在Node上的任何媒介，可能是磁盘，SSD或者网络存储，这些取决于你的环境。然而你也可以通过设置‘emptyDir.medium字段为Memory来告诉Kubernetes挂载tmpfs(RAM-backed 文件系统）。尽管tmpfs十分快，也要明白他和磁盘不一样，tmpfs会在node重启后被清除而且你可以操作的文件是要被记入你容器的内存限制的
```
实例Pod
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### fc(fibre channel)
fc卷可以将一个存在的fibre channel存储挂载到Pod中。 可以在存储配置中通过参数targetWWN指定单一的或者多个目标全球唯一名称。 如果有多个全球唯一名称被指定那么targetWWNs参数则期望这些名称来自多条连接。
**重要提示** 事先我们的配置光纤通道来对目标全球唯一名分配和屏蔽这些存储，这样Kubernetes主机就能够访问他们了。
实例Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: fc
spec:
  containers:
    - image: kubernetes/pause
      name: fc
      volumeMounts:
        - name: fc-vol
          mountPath: /mnt/fc
  volumes:
    - name: fc-vol
      fc:
        targetWWNs: ['2243r23rfewer', '234242sfadsf']
        lun: 2
        fsType: ext4
        readOnly: true
```


### flocker
Flocker是一款开源的集群级别的容器数据管理器。他用来管理和编排由各种不同存储终端支撑的数据卷。 

flocker卷可以将Flocker数据集挂载到Pod中去，如果这些数据集在Flocker中不存在，这就需要我们用Fokcer命令行或者Flocker API来创建好。 如果数据集已经存在了那么Flocker就会把它重新关联到Pod被调度到的节点机器上。 这就意味着数据可以在Pods之间按需移交。
**重要提示** 在使用Flocker之前我们的要安装并且配置好它
实例Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: flocker-web
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
      volumeMounts:
        - name: www-root
          mountPath: "/usr/share/nginx/html"
  volumes:
    - name: www-root
      flocker:
        datasetName: my-flocker-vol
        
---
apiVersion: v1
kind: Service
metadata:
  name: flocker-ghost
  labels:
    app: flocker-ghost
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: flocker-ghost
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: flocker-ghost
  labels:
    purpose: demo
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: flocker-ghost
    spec:
      containers:
      - name: focker-ghost
        image: host:0.7.1
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - containerPort: 2368
          hostPort: 80
          protocol: TCP
        volumeMounts:
          - name: ghost-data
            mountPath: /var/lib/ghost
      volumes:
        - name: ghost-data
          flocker:
            datasetName: my-flocker-vol
```

### gcePersistentDisk
gcePersistentDisk卷可以将GCE 持久化磁盘挂载到Pod中。 和emptyDir在每次Pod被清除是卷也会被清除不一样，持久化的磁盘会被保留并且挂载的卷很少会被取消挂载。这意味着持久化磁盘可以被预埋数据，然后这些数据可以在Pods间移交。
实例Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4
```

### glusterfs
glusterfs卷允许一个Glusterfs卷被挂载到Pod中。 和emptyDir会在Pod被移出时相应的卷也会被移除不同，glusterfs卷内容是能够被保留并且很少被取消挂载。这就意味着glusterfs卷是能够被预埋数据然后在Pods之间被移交。 GlusterFS可以被多个写程序同时挂载。
**重要提醒** 在你使用glusterFS之前必须安装GlusterFS并且配置它

### hostPath
hostPath卷可以将主机上的目录或者文件挂载到你的Pod中去。 并不是所有的Pod都会用到这个功能但是他位一些应用提供一个强有力的逃生舱
比如，hostPath的一些使用：
1，运行需要访问Docker内部文件的容器，可以使用/var/lib/docker的hostPath
2，在容器中使用cAdvisor  使用/sys的hostPath
3，允许Pod来指定在Pod运行之前是否需要hostPath指定的目录存在，如果不存在是否需要被创建，应该以怎样的形式存在

除了必须的path属性，对hostPath卷用户还可以可选的指定type属性，type属性支持的值有：


| Value | Behavior |
|:------|:---------|
| | 默认为空值这是为了向后兼容，在挂载hostPath卷之前不需要做任何检查 |
| `DirectoryOrCreate` | 如果在指定的路径不存在，会创建一个和kubelet在同一个组和所有权且权限值为0755的空目录 |
| `Directory` | 指定的目录必须存在 |
| `FileOrCreate` | 如果在指定的路径下面没有任何文件，回去创建一个和kubelet在同一个组和所有权且权限值为0644的空文件 |
| `File` | 指定的路径下文件必须存在 |
| `Socket` | 在指定的路径下UNIX socket必须存在 |
| `CharDevice` | 在指定的路径下字符设备必须存在 |
| `BlockDevice` | 在指定的路径下块设备必须存在 |


在使用这些type是要注意，因为：
1， 在不同的节点上由于不同的文件会导致使用相同配置创建出来的Pods有不同的行为
2， 按照计划当Kubernetes使用了资源敏感调度方式，hostPath使用type类型会被忽略
3， 底层主机上创建出来的文件或者文件夹只有root具有写权限。 这样一来你的在特权容器中运行你的进程或者修改文件的权限使得你的进程可以往hostPath卷写入数据。
实例Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
    containers:
    - image: k8s.gcr.io/test-webserver
      name: test-container
      volumeMounts:
      - mountPath: /test-pd
        name: test-volume
    volumes:
    - name: test-volume
      hostPath:
        path: /data
        type: Directory
```

### ISCSI
ISCSI卷允许一个已经存在的iSCSI卷被挂载到你的Pod中。 和emptyDir卷在Pod被移出Node时卷就会被清除不一样，iscsi卷内容会被保留而且卷很少会被取消挂载。这就意味着iscsi卷可以事先装入数据这些数据可以在Pods间被移交。
**重要提示** 在使用iscsi卷功能之前你必须有你自己的iscsi服务器而且服务器上已经有可用的卷
iscsi的特性是它可以被挂载为多个同时只读的卷。这就意味着你可以事先在卷中填充数据然后按照需求给多个Pods提供并行的只读服务。不幸的是iscsi卷只能被挂载为一个读写模式的消费者，多个写模式是不被允许的。
实例Pod 
```
---
apiVersion: v1
kind: Secret
metadata:
  name: chap-secret
type: "kubernetes.io/iscsi-chap"
data:
  discovery.sendtargets.auth.username:
  discovery.sendtargets.auth.password:
  discovery.sendtargets.auth.username_in:
  discovery.sendtargets.auth.password_in:
  node.session.auth.username:
  node.session.auth.password:
  node.session.auth.username_in:
  node.session.auth.password_in:

---
apiVersion: v1
kind: Pod
metadata:
  name: iscsipd
spec:
  containers:
  - name: iscsipd-ro
    image: kubernetes.oi/pause
    volumeMounts:
    - mountPath: /mnt/iscsipd
      name: iscsivol
  volumes:
  - name: iscsivol
    iscsi:
      targetPortal: 127.0.0.1
      iqn: iqn.2015-02.example.com:test
      lun: 0
      fsType: ext4
      readOnly: true
      chapAuthDiscovery: true
      chapAuthSession: true
      secretRef:
        name: chap-secret
 ---
 apiVersion: v1
 kind: Pod
 metadata:
   name: iscsipd
 spec:
   containers:
   - name: iscsipd-rw
     image: kubernetes/pause
     volumeMounts:
     - mountPath: /mnt/iscsipd
       name: iscsipd-rw
   volumes:
   - name: iscsipd-rw
     iscsi:
       targetPortal: 10.0.2.15:3260
       portals: ['10.0.2.16:3250']
       iqn: iqn.2001-05.com.example:storag.kube.sys1.xyz
       lun: 0
       fsType: ext4
       readOnly: true
 
```

### local
local卷代表了本地存储设备比如：磁盘，分区，目录
Local卷只能被用在静态创建出来的PersistentVolume。不支持动态提供PersistentVolume。
相比于hostPath卷，local卷可以以持久的便易的方式被使用，他不需要手动调度Pods到节点上去，系统通过查看PersistentVolume的节点亲和性来感知卷的节点限制。
然而，local卷仍然受到底层节点可见性的限制不是对所有的应用程序都适用。 如果节点变得不健康，local卷就会变得不可访问，使用这个卷的Pod就会变得不能运行。 应用程序必须依赖底层磁盘的持久性容忍这种可见性，以及潜在的数据丢失。

下面的例子就是使用了local卷和nodeAffinity的PersistentVolue规范描述
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorterms:
      - matchExpressions;
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

当使用local卷时PersistentVolume的nodeAffinity是必须的。 它可以使Kubernetes调度器使用local卷正确的调度Pod到正确的节点上。PersistentVolume volumeMode属性现在能被设置为‘块’（而不是默认值‘Filesystem')来对外暴露local卷为原生块卷。 这个需要启用Alpha功能开关。

当使用local 卷，推荐创建设置volumeBindingMode为WaitForFirstConsumer的StorageClass。 延缓卷的绑定可以考虑更多的节点限制条件，比如资源需求，节点选择器，Pod亲和性和Pod反亲和性。

使用外部的静态提供者可以对local卷的生命周期进行分离管理。

### nfs
nfs卷可以将已经存在的NFS以共享的方式挂载到你的Pod中。 和emptyDir卷在Pod被移出Node时卷会清空并被删除， nfs的卷内容会被保留并且卷很少被解除挂载。这意味着NFS卷可以被预先填充数据，这写数据可以被移交给Pods。 NFS可以同时挂载多个写程序。

### persistentVolumeClaim
persistentVolumeClaim被用来挂载PersistentVolume到Pod中。 PersistemVolumes是不需要知道特定云环境详细内容的前提下要求持久的存储。

### projected
一个计划卷可以将几个已有的卷源映射到相同的目录中去
目前，一下的卷源可以被作为计划卷：
* secret
* downloadAPI
* configMap
* servieAccountToken

Pod相关的资源都必须和Pod在同一个namespace中。 在Kubernetes1.11中引入的service account token计划

#### Pod中使用了secret，downloadAPI和configMap的实例
```
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
        name: mysecret
        items:
          - key: username
            path: my-group/my-username
      - downloadAPI:
        items:
          - path: "labels:
            fieldRef:
              fieldPath: metadata.labels
          - path: "cpu_limit"
            resouceFieldRef:
              containerName: container-test
              resource: limits.cpu
      - configMap:
          name: myconfigMap
          items:
            - key: config
              path: my-group/my-config
```

使用了多个不是默认权限secrets的Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: volme-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: /projected-volume
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - secret:
          name: mysecret2
          items:
            - key: passw0rd
              path: my-group/my-password
              mode: 511
```
在sources的规范描述下列出了每一个projected卷源。 参数有点儿像两个异常似的。
* 对于secrets来说，为了和ConfigMap名称一致secretName字段被更改为name
* defaultMode只能在projected层级指定不能在卷源级别指定。然而，正如上面列出的你可以显示的为每一个projection设置mode

当TokenRequestProjection功能被启用，你可以将当前的serviceaccount的token注入到Pod的指定路径中去。
```
apiVersion: v1
kind: Pod
metadata:
  name: sa-token-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: token-vol
      mountPath: /service-account
      readOnly: true
  volumes:
  - name: token-vol
    projected:
      sources:
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
```
实例Pod中有一个包含了注入了service account token的projected卷，这个token可以被Pod的容器用来访问Kubernetes API server。audience字段指定了token的目标受众。 token的接收者必须能够使用在token受众中指定的标识符标识他自己，否则应该拒绝token。 这个字段是可选的默认值是API server的标识符

### portworxVolume
portworxVolume是一个与Kubernetes超融合运行的弹性块存储层。Portworx指纹存储是在服务器上，这些个服务器基于性能层层排列并在在多个服务器容量之间聚合。 Portworx运行在虚拟机中或者在Liunx裸设备节点上

portworxVolume可以通过Kubernetes被动态创建或者预先创建然后在kubernetes Pod内部被引用。 以下是一个预先提供的Portworx 卷：
```
apiVersion: v1
kind: Pod
metadata:
  name: test-portworx-volume-pod
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /mnt
      name: pxvol
      
  volumes:
  - name: pxvol
    portworxVolume:
      volumeID: "pxvol"
      fsType: "<fs-type>"
```


### secret
secret卷被用来传递敏感信息，比如： 密码到Pod中去。 你可以把secrets存放到Kubernetes API然后以文件形式挂载他们让Pod使用他们而不需要直接拷贝到Kubernetes。 secret卷以tmpfs为支撑这样他们从不会被写到稳定的存储中去。































