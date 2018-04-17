# Secrets
Secret对象的用意是用来存放敏感的信息，比如：密码，OAuth令牌和SSH公钥。 把这个信息放到Secret对象里面比把它直接放到Pod的定义里或者Docker镜像里面更加的灵
活和安全。


## Secrets概览
Secret就是保存了数量很少的敏感数据的对象，比如：密码，令牌，公钥。 这样的信息可以被放到Pod的详细规范中或镜像中，而把他们放到Secret对象里面可以获得更多的
怎样去使用他们的控制权， 从而降低被暴露的风险。

用户和系统都能去创建Secret。
当使用Secret的时候Pod需要引用Secret。 Pod通过两种方式使用Secret：1， 以文件方式被挂在的容器使用；2，Kubelet使用它来为Pod拉去镜像

## 内建的Secrets

### 服务帐号使用证书API自动创建和附加Secrets
Kubernetes会自动的创建包含有可以访问API的证书的Secrets， 而且还能自动的修改你的Pod来使用这个类型的Secret。
如果需要这个自动的创建和使用API证书的功能能被不启用或者重写。 然而如果是你需要这样做的目的仅仅是安全的访问APIServer,那这是推荐的工作流程。
请查看Service Account的文档来查阅详细的Service Account工作方式


### 创建你自己的Secrets
#### 使用kubectl 创建Secret
比如说有一些Pod要访问数据库，Pod要使用的用户名和密码应该被放到本地的./username.txt文件和./password.txt文件中。
```
#create files needed for rest of exmple
echo -n "admin" > ./username.txt
echo -n "1f2d3s4a" > ./password.txt
```
kubectl create secret命令就会将这两个文件打包并在APIServer上创建对应的对象。 
```
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
secret 'db-user-pass' created
```

可以通过命令查看secret：
```
kubectl get secrets
NAME            TYPE               DATA       AGE
db-user-pass    Opaque             2          51s

kubectl describe secrts/db-user-pass
Name:  db-user-pass
Namespace: default
Labels: <none>
Annotations: <none>
Type: Opaque
... ...
```
默认可以通过get 或者describe来显示secret的内容。这就能保证secret不会不小心暴露或者被放到日志文件里面。

#### 手工创建Secret
你可以以json或yaml文件格式创建Secret对象， 然后在APIServer上创建这个对象。
每一个条目都是经过base64编码过的：
```
echo -n "admin" | base64

echo -n "asdf3"　｜base64
```
然后创建Secret详细描述规范：
```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: ... ...
  password: ... ...
```
数据信息是一个图数据格式。它的键名必须只由字符，‘-’，‘_‘,’.‘和数字组成。他的键值则是任何数据但必须是经过base64编码过的

使用kubectl create 创建这个secret
```
kubectl create -f ./secret.yaml
secret 'mysecret' created
```
> 编码的注意事项： 序列化后的JSON和YAML值都是用base64编码过的字符串。换行符在这个字符串中是无效的会被忽略掉。 在OS/X系统中用户应当避免使用-b选项来分割长数据行。相反Linux用户则应该田间-w 0命令行选项。

## 反序列化Secret
Secrets可以通过kubect get secret命令来获取。比如以下命令就可以获取我们之前创建的Secrets对象：
```
kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
kind: Secret
metadata:
  creationTimestamp: 2016-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: cfee02d6-c137-11e5-8d73-42010af00002
type: Opaque
```
反编码密码字段
```
echo "sfdafpiwjqprekDfew" | base64 --decode
```

## 使用Secrets
Secrets可以以数据卷的方式挂载使用或以环境变量的方式被Pod中的容器使用。 此外它们还可以不用被暴露给Pod而被系统的其他部分使用，比如：他们可以存放集群安全信息然后系统的其他部分就能代表集群用户来和外部系统进行交互。

### 从Pod中以文件的方式使用Secrets
消费Pod中的数据卷
1,  创建或者使用一个已经存在的Secret，同一个Secret对象可以被多个Pod引用
2,  修改Pod的定义，在spec.volumes[]加入数据卷。可以给这个Secret起任何合法的名字，之后Pod的定义文件是这样的spec.containers[].volumeMounts[].readOnly=true 和 spec.container[].volumeMounts[].mountPath指向一个没有被使用的路径，我们引用的secret对象就会在这个路径中出现。
4,  修改我们的应用程序使他们使用这个路径中的文件。路径中的文件名会以data图中的键名创建文件。
这是个例子：
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```
每一个被引用的secret对象都会在spec。volumes中出现。如果在Pod中有多个secrets被使用，每一个container都的有他们iji的volumeMounts配置块，但是spec.volumes是只有一个的。

你可以打包多个文件到一个secret中也可以使用多个secrets对象，选一个你方便的方式来使用就好了。































