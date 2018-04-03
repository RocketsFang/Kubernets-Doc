# Connecting Applications with Services

## The Kubernetes model for connecting containers

到目前为止你可以个持续运行一个带有副本的应用程序并且你还可以将它开放到网络上。在讨论Kubernetes访问网络之前，值得和Docker常用访问网络的方式进行比较。

默认情况下，Docker使用主机私有方式来访问网络，所以各个container之间要想彼此互联只有他们同时处于一个物理机器上才可以。如果Docker要想进行跨节点相互访问，
就必须在物理机器所有的IP地址上开放端口，然后再由物理机器把流量代理或转发到container上去，这样一来在物理机器所有的IP上开放端口就成了必须要做的事情，这很
显然container必须做同一个IP上的端口协调工作或者具备动态分配能力。

在多个开发者他们做集群扩容操作时协调端口的使用只是想想都是件十分困难的事儿，而把用户置于集群级别的问题上对于他们来说也不能解决。 Kubernetes假设不管Pods
在何处他们之间都是可以互相通讯的。我们为每一个Pod指定了属于他们自己私有的集群IP地址他们不需要显示的相互建立连接或者映射他们Container的端口到节点的端口。
这就是说在一个Pod里面的Container可以个访问到本地上的所有端口，并且集群里面的所有的Pods相互是可见的不需要额外的网络地址转换（NAT，Network Address 
Translatoin），接下来的论述中你将看到怎样在这样的网络模型中去运行一个可靠的Services。
以下的例子中我们用到了简单的额nginx服务器来做概念的论证。同样的原型被应用到了一个更加复杂的场景中----[Jenkins CI Appication]
(http://blog.kubernetes.io/2015/07/strong-simple-ssl-for-kubernetes.html)

# 开放Pods到集群
我们之前做过这个例子，这次我们会关注网络这个方面。创建一个nginx Pod， 然后注意它的描述规范中有Container 端口的设定。
```
apiVersion: apps/v1
kind: Development
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```
之后你就可以从集群中的任何节点访问nginx服务。
你也可以使用ssh工具登录到集群中的任何一个节点机器上然后用curl命令访问这个IPs地址。
>注意: 容器没有使用默认节点上的80端口， 也没有特殊的网络地址转换（NAT，network address translation）来路由这个流量到你的Pod。这意味着多个ngnix
的Pods能够运行在同一个节点上而且还能在集群中使用同一个containerPort还能在一个集群中相互访问。和Docker一样，端口仍然在主机上被打开了，由于网络模型
的使用，对它的需求却没有那么的强烈了。

可以通过这篇文章来看看我们是怎么做到的
[How we achieve this](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)

# 创建一个服务
我们以扁平，集群级别的地址空间中的Pods上运行着nginx。理论上你可以直接访问这些Pods，但是如果一个Node挂掉了怎么办？ 在他上面运行的Pods也都一起挂掉了
Deploymnet会创建出来新的Pods，这些Pods会使用不同的IPs地址。 这就是Service要解决的问题。

在Kubernetes里面服务是对在集群中任何一个地方运行的一组Pods的逻辑抽象，这些Pods提供相同的功能。Services在创建之时会被分配一个唯一的IP地址（称之
为集群IP地址）。这个地址会绑定在服务的整个生命周期上，服务不死IP不变。Pods被配置成能访问Service的，这些同质的Pods和Service相互访问是能够做到负
载均衡的。

你可以是使用这个命令把你的两个ngix副本开放出来
```
kubectl expose deployment/my-nginx
```
这个命令等价于这样的yaml描述
```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```
这个描述规范会创建一个这样的服务，这个服务的目标TCP端口是80，任何有run:my-nginx标签的Pod都可以被Service代理的流向访问到，同时这些个Pods被暴露到
了一个抽象的服务端口targetPort上（targetPort：这个端口是container开放并接受请求的端口，port：是抽象的服务端口，其他的pods可以用这个端口
访问到服务）。可以查看这个包含所有支持的属性[列表]()。

查看创建的服务：
```
kubectl get svc my-ngix
```
正如之前提到的，一个Service是被很多的Pods所支撑的，提供Service的实现。这些Pods十一endpoints的形式开放的。Service的选择器不停的甄选并将结果POST
发送给同样称之为‘my-nginx’的Endpoints对象。遇到有不能提供服务的Pod，Service就自动的将个Pod从Endpoints对象中将其删除，新创建的满足Service选择器的
Pod又会自动地加入到endpoints对象中来。 
查看endpoints对象，注意他们的IPs地址和Pods的ip地址是一样的。
```
kubectl describe svc my-nginx
kubectl get ep my-nginx
```
现在你可以使用curl命令来测试nginx服务了在集群级别<cluster-ip>:<port> 在你集群中的任何一个节点上。注意服务的IP地址是完完全全的虚拟IP地址，他不会被
网线连接到，如果你对这个很感兴趣可以看看这个文章
[service proxy](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies)

## 访问Service
Kubernetes支持两种基本的服务发现模式 - 环境变量和DNS。 前者是可以立即使用的，而后者你的安装Kubernetes的集群附加插件。

### 环境变量
当Pod在一个Node上运行时，kubelet会给所有的可用的Service添加环境变量。 这会带来顺序问题，想找到是什么请审查你的工作的nginx pod
```
kubectl exec my-nginx-3800581812-jr2a3 -- printenv | grep SERVICE
```
注意这里根本没有提示Pod所支持的Service。 这是因为创建Service之前我们先创建了Pod的副本。还有个不利的影响是调度器会把这两个Pods都创建在同一节点上
，这样的话如果节点挂了整个服务就挂了。要让这一切正常我们可以把这两个Pods删除掉然后让Deployment重新去创建他们。这样一样Service上就会有两个后面
创建的副本了。这会让Service在调度器级别分发Pods（假设所有的Node都有相同的能力）还能添加正确的环境变量。
```
kubectl scale deployment my-nginx --replicas=0; kubectl scale deployment my-nginx --replicas=2;
kubectl get pods -l run=my-nginx -o wide
```

### DNS 方式
Kubernetes以集群添加项方式提供了DNS服务，使用了skydns自动的dns名称指定给其他的Services对象。通过这个kubectl命令来查看这个添加项是否被安装和启用
```
kubectl get service kube-dns --namespace=kube-system
```
如果没有使用这个添加项，可以通过这个文章来安装和使用[enable it](http://releases.k8s.io/master/cluster/addons/dns/README.md#how-do-i-configure-it)
接下来的章节我们假设你已经有了一个IP地址不发生改变的服务(my-nginx), 以及在DNS服务器中有了这个IP的记录，所以在集群中的任何一个pod上都可以使用标准
的方法来访问这个nginx服务。我们使用一个curl应用来测试这个场景：
```
kubectl run curl --image=radial/busyboxplus:curl -i --tty

nslookup my-nginx
```

## 提高服务的安全性
到现在为止我们只是在集群内部访问了nginx服务。 在暴露这个服务到互联网上之前，你一定希望在一个安全的隧道中去访问他， 为此你需要以下的工作：
* https需要的自签名证书
* nginx服务被配置成使用了这个证书
* 存在一个secret能使Pods使用这个证书

你可以从这里获得一个现成的，这需要安装了go和make工具。如果不想这么麻烦，可以自己亲自体验一下。简言之：
```
make keys secret KEY=/tmp/nginx.key CERT=/tmp/nginx.crt  SECRET=/tmp/secret.json
kubectl create -f /tmp/secret.json
kubectl get secrets
```
如果执行make命令出错，可以使用一下的命令
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /d/tmp/nginx.key -out /d/tmp/nginx.crt  -subj "/CN=my-nginx/o=my-nginx"
cat /d/tmp/nginx.crt  | base64
cat /d/tmp/nginx.key  | base64
```

然后使用上面命令的输出来创建一个yaml文件， 注意base64的数据不能有回车换行
```
apiVersion: v1
kind: secret
metadata:
  name: nginxsecret
  namespace: default
data:
  nginx.crt: ... ...
  nginx.key: ... ...
```
然后使用如下命令去创建一个Secrets
```
kubectl create -f nginxsecrets.yaml
kubectl get secrets
```

现在更新你的nginx的副本以secret的方式来提供https服务。
```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
----
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      volumes:
      - name: secret-volume
        secretName: nginxsecret
      containers:
      - name: nginxhttps
        image: bprashanth/nginxhttps:1.0
        port:
        - containerPort: 443
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/nginx/ssl
          name: secret-volume
```
在这个nginx的应用里有以下值得注意的点：
* 我们把Deployment和Service描述规范放在了一个yaml文件里面
* nginx服务器在80端口提供http服务，在443端口提供启用了安全机制的https服务，nginx服务把他们同时开放了
* 每一个Container通过挂载磁盘的方式获得这个用户私钥。 所以这一步必须要在nginx 服务器容器启动前准备好。
```
kubectl delete deployments, svc my-nginx; kubectl create -f ./nginx-secure-app.yaml
```

然后你就可以在任何一个节点上访问nginx服务器了，
```
kubectl get pods -o yaml | grep -i podip
podIP: 10.224.3.5

node$ curl -k https://10.224.3.5
...
<h1> Welcome to nginx!</h1>
```

> 注意在curl命令中使用了‘-k'参数，是因为在证书的生成期间我们并不知道有关运行nginx的Pods信息，所以我们的
告诉curl命令忽略CName不匹配的错误。 通过在Service查找时被Pod真正的DNS名称我们创建一个我们关联了在证书中
的CName的服务。这一点我们可以从一个pod中简单的测试到（简单起见，我们重用上一个secret，这里pod只是使用
nginx.crt证书文件来访问服务）
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-deployment
spec:
  selector:
    matchLabels:
      app: curlpod
    replicas: 1
    template:
      metadata:
        labels:
          app: curlpod
      spec:
        volumes:
          - name: secret-volume
            secret:
              secretName: nginxsecret
        containers:
        - name: curlpod
          command:
          - sh
          - -c
          - while true: do sleep 1; done
          image: radial/busyboxplus:curl
          volumeMounts:
          - mountPath: /etc/nginx/ssl
            name: secret-volume

kubectl create -f ./curlpod.yaml
kubectl get pods -l app=curlpod
kubectl exec curl-deployment-1515-44  -- curl https://my-nginx --cacert /etc/nginx/ssl/ssl/nginx.crt
...
<h1>Welcome to nginx!</h1>
...            
```

### 对外开放服务
有些情况下你可能要在外部IP地址上开放你的服务。Kubernets 支持两种方式： NodePorts和LoadBalances。 上一节
中我们创建的服务已经使用了NodePorts的方式，所以如果你的节点能在公网上被访问那么你的https副本已经接受来
自internet的流量。
```
kubectl get svc my-nginx -o yaml | grep nodePort -C 5
spec:
  clusterIP: 10.0.162.149
  ports:
  - name: http
    nodePort: 31704
    port: 8080
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 32453
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    run: my-nginx
    
    
kubectl get nodes -o yaml | grep ExternalIP -C  1
  - address: 104.197.41.11
```

我们在试试LoadBalancer方式， 只要更改Service的Type属性为LoadBalancer就好了


















































