一、通过Taint/Toleration调整Master执行Pod的策略
1. 默认情况下Master节点是不允许运行用户Pod的。Kubernetes做到这一点依靠的是Kubernetes的Taint/Toleration机制
2. 原理：一旦某个节点被加上了Taint，即被"打上了污点"，那么所有Pod就都不能在这个节点运行，因为Kubernetes的Pod都有"洁癖"。除非，有个别的Pod声明自己能"容忍"这个"污点"，即声明了Toleration，它才可以在这个节点上运行。总结下，就是如果节点上设置了规则，那么只有符合这个规则的Pod才能运行。
3. 为节点打上"污点"(Taint)命令如下
```
kubectl taint nodes node1 foo=bar:NoSchedule
```
这时，这个node1节点上就会增加一个键值对格式的Taint，即：foo=bar:NoSchedule。其中值里面的NoSchedule表示这个Taint只会在调度新Pod时产生作用，而不会影响已经在node1上运行的Pod，即打"污点"前的Pod，即使之前的Pod没有声明Toleration
4. 声明Toleration，在Pod的.yaml文件中的spec部分，加入tolerations字段
```
apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
    operator: "Equal"
    value: "bar"
    effect: "NoSchedule"
```
这个Toleration的含义是，这个Pod能"容忍"所有键值对为foo=bar的Taint(operator:"Equal"，"等于"操作)
5. 查看集群中Master节点的Taint字段，Master节点上默认被加上了node-role.kubernetes.io/master:NoSchedule这样一个"污点"，其中键为node-role.kubernetes.io/master，没有值。
```
[root@kubeadm ~]# kubectl describe node kubeadm
Name:               kubeadm
Roles:              master
······
Taints:             node-role.kubernetes.io/master:NoSchedule
······
```
如果想让新Pod在这个Master节点上运行，就需要如下使用"Exists"操作符(operator:"Exists"，"存在"即可)来说明，该Pod能容忍所有以foo为键的Taint
```
apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
    operator: "Exists"
    effect: "NoSchedule
```
6. 如果就想要一个单节点的Kubernetes，那就可以删除这个Taint，删除所有以"node-role.kubernetes.io/master"为键的Taint指令如下，注意键最后有个短横(-)
```
[root@kubeadm ~]# kubectl taint nodes --all node-role.kubernetes.io/master-
```

二、部署Dashboard可视化插件
1. Dashboard项目可以提供一个可视化的Web界面来查看当前集群的各种信息，部署方式如下
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```
由于无法获取到k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1，所以参照了github上[https://github.com/kainonly/kubernetes-dashboard-amd64](https://github.com/kainonly/kubernetes-dashboard-amd64)
2. 查看Dashboard对应的Pod状态
```
[root@kubeadm ~]# kubectl get pods -n kube-system
```
3. 由于Dashboard是一个WebServer，很多人经常无意中在自己的公有云上暴漏Dashboard的端口，造成安全隐患，所以1.7版本后的Dashboard项目部署完后，默认只能通过Proxy方式在本地访问，具体操作参考[https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard)

三、部署容器存储插件
1. 容器无状态特征：在某一台机器上启动的一个容器，无法看到其他机器上的容器在它们的数据卷里写入的文件
2. 容器持久化：用来保存容器存储状态。存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系。这样，无论在其他哪个宿主机上启动新的容器，都可以请求挂载指定的持久化存储卷，从而访问到数据卷里保存的内容
3. Rook项目是一个基于Ceph的Kubernetes存储插件，不同于Ceph的简单封装，Rook实现了水平扩展、迁移、灾备、监控等大量企业级功能。Rook使用容器化技术部署Ceph存储后端。
```
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/common.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml
```
4. 基于Rook持久化存储集群就以容器方式运行起来了，而接下来在Kubernetes项目上创建的所有Pod就能通过Persistent Volume(PV)和Persistent Volume Claim(PVC)的方式，在容器里挂载由Ceph提供的数据卷了

四、其他指令
1. 查看所有pod
```
 kubectl get pods --all-namespaces 
```
2. 查看所有namespace
```
[root@kubeadm ~]# kubectl get namespaces
NAME          STATUS    AGE
default       Active    2d
kube-public   Active    2d
kube-system   Active    2d
rook-ceph     Active    2h
```
3. 查看指定namespace的pod，如果不写，默认是default
```
kubectl get pods -n kube-system
```
4. 查看pod描述信息，如果不带-n，则是default
```
kubectl describe pod etcd-kubeadm -n kube-system
```
5. 查看Pod日志信息，如果不带-n，则是default
```
kubectl logs etcd-kubeadm -n kube-system
```
6.新增或更新Pod
```
kubectl apply -f XXX.yaml
```
7. 删除Pod
```
kubectl delete -f XXX.yaml
```
```
kubectl delete --namespace=kube-system pod kube-dns-86f4d74b45-d2vh6
```
```
 kubectl delete pod kube-proxy-fgtx5 -n kube-system
```
