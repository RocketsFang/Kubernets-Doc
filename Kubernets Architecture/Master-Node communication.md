# 主节点和节点间的通信
## 概括
这篇说明是关于主节点（实际是APIServer）和集群之间的通信。 目的是帮助用户去自定义他们的安装来加强网络配置进而让集群能够运行到不可信的网络上(或者是完全
公开的公网IP上)。

## 集群 -> 主节点
所有从集群到主节点的通信路线都是终结于APIServer(主节点上再没有其他的模块是被设计用来提供远程服务的)。 在一个典型的部署方式中，APIServer是被配置为在
HTTPS 443端口上监听远程以任何方式进行身份认证的连接请求。多种方式认证方式应该被启用，特别是允许匿名连接请求和服务帐号令牌的方式。

在创建节点时应当一起创建共有根用户的证书以备和APIServer建立安全联接时使用。比如：在默认的GCE集群部署过程中，客户端的凭证以客户端证书的方式提供给kubelet
参阅 kubelet TLS 启动过程中自动创建kubelet证书。

利用服务帐号令牌的方式Pod可以建立和APIServer之间安全的连接，这样Kubernetes将自动拒绝公用根账号证书和有效的裸机令牌在Pod实例化的时候注入到Pod中。

所有命名空间中的Kubernetes服务都被配置成使用虚拟IP地址，而这个IP地址是被重定向到APIServer上的HTTPS终端的。

其他的主模块也是在安全端口上和集群APIServer建立连接的。

总上所述，默认的从集群到主节点通信操作模型是安全的是有能力运行在不可信的或者公共网络上的。

## 主节点 -> 集群
这种方向的通信一共有两种主要的路线：
1， 从APIServer到kubelet进程的通信
2， 通过apiserver的代理功能从APIServer到任何一个node，pod或者service

### APIServer -> kubelet
这种通信从APIServer到kubelet主要用于：
* 为Pods抓取日志信息
* 附加正在运行的Pods
* 提供kubelet的port-forwarding功能
这种通信终结于kubelet的HTTPS终端。 默认APIServer不会去验证kubelet的服务证书，这就会有中间人攻击的可能性发生，所以他不建议被部署在不安全的或公共网络
上。
要使用验证功能， 使用--kubelet-certificate-authority参数来提供跟用户证书，这样就能验证kubelet的服务证书了。
如果没有可能性， 但是又要求运行在一些不可行的网络上，那我们可以在APIServer和kubelet之间使用SSH tunneling
最后，kubelet的认证和授权就可以被启用了，也就加固了kubelet API

### APIServer -> nodes, pods和services
APIServer到node，pod或service的通信默认是扁平的HTTP连接，因此既不是经过认证的也不是加密的。 虽然他们之间可以通过HTTPS协议建立安全的连接，他们也不会
去验证HTTPS端点的证书也不提供客户端认证，所以即使连接是加密的他们也不能保证通信信息的完整性。 这些连接是不能用于不可信认的网络或者公共网络的。

### SSH Tunnels
Google Kubernetes Engine 使用SSH隧道来保护主节点到集群的通信线路。在他的配置中APIServer会初始化到集群中每一个节点的SSH通行隧道(使用的是SSH服务器
的22端口)然后用这个隧道和所有的node，pod，service通信。 这个隧道能保证在这个运行有集群的私有流量不会被暴露到外部去。











