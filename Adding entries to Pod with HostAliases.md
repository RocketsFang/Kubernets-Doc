# 为Pod机器的/etc/hosts文件添加HostAliases条目
当DNS及其他的域名解析服务不工作的时候我们提供Pod级别的域名解析，具体是通过在Pods机器/etc/hosts文件添加条目。在1.7版本后用户可以在PodSpec描述规范中
添加HostAliases用户自定义的条目。
不使用HostAliases做修改是不被建议的，因为这个文件是被Kubelete管理的有可能在Pod的重建或者重启过成功被重写。

## 默认的Hosts文件内容
我们从一个被指定IP地址的nginx pod开始
```
kubectl run nginx --image nginx --generator=run-pod/v1

kubectl get pods --output=wide
```
这个/etc/hosts文件目前是这样的
```
kubectl exec nginx -- cat /etc/hosts
127.0.0.1      localhost
::1    localhost ip6-localhost ip6-loopback
fe00:0 ip6-localnet
fe00:0 op6-mcastprefix
10.200.0.4   nginx
```
默认，这个hosts文件只有ipv4和ipv6的模板内容比如localhost和他的主机名

## 通过HostAliases添加额外的条目
除了默认的模板，我们可以添加额外的条目到hosts文件中来解析foo.local, bar.local到127.0.0.1 以及， foo.remote, bar.remote 到10.1.2.3 ，使用.spec
.hostAliases来完成
```
apiVersion: v1
kind: Pod
metadata:
  name: hostaliases-pod
spec:
  restartPolicy: Never
  hostAliases:
  - ip: "127.0.0.1"
    hostname:
    - "foo.local"
    - "bar.local"
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
  containers:
  - name: cat-hosts
    image: busybox
    command:
    - cat
    args:
    - "/etc/hosts"
```
通过以下的命令来启动这个Pod
```
kubectl apply -f hostaliases-pod.yaml

kubectl get pod -o=wide
```

这时候/etc/hosts文件内容则变成
```
kubectl logs hostaliases-pod
127.0.0.1   localhost
::1   localhost ip6-localhost ip6-loopback
fe00::0 ip-localhost
10.244.135.10 hostaliases-pod
127.0.0.1  foo.local
127.0.0.1 bar.local
10.1.2.3    foo.remote
10.1.2.3     bar.remote
```
我们期望的条目被添加到了文件的底部

## 局限
HostAlias只在Kubernetes1.7以上（包含）被支持。 只有在non-hostNetwork网络模型中被支持因为kubelet只有在non-hostNetwork网络模型的Pods中才会接管hosts
文件。
在Kubernetes1.8以上（包含），HostAlias在任何一个网络模型中被所有的Pods支持。

## 为什么Kubelet回去管理Hosts文件
kubelet为Pod中的每一个容器管理hosts文件，因为Docker在重启时会修改这个文件。由于文件的管理本性，任何用户修改的内容都有可能由于容器的重启或者Pod的重新
重新规划被覆盖不管这个hosts文件被kubelet挂载到哪里，因此不建议去修改这个文件。















