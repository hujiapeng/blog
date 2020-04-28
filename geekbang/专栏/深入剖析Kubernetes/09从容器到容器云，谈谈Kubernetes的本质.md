一、回顾容器
1. 一个"容器"，实际上是一个由Linux Namespace、Linux Cgroups和rootfs三种技术构建出来的进程的隔离环境
2. 一个正在运行着的Linux容器，可以被"一分为二"地看待
(1)、一组联合挂载在/var/lib/docker/aufs/mnt上的rootfs，这一部分我们称为"容器镜像"(Container Image)，是容器的静态视图
(2)、一个由Namespace+Cgroups构成的隔离环境，这一部分我们称为"容器运行时"(Container Runtime)，是容器的动态视图

二、Kubernetes架构
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/geekbang/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90Kubernetes/Kubernetes%E6%9E%B6%E6%9E%84%E5%9B%BE.jpg)
1. 由Master和Node两种节点组成，这两种角色分别对应着控制节点和计算节点
2. Master节点，由三个紧密协作的独立组件组合而成，它们分别是负责API服务的kube-apiserver、负责调度的kube-shecduler，以及负责容器编排的kube-controller-manager。整个集群的持久化数据，则由kube-apiserver处理后保存在Etcd中
3. 计算节点上最核心的部分是kubelet组件。在Kubernetes项目中，kubelet主要负责同容器运行时(比如Docker项目)打交道。而这个交互依赖的是CRI(Container Runtime Interface)远程调用接口，这个接口定义了容器运行时的各项核心操作，如启动一个容器需要的所有参数。标准的容器镜像，不管什么容器运行时、什么技术实现，都可以通过实现CRI接入到Kubernetes项目中。
4. 具体的容器运行时，如Docker项目，则一般通过OCI这个容器运行时规范同底层的Linux操作系统进行交互，即：把CRI请求翻译成对Linux操作系统的调用(操作Linux Namespace和Cgroups等)
5. kubelet还通过gRPC协议同一个叫做Device Plugin的插件进行交互。这个插件，是Kubernetes用来管理GPU等宿主机物理设备的主要组件，也是基于Kubernetes项目进行机器学习训练、高性能作业支持等工作必须关注的功能
6. kubelet的另一个重要功能，是调用网络插件和存储插件为容器配置网络和持久化存储。这两个插件与kubelet进行交互的接口，分别是CNI(Container Networking Interface)和CSI(Container Storage Interface)

三、平台项目间容器关系与形态
1. 常规环境下，多个相关应用部署在同一台机器上，通过Localhost通信，通过本地磁盘目录交换文件。而在Kubernetes项目中，这些容器则会被划分为一个"Pod"，Pod里的容器共享同一个Network Namespace、同一组数据卷，来达到高效率交换信息的目的
2. 常见需求，Web应用与数据库之间的访问关系。两个应用一般不部署在同一台机器上，这样某台宕机，另一个不受影响。但是对于容器来说，IP地址等信息不是固定的。Kubernetes项目的做法是给Pod绑定一个Service服务，而Service服务声明的IP地址等信息是"终生不变"的。这个Service服务的主要作用，就是作为Pod的代理入口(Portal)，从而代替Pod对外暴露一个固定的网络地址。这样对于Web应用的Pod来说，它需要关心的就是数据库Pod的Service信息。Service后端真正代理的Pod的IP地址、端口等信息的自动更新、维护，则是Kubernetes项目的职责。
3. Kubernetes项目核心功能图
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/geekbang/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90Kubernetes/Kubernetes%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD%E5%9B%BE.jpg)
为解决容器(Container)间"紧密协作"关系，扩展到了Pod；有了Pod之后，希望能一次启动多个应用实例，这样就需要Deployment这个Pod的多实例管理器；而有了这样一组相同的Pod后，我们又需要通过一个固定的IP地址和端口以负载均衡的方式访问它，于是就有了Service
4. 如果两个不同的Pod之间不仅有"访问关系"，还要求在发起时加上授权信息，如Web应用访问数据库时需要Credential(数据库的用户名和密码)信息。这种情况下，Kubernetes项目提供了一种叫作Secret的对象，它其实是一个保存在Etcd里面的键值对数据。这样，你把Credential信息以Secret的方式存在Etcd里，Kubernetes就会在你指定的Pod(如，Web应用的Pod)启动时，自动把Secret里的数据以Volume的方式挂载到容器里。这样Web应用就可以访问数据库了
5. 应用运行的形态是影响"如何容器化这个应用"的第二个重要因素。Kubernetes定义了新的、基于Pod改进后的对象。如Job，用来描述一次性运行的Pod(如，大数据任务)；再比如DaemonSet，用来描述每个宿主机上必须且只能运行一个副本的守护进程服务；又比如CronJob，用于描述定时任务等

四、Kubernetes核心设计理念，声明式API，这种API对应的"编排对象"和"服务对象"，都是Kubernetes项目中的API对象(API Object)
1. 首先通过一个"编排对象"，比如Pod、Job、CronJob等，来描述试图管理的应用；
2. 然后再为它定义一些"服务对象"，比如Service、Secret、Horizontal Pod Autoscaler(自动水平扩展器)等。这些对象会负责平台级功能