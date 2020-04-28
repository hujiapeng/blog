目前，各大云厂商最常用的部署方法，是使用SaltStack、Ansible等运维工具自动化地执行这些步骤
一、kubeadm的工作原理
1. Kubernetes在部署时，它的每一个组件都是一个需要被执行的、单独的二进制文件。SaltStack这样的运维工具或者由社区维护的脚本的功能，就是要把这些二进制文件传输到指定的机器中，然后编写控制脚本来启停这些组件
2. 没有把Kubernetes容器化的主要原因是kubelet在配置容器网络、管理容器数据卷时都需要直接操作宿主机。如果kubelet本身就运行在一个容器里，那么直接操作宿主机就会变得很麻烦
3. kubeadm妥协方案：把kubelet直接运行在宿主机上，然后使用容器部署其他的Kubernetes组件
4. kubeadm的作者已经为各个发行版Linux准备好了安装包，所以安装只需执行
```
apt-get install kubeadm
```
然后，就可以使用'kubeadm init'部署Master节点了。

二、kubeadm init的工作流程(主要四步，检查 1、生成证书 2~7、生成配置文件 8~15、安装默认插件16)
1. 执行kubeadm init指令后，kubeadm先执行检查工作(Preflight Checks)，确保可以部署Kubernetes
2. Kubernetes对外提供服务时，除非开启"不安全模式"，否则都要通过HTTPS才能访问kube-apiserver。这就需要为Kubernetes集群配置好证书文件
3. kubeadm为Kubernetes项目生成的证书文件都在Master节点的/etc/kubernetes/pki目录下。这个目录主要证书文件是ca.crt和对应的私钥ca.key
4. 用户使用kubectl获取容器日志等streaming操作时，需要通过kube-apiserver向kubelet发起请求，这个连接也必须是安全的。kubeadm为这一步生成的是apiserver-kubelet-client.crt文件，对应的私钥是apiserver-kubelet-client.key
5. 除此之外，Kubernetes集群中还有Aggregate APIServer等特性，也需要专门的证书
6. 如果不想让kubeadm生成证书，就需要拷贝现有证书到如下证书目录，如此kubeadm就会跳过证书生成步骤
```
/etc/kubernetes/pki/ca.{crt,key}
```
7. 证书生成后，kubeadm接下来会为其他组件生成访问kube-apiserver所需的配置文件。这些文件路径是``` /etc/kubernetes/xxx.conf ```。这些文件里记录的是，当前这个Master节点的服务器地址、监听端口、证书目录等信息。这样，对应的客户端(比如scheduler，kubelet等)，可以直接加载相应的文件，使用里面的信息与kube-apiserver建立安全连接
8. 接下来，kubeadm会为Master组件生成Pod配置文件。kube-apiserver、kube-controller-manager、kube-scheduler这三个组件都会以Pod方式部署起来。此时Kubernetes集群还不存在，kubeadm不会直接运行docker run来启动这些容器。而是通过特殊的容器启动方法"Static Pod"
9. "Static Pod"特殊容器启动方法，允许把要部署的Pod的YAML文件放到指定目录下，这样在这台机器上启动kubelet时，会自动检查这个目录，加载目录下所有Pod YAML文件，然后在这台机器上启动它们。由此看出kubelet在Kubernetes项目中地位很高，在设计上它就是完全独立的组件，而其他Master组件更像辅助性的系统容器
10. 在kubeadm中，Master组件的YAML文件会被生成在/etc/kubernetes/manifests路径下
11. kubeadm还会再生成一个Etcd的Pod YAML文件，通过同样的Static Pod方式启动Etcd。
12. 最后Master组件的Pod YAML文件如下
```
[root@ittutu ~]# ls /etc/kubernetes/manifests/
etcd.yaml kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml
```
一旦这些YAML文件出现在被kubelet监视的/etc/kubernetes/manifests目录下，kubelet就会自动创建这些YAML文件中定义的Pod
13. Master容器启动后，kubeadm会通过检查localhost:6443/healthz这个Master组件的健康检查URL，等待Master组件完全运行起来
14. kubeadm会为集群生成一个bootstrap token，这个token值及使用方法会在kubeadm init结束后打印出来。在后面，只要持有这个token，任何一个安装了kubelet和kubeadm的节点，都可以通过kubeadm join加入到这个集群当中
15. 在token生成之后，kubeadm会将ca.crt等Master节点的重要信息，通过ConfigMap的方式保存在Etcd当中，供后续部署Node节点使用。这个ConfigMap的名字是cluster-info
16. kubeadm init最后一步就是安装默认插件。Kubernetes默认kube-proxy和DNS这两个插件是必须安装的。它们分别用来提供整个集群的服务发现和DNS功能。其实这两个插件也只是两个容器镜像而已，所以kubeadm只要用Kubernetes客户端创建两个Pod就可以了

三、kubeadm join的工作流程
1. kubeadm join需要token原因：任何想要成为Kubernetes集群中的一个节点，就必须在集群的kube-apiserver上注册。和apiserver打交道就需要获取到相应的证书文件(CA文件)。为了能够一键安装，不让用户自己去Master节点上手动拷贝这些文件，所以需要kubeadm至少发起一次"不安全模式"的访问到kube-apiserver，从而拿到保存在ConfigMap中的cluster-info(它保存了APIServer的授权信息)。而bootstrap token在这个过程中扮演安全验证角色。
2. 只要有了cluster-info里的kube-apiserver的地址、端口、证书，kubelet就可以以"安全模式"连接到apiserver上，这样一个新的节点就部署完成了

四、配置kubeadm的部署参数
1. 除了直接修改上面提到的配置文件，强烈建议使用下面方式
```
[root@ittutu ~]# kubeadm init --config kubeadm.yaml
```
这时通过提供一个YAML文件方式，来自定义部署参数。文件样式如下，kind是MasterConfiguration
```
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.0
api:
  advertiseAddress: 192.168.0.102
  bindPort: 6443
  ...
etcd:
  local:
    dataDir: /var/lib/etcd
    image: ""
imageRepository: k8s.gcr.io
kubeProxy:
  config:
    bindAddress: 0.0.0.0
    ...
kubeletConfiguration:
  baseConfig:
    address: 0.0.0.0
    ...
networking:
  dnsDomain: cluster.local
  podSubnet: ""
  serviceSubnet: 10.96.0.0/12
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  ...
```
2. 比如，要指定kube-apiserver的参数，只要在如上文件加上这样一段信息
```
...
apiServerExtraArgs:
  advertise-address: 192.168.0.103
  anonymous-auth: false
  enable-admission-plugins: AlwaysPullImages,DefaultStorageClass
  audit-log-path: /home/johndoe/audit.log
```
然后，kubeadm就会用上面这些信息替换/etc/kubernetes/manifests/kube-apiserver.yaml里的command字段里的参数了

五、kubeadm的源码在kubernetes/cmd/kubeadm目录下，是Kubernetes项目的一部分。其中app/phases文件夹下的代码，对应的就是这篇文章中详细介绍的每一个具体步骤
六、kubeadm目前(2018年9月)不能用于生产环境，最欠缺的是，一键部署一个高可用的Kubernetes集群，即：Etcd、Master组件都应该是多节点集群，而不是现在这样单节点。部署生成环境需求，推荐使用kops或者SaltStack这样的工具