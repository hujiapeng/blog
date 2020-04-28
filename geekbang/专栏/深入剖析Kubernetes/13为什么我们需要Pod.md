一、回顾
1. Pod是Kubernetes项目中最小的API对象，或者说是Kubernetes项目中的原子调度单位
2. Docker容器本质：Namespace做隔离，Cgroups做限制，rootfs做文件系统
3. 容器的本质是进程，镜像类似安装包，那么Kubernetes就相当于操作系统

二、为什么需要Pod
1. 使用指令pstree -gp查看Linux进程树，以Linux操作系统日志处理进程rsyslogd为例
```
···
 ├─rsyslogd(18275,18275)─┬─{rsyslogd}(18277,18275)
             │                       └─{rsyslogd}(18278,18275)
···
```
rsyslogd对应一个主进程(main)，和两个子进程(imklog、imuxsock)，同属于18275进程组，这些进程间相互协作完成rsyslogd的职责。注意，Linux中的线程是一种轻量级进程，可以共享文件、信号、数据内存、甚至部分代码
2. 将上面rsyslogd对应的三个进程进行容器化，由于容器是"单进程模型"(容器里并不是只能运行一个"进程"，而是容器只能管理一个进程)，所以把三个进程做成三个容器，并设置容器内存配额都是1GB。这三个容器必须允许在同一台物理机上，否则它们之间基于Socket的通信和文件交换会出现问题。
3. 假设集群上有两个节点，node1内存3GB，node2内存2.5GB。假设用Docker Swarm部署容器，为了让三个容器运行在同一台机器上，就要在另外两个容器上设置一个affinity=main(与main容器有亲密性)的约束，即：它们俩必须和main容器运行在同一台机器上。顺序执行，docker run main,docker run imklog,docker run imuxsock。
4. 很有可能前两个容器运行在node2上，导致剩余内存0.5GB，而第三个容器必须要和前两个运行在同一台机器上，但内存又不够用，这就是典型的成组调度没有妥善处理的例子。资源囤积(等Affinity约束的任务都到达再统一调度)带来了不可避免的调度损失和死锁可能性；乐观调度(如果有问题了，通过回滚机制处理)复杂程度不是常规技术团队所能驾驭的
5. Kubernetes中，Pod是原子调度单位，Kubernentes的调度器是按照Pod而非容器资源需求进行计算。main、imlog和imuxsock三个容器组成一个Pod，Kubernetes调度时，直接可以选择内存3GB的node1节点进行绑定
6. 如上的三个容器间的紧密协作，称为"超亲密关系"，典型特征包括但不限于：互相之间会发生直接的文件交换、会发生非常频繁的远程调用、需要共享某些Linux Namespace(比如，一个容器要加入另一个容器的Network Namespace)等等。并不是所有有"关系"的容器都属于一个Pod，如PHP应用容器和MySQL，虽然会发生访问关系，但没必要也不应该部署在同一台机器上，它们更适合做成两个Pod。

三、Pod实现原理
1. Pod只是一个逻辑概念，Kubernetes真正处理的还是宿主机操作系统上Linux容器的Namespace和Cgroups，而并不是一个什么Pod边界或隔离环境
2. Pod其实是一组共享了某些资源的容器。具体的说，Pod的里的所有容器共享的是同一个Network Namespace，并且可以声明共享同一个Volume。
3. 假设有两个容器A和B，如果使用docker实现两个容器网络和Volume共享，可以使用类似指令``` docker run --net=B --volumes-from=B --name=A image-A ···· ```，但是这样的话，要先启动容器B，再启动容器A，这样一个Pod里面的容器就不是对等关系，而是拓扑关系了。
4. 所以在Kubernetes项目里，Pod实现需要使用一个中间容器，这个容器叫作Infra容器。在这个Pod中，Infra容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过Join Network Namespace的方式，与Infra容器关联在一起。这样的组织关系如下图，Infra容器是一个占用资源极少的特殊镜像，叫k8s.gcr.io/pause。这个镜像是用汇编语言编写，永远处于"暂停"状态的容器，解压后大小也就100~200KB左右
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/geekbang/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90Kubernetes/Pod%E4%B8%AD%E5%AE%B9%E5%99%A8%E4%B8%8EInfra%E5%AE%B9%E5%99%A8%E5%85%B3%E7%B3%BB%E5%9B%BE.jpg)
5. Infra容器"Holder住""Network Namespace"后，用户容器就可以加入到Infra容器的Network Namespace当中了。如果查看这些容器在宿主机上的Namespace文件，它们指向的值一定是完全一样的。
6. 对于Pod里的容器A和容器B来说
1）、它们可以直接使用localhost进行通信
2）、它们看到的网络设备跟Infra容器看到的完全一样
3）、一个Pod只有一个IP地址，也就是这个Pod的Network Namespace对应的IP地址
4）、当然其他的所有网络资源都是一个Pod一份，并且被该Pod中的所有容器共享
5）、Pod的生命周期只跟Infra容器一致，而与容器A和容器B无关
6）、对于同一个Pod里的所有用户容器来说，它们的进出流量都是通过Infra容器完成的。所以注意，如果要为Kubernetes开发一个网络插件，要考虑如何配置这个Pod的Network Namespace(Infra容器的Network Namespace)
7. 一个Volume对应的宿主机目录对于Pod来说就只有一个，Pod里的容器只要声明挂载这个Volume，就一定可以共享这个Volume对应的宿主机目录，如下案例
```
apiVersion: v1
kind: Pod
metadata:
     name: two-containers
spec:
     restartPolicy: Never
     volumes:
     - name: shared-data
       hostPath:      
         path: /data
     containers:
     - name: nginx-container
       image: nginx
       volumeMounts:
       - name: shared-data
         mountPath: /usr/share/nginx/html
     - name: debian-container
       image: debian
       volumeMounts:
       - name: shared-data
         mountPath: /pod-data
       command: ["/bin/sh"]
       args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```
例子中，debian-container和nginx-container都声明挂载了shared-data这个Volume。而shared-data是hostPath类型。所以，它对应在宿主机上的目录就是/data。而这个目录，其实就被同时绑定挂载进了上述两个容器中。所以在debian-container生成的index.html文件，在nginx-container容器的/usr/share/nginx/html目录中也能找到

四、容器设计模式
1. 典型案例1：WAR包与Web服务器关系，现在有个Java Web应用的WAR包，它需要被放到Tomcat的webapps目录下运行起来
1）、如果只用Docker来处理这个组合关系：一种方法是把WAR包直接放到Tomcat镜像的webapps目录下，做成一个新镜像运行起来，这样每次更新WAR包或升级Tomcat都要重新做镜像，很麻烦；另一种方法是，只发布一个Tomcat容器，而这个容器的webapps目录就必须声明一个hostPath类型的Volume，从而把宿主机上的WAR包挂载进Tomcat容器中运行起来，不过这样要解决如何让每台宿主机都预先准备好存有这个WAR包的目录，如此要搞一个分布式存储系统，也比较麻烦
2）、使用容器设计模式，把WAR包和Tomcat分别做成镜像，然后把它们作为一个Pod里的两个容器"组合"在一起。Pod配置如下
```
apiVersion: v1
kind: Pod
metadata:
     name: javaweb-2
spec:
     initContainers:
     - image: geektime/sample:v2
       name: war
       command: ["cp", "/sample.war", "/app"]
       volumeMounts:
       - mountPath: /app
         name: app-volume
     containers:
     - image: geektime/tomcat:7.0
       name: tomcat
       command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
       volumeMounts:
       - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
         name: app-volume
       ports:
       - containerPort: 8080
         hostPort: 8001 
     volumes:
     - name: app-volume
       emptyDir: {}
```
Pod配置中有两个容器，第一个是Init Container类型的容器。在Pod中，所有Init Container定义的容器，都会比spec.containers定义的容器先启动，并且Init Container容器按顺序启动，直到都启动完成，spec.containers定义的容器才会启动。所以，Init Container类型的WAR包容器，启动后执行了一句"cp /sample.war /app"，在容器中把应用的WAR包拷贝到/app目录下，然后退出。这个WAR包容器的/app目录挂载了一个叫app-volume的Volume，而Tomcat容器的/root/apache-tomcat-7.0.42-v2/webapps目录也挂载到一个叫app-volume的Volume。两容器共享这个app-volume的Volume容器。所以等Tomcat容器启动后，它的webapps目录下就会有sample.war文件。
3）、如上的"组合"方式就解决了WAR包与Tomcat容器之间的耦合关系问题，这种"组合"操作正是容器设计模式里常用的一种设计模式，叫sidecar。sidecar指的就是我们可以在一个Pod中，启动一个辅助容器，来完成一些独立于主进程(主容器)之外的工作。
2. 典型案例2：容器日志收集
1）、现在有一个应用，需要不断地把日志文件传输到容器的/var/log目录上
2）、这时就把Pod里的一个Volume挂载到应用容器的/var/log目录上
3）、然后在这个Pod里同时运行一个sizecar容器，它也声明挂载这个Pod里同一个Volume到自己的/var/log目录上
4）、如此，两个容器共享Volume，应用容器生成的日志文件在sidecar容器里也能获取到，这样，接下来，sidecar容器就可以按照自己的逻辑处理日志了。
5）、Istio这个微服务治理项目原理就是使用sidecar容器完成的

市面上的小镜像，如busybox、alpine，适合用来做baseimage
