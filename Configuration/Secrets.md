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
> 编码的注意事项： 序列化后的JSON和YAML值都是用base64编码过的字符串



























