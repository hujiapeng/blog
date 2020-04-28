Kubernetes项目API对象上，哪些属性属于Pod对象，哪些属性属于Container呢？
如果，把Pod看出传统环境里的"机器"，把容器看作是运行在这个"机器"里的"用户程序"，就比较好理解了。
凡是调度、网络、存储，以及安全相关的属性，基本上是Pod级别的。这些属性的共同特征是描述的是"机器"这个整体，而不是里面运行的"程序"。
一、Pod中几个重要字段的含义和用法
1. NodeSelector：是一个供用户将Pod与Node进行绑定的字段，用法如下
```
apiVersion: v1
kind: Pod
······
spec:
     nodeSelector:
       disktype: ssd
```
这样的一个配置，意味着这个Pod永远只能运行在携带了"disktype: ssd"标签的节点上，否则调度失败。
2. NodeName：一旦Pod的这个字段被赋值，Kubernetes项目就会被认为这个Pod已经经过了调度，调度结果就是赋值的节点名字。所以这个字段一般有调度器负责设置，但用户也可以设置它来"骗过"调度器，这个方法一般在测试或者调试时候用到。
3. HostAliases：定义了Pod的hosts文件(如/etc/hosts)里的内容，用法如下
```
apiVersion: v1
kind: Pod
······
spec:
     hostAliases:
     - ip: "10.1.2.3"
       hostnames:
       - "foo.remote"
       - "bar.remote"
······
```
在这个Pod的YAML文件中，设置了一组IP和hostname的数据。这样，这个Pod启动后，/etc/hosts文件的内容如下：
```
cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```
其中最下面两行，就是通过HostAliases字段为Pod设置的。注意，在Kubernetes项目中，如果要设置hosts文件里的内容，一定要通过这种方法。否则，如果直接修改了hosts文件的话，在Pod被删除重建后，kubelet会自动覆盖掉被修改的内容

二、凡是跟容器的Linux Namespace相关的属性，也一定是Pod级别的。因为Pod的设计就是让它里面的容器尽可能的共享Linux Namespace，仅保留必要的隔离和限制能力
1. 如下Pod的YAML文件，定义了shareProcessNamespace=true，意味着这个Pod里的容器共享PID Namespace。即，容器中的进程可以互相看到
```
apiVersion: v1
kind: Pod
metadata:
     name: nginx-shell
spec:
     shareProcessNamespace: true
     containers:
     - name: nginx
       image: nginx
     - name: shell
       image: busybox
       stdin: true
       tty: true
```
YAML中定义了两个容器，一个是nginx容器，一个是开启了tty和stdin的shell容器。如下命令创建Pod
```
kubectl create -f nginx-shell.yaml
```
使用kubectl attach命令连接到shell容器的tty上
```
kubectl attach -it nginx-shell -c shell
```
在shell容器里执行ps指令，查看运行中的进程
```
$ kubectl attach -it nginx -c shell
/ # ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   14 101       0:00 nginx: worker process
   15 root      0:00 sh
   21 root      0:00 ps ax
```
可以看到，这个容器里，不仅可以看到本身的ps ax指令，还可以看到nginx容器的进程，以及Infra容器的/pause进程。这就意味着，整个Pod里的每个容器进程，对所有容器是可见的：它们共享了同一个PID Namespace。注意Kubernetes项目1.11后，apiserver配置中--feature-gates的PodShareProcessNamespace默认为true，之前都为false。本机环境PodShareProcessNamespace默认为false，所以shareProcessNamespace为true不起作用，shell容器里看不到其他容器进程。

三、凡是Pod中的容器要共享宿主机的Namespace，也一定是Pod级别的定义
1. 如下Pod的YAML文件，定义了共享宿主机的Network、IPC和PID Namespace。这就意味着，这个Pod里所有容器，会直接使用宿主机的网络、直接与宿主机进行IPC通信、看到宿主机里正在运行的所有进程
```
apiVersion: v1
kind: Pod
metadata:
     name: nginx
spec:
     hostNetwork: true
     hostIPC: true
     hostPID: true
     containers:
     - name: nginx
       image: nginx
     - name: shell
       image: busybox
       stdin: true
       tty: true
```


