http://docs.kubernetes.org.cn/618.html
https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-interactive/
注意：一定要修改各系统主机名(/etc/hostname)及hosts(/etc/hosts)映射，如下
```
[root@kubeadm ~]# cat /etc/hosts
127.0.0.1   localhost kubeadm
::1         localhost kubeadm
[root@kubeadm ~]# cat /etc/hostname
kubeadm
```
本地虚拟机环境：CentOS7系统，4G内存，2核CPU，8G硬盘
```
[root@localhost ~]# uname -a
Linux localhost.localdomain 3.10.0-693.el7.x86_64 #1 SMP Tue Aug 22 21:09:27 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```
一、安装kubeadm和Docker
1. 更新yum
```
[root@localhost ~]# yum -y update
```
2. 安装Docker
```
[root@localhost ~]# yum install -y docker
```
3. (此步只是参考，不能翻墙就跳过此步，如果能翻墙就跳过下面配置及安装步骤)官方配置Kubernetes yum源及安装，官方网站([https://v1-11.docs.kubernetes.io/docs/tasks/tools/install-kubeadm/](https://v1-11.docs.kubernetes.io/docs/tasks/tools/install-kubeadm/))
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
```
setenforce 0是设置SELinux为宽容模式(permissive)
SELinux有三种模式，强制模式(enforcing)，代表SELinux运作中且已经正确限制domain/type、宽容模式(permissive)，SELinux运作中但不限制domain/type存取仅警告、关闭(disabled)，停止SELinux运作。```getenforce```命令查看SELinux模式。setenforce [0|1]宽容模式|强制模式切换。如果想永久设置为禁用，需要修改配置文件/etc/selinux/config中SELINUX=disabled，重启系统。
4. 配置阿里云Kubernetes yum源并禁用SELinux
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
```
5. 安装kubectl、kubelet、kubeadm组件的1.11.1版本，注意安装顺序，因为安装kubeadm时会安装依赖的kubectl，而此时靠依赖安装的kubectl是当前库中新版本，不是1.11.1版本，后面会报版本问题。如果报了版本问题，执行yum remove kubectl后，再安装1.11.1版本即可
```
[root@localhost ~]# yum install -y kubectl-1.11.1
[root@localhost ~]# yum install -y kubelet-1.11.1
[root@localhost ~]# yum install -y kubeadm-1.11.1
```
课程中说安装kubeadm过程中，kubeadm和kubelet、kubectl、kubernetes-cni这几个二进制文件都会被自动安装好。确实如此，但是考虑到使用课程中版本问题，不能使用默认安装，就按照如上顺序安装

二、部署Kubernetes的Master节点
1. 可以直接使用kubeadm init命令，使用默认方式配置Master节点，但是此处通过配置文件开启一些实验性功能，kubeadm.yaml文件内容如下
```
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
controllerManagerExtraArgs:
     horizontal-pod-autoscaler-use-rest-clients: "true"
     horizontal-pod-autoscaler-sync-period: "10s"
     node-monitor-grace-period: "10s"
apiServerExtraArgs:
     runtime-config: "api/all=true"
kubernetesVersion: "v1.11.1"
```
配置中，给kube-controller-manager设置了horizontal-pod-autoscaler-use-rest-clients: "true"，意味着将来部署的kube-controller-manager能够使用自定义资源(Custom Metrics)进行自动水平扩展。
配置Master节点命令如下
```
[root@localhost ~]# kubeadm init --config kubeadm.yaml 
```
执行完后看到如下信息
```
[init] using Kubernetes version: v1.11.1
[preflight] running pre-flight checks
        [WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
        [WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
I0708 18:31:54.388258   21889 kernel_validator.go:81] Validating kernel version
······
······
CGROUPS_MEMORY: enabled
        [WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
[preflight] Some fatal errors occurred:
        [ERROR Service-Docker]: docker service is not active, please run 'systemctl start docker.service'
        [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables does not exist
        [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
        [ERROR Swap]: running with swap on is not supported. Please disable swap
        [ERROR SystemVerification]: failed to get docker info: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
```
解决第一个Warning，关闭防火墙
```
[root@localhost ~]# firewall-cmd --state
running
[root@localhost ~]# systemctl stop firewalld
[root@localhost ~]# systemctl mask firewalld
Created symlink from /etc/systemd/system/firewalld.service to /dev/null.
[root@localhost ~]# 
```
解决第二个Warning，执行如下
```
[root@localhost ~]# systemctl enable docker.service
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@localhost ~]# 
```
解决第三个Warning，执行如下
```
[root@localhost ~]# systemctl enable kubelet.service
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /etc/systemd/system/kubelet.service.
[root@localhost ~]# 
```
解决第一个Error
```
[root@localhost ~]# systemctl start docker.service
```
第二、三个Error是在开启防火墙时需要配置的，可以参考官网的设置，关闭防火墙后，再次执行kubeadm init --config kubeadm.yaml，可以看到只有一个Error了。
解决Error [ERROR Swap]
```
[root@localhost ~]# swapoff -a
```
再次执行kubeadm init --config kubeadm.yaml，通过校验，但是在拉取相关镜像失败
```
[root@localhost ~]# kubeadm init --config kubeadm.yaml 
[init] using Kubernetes version: v1.11.1
[preflight] running pre-flight checks
I0708 12:54:58.944945   13421 kernel_validator.go:81] Validating kernel version
I0708 12:54:58.945036   13421 kernel_validator.go:96] Validating kernel config
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[preflight] Some fatal errors occurred:
        [ERROR ImagePull]: failed to pull image [k8s.gcr.io/kube-apiserver-amd64:v1.11.1]: exit status 1
        [ERROR ImagePull]: failed to pull image [k8s.gcr.io/kube-controller-manager-amd64:v1.11.1]: exit status 1
        [ERROR ImagePull]: failed to pull image [k8s.gcr.io/kube-scheduler-amd64:v1.11.1]: exit status 1
        [ERROR ImagePull]: failed to pull image [k8s.gcr.io/kube-proxy-amd64:v1.11.1]: exit status 1
        [ERROR ImagePull]: failed to pull image [k8s.gcr.io/pause:3.1]: exit status 1
        [ERROR ImagePull]: failed to pull image [k8s.gcr.io/etcd-amd64:3.2.18]: exit status 1
        [ERROR ImagePull]: failed to pull image [k8s.gcr.io/coredns:1.1.3]: exit status 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
[root@localhost ~]# 
```
2. 参照文章评论，获取到镜像，感谢这位哥们，地址[https://www.datayang.com/article/45](https://www.datayang.com/article/45)，编写获取镜像脚本pullimages.sh
```
#!/bin/bash
images=(kube-proxy-amd64:v1.11.1 kube-scheduler-amd64:v1.11.1 kube-controller-manager-amd64:v1.11.1
kube-apiserver-amd64:v1.11.1 etcd-amd64:3.2.18 coredns:1.1.3 pause:3.1 )
for imageName in ${images[@]} ; do
docker pull anjia0532/google-containers.$imageName
docker tag anjia0532/google-containers.$imageName k8s.gcr.io/$imageName
docker rmi anjia0532/google-containers.$imageName
done
```
执行脚本
```
[root@localhost ~]# sh pullimages.sh 
```
查看获取的镜像，仓库和版本都正确
```
[root@localhost ~]# docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy-amd64                v1.11.1             d5c25579d0ff        11 months ago       97.8 MB
k8s.gcr.io/kube-scheduler-amd64            v1.11.1             272b3a60cd68        11 months ago       56.8 MB
k8s.gcr.io/kube-controller-manager-amd64   v1.11.1             52096ee87d0e        11 months ago       155 MB
k8s.gcr.io/kube-apiserver-amd64            v1.11.1             816332bd9d11        11 months ago       187 MB
k8s.gcr.io/coredns                         1.1.3               b3b94275d97c        13 months ago       45.6 MB
k8s.gcr.io/etcd-amd64                      3.2.18              b8df3b177be2        15 months ago       219 MB
k8s.gcr.io/pause                           3.1                 da86e6ba6ca1        18 months ago       742 kB
[root@localhost ~]# 
```
再次执行kubeadm init --config kubeadm.yaml，成功后会有如下信息
```
······
Your Kubernetes master has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
You can now join any number of machines by running the following on each node
as root:
  kubeadm join 192.168.5.221:6443 --token o2ftww.m5061fz465yhhsib --discovery-token-ca-cert-hash sha256:1d7496ac9bccf7d33189ff7ec6ade8529490138f632dce3dc6a6ad4ea9ed8337
[root@localhost ~]# 
```
如果token忘掉，可以执行如下，其他重新创建token等命令可参考官网[https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)
如果获取--discovery-token-ca-cert-hash，执行openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
>    openssl dgst -sha256 -hex | sed 's/^.* //'参考官网[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
```
[root@localhost ~]# kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION   EXTRA GROUPS
o2ftww.m5061fz465yhhsib   23h       2019-07-09T19:05:04+08:00   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
```

三、关机重启后错误处理
注意，如果没有永久关闭SELinux，则需要再执行``` setenforce 0 ```；关闭交换内存(swapoff -a)
1. 获取token失败处理，如下错误，一般是交换内存没有关闭
```
[root@kubeadm ~]# kubeadm token list                        
failed to list bootstrap tokens [Get https://192.168.5.221:6443/api/v1/namespaces/kube-system/secrets?fieldSelector=type%3Dbootstrap.kubernetes.io%2Ftoken: dial tcp 192.168.5.221:6443: connect: connection refused]
```
2. 如果想要重新部署Master，重新初始化kubeadm，先执行如下
```
[root@kubeadm ~]# kubeadm reset -f
```
3. 再初始化配置Master节点
```
[root@kubeadm ~]# kubeadm init --config kubeadm.yaml
```
4. 如果过程中有错误如下
```
 [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
        [ERROR Swap]: running with swap on is not supported. Please disable swap
```
执行如下操作
```
[root@kubeadm ~]# swapoff -a
```
```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```
[root@kubeadm ~]# sysctl --system
```
再执行第3步
5. 执行完如下操作后，执行kubeadm token list获取token
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

四、kubectl检查节点
1. 第一次使用Kubernetes集群进行的配置(如果上面做过了就跳过)
```
[root@kubeadm ~]# mkdir -p $HOME/.kube
[root@kubeadm ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@kubeadm ~]# chown $(id -u):$(id -g) $HOME/.kube/config
```
这些配置命令原因：Kubernetes集群默认需要加密方式访问，这几条命令就是，将刚刚部署生成的Kubernetes集群的安全配置文件，保存到当前用户的.kube目录下，kubectl默认会使用这个目录下的授权信息访问Kubernetes集群。如果不这样做，每次都需要通过export KUBECONFIG环境变量告诉kubectl这个安全配置文件的位置
2. kubectl get命令查看节点状态，发现status是notready
```
[root@kubeadm ~]# kubectl get nodes 
NAME      STATUS     ROLES     AGE       VERSION
kubeadm   NotReady   master    36m       v1.11.1
```
3. kubectl describe查看节点(Node)详细信息、状态和事件(Event)，可分析出Master节点notready原因是没有部署任何网络插件。注意命令最后是节点名称
```
[root@kubeadm ~]# kubectl describe node kubeadm
Name:               kubeadm
Roles:              master
······
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  OutOfDisk        False   Tue, 09 Jul 2019 20:28:00 +0800   Tue, 09 Jul 2019 20:20:39 +0800   KubeletHasSufficientDisk     kubelet has sufficient disk space available
······
 Ready            False   Tue, 09 Jul 2019 20:28:00 +0800   Tue, 09 Jul 2019 20:20:39 +0800   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```
4. kubectl检查这个节点上各个系统Pod的状态，其中kube-system是Kubernetes项目预留的系统Pod的工作空间(Namespace，区别于Linux Namespace，此处的Namespace只是Kubernetes划分不同工作空间的单位)
```
[root@kubeadm ~]# kubectl get pods -n kube-system
NAME                              READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-hp5gg          0/1       Pending   0          49m
coredns-78fcdf6894-qmt2w          0/1       Pending   0          49m
etcd-kubeadm                      1/1       Running   0          48m
kube-apiserver-kubeadm            1/1       Running   0          48m
kube-controller-manager-kubeadm   1/1       Running   0          48m
kube-proxy-n72z7                  1/1       Running   0          49m
kube-scheduler-kubeadm            1/1       Running   0          48m
```
由于Master节点的网络尚未就绪，所以CoreDNS等依赖网络的Pod处于Pending状态，即调度失败

五、部署网络插件
1. 以Weave为例，部署网络插件
```
[root@kubeadm ~]# kubectl apply -f https://git.io/weave-kube-1.6
```
2. 再次查看Pod状态，如果发现镜像weave-net-gqxr4的status开始为ContainerCreating，然后变为ErrImagePull，说明其中镜像获取失败，那么就要打开https://git.io/weave-kube-1.6这个yaml文件，找到对应镜像，使用docker pull命令拉取下来就可以了
```
[root@kubeadm ~]# kubectl get pods -n kube-system               
NAME                              READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-hp5gg          1/1       Running   0          1h
coredns-78fcdf6894-qmt2w          1/1       Running   0          1h
etcd-kubeadm                      1/1       Running   0          1h
kube-apiserver-kubeadm            1/1       Running   0          1h
kube-controller-manager-kubeadm   1/1       Running   0          1h
kube-proxy-n72z7                  1/1       Running   0          1h
kube-scheduler-kubeadm            1/1       Running   0          1h
weave-net-gqxr4                   2/2       Running   0          7m
```
发现系统Pod都成功启动了，而刚刚部署的Weave网络插件则在kube-system(-n kube-system，工作空间为kube-system的Pods)下面新建了weave-net-gqxr4的Pod，一般这些Pod是容器网络插件在每个节点上的控制组件
3. Kubernetes支持容器网络插件，使用的是一个名叫CNI的通用接口(回想Kubenetes架构图)，它也是当前容器网络的事实标准，市面上的所有容器网络开源项目都可以通过CNI接入Kubernetes，如Flannel、Calico、Canal、Romana等，它们的部署方式都是类似的"一键部署"

六、部署Kubernetes的Worker节点
1. Kubernetes的Worker节点跟Master节点几乎是相同的，它们运行着的都是一个kubelet组件。唯一区别在于，在kubeadm init的过程中，kubelet启动后，Master节点上还会自动运行kube-apiserver、kube-scheduler、kube-controller-manager这三个系统Pod。
2. 部署Worker节点只需两步
1)、在所有Worker节点上执行"安装kubeadm和Docker"一节的所有步骤
2)、执行部署Master节点时生成的kubeadm join指令
```
yum -y update
```
```
 yum install -y docker
```
```
systemctl stop firewalld
systemctl mask firewalld
```
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
```
setenforce 0
```
```
yum install -y kubectl-1.11.1
```
```
yum install -y kubelet-1.11.1
```
```
yum install -y kubeadm-1.11.1
```
```
systemctl enable docker.service
systemctl start docker.service
```
```
systemctl enable kubelet.service
```
```
swapoff -a
```
```
kubeadm join 192.168.5.221:6443 --token bbd5zc.g5u3b2mr7tuiwg3h --discovery-token-ca-cert-hash sha256:6d02d471b73289f17459918d17ee01d5020630acf044109e2fb1f42bb2036b46 
```
3. Worker节点删除后重新加入
现在Master节点执行如下操作
```
kubectl get nodes
```
```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
```
```
kubectl delete node <node name>
```
再在Worker节点执行如下，如果中间出现swap等错误，执行完相应操作后再次执行下面操作
```
kubeadm reset -f
```
```
kubeadm join 192.168.5.221:6443 --token 74uhkn.wmgt0vjts40928ec --discovery-token-ca-cert-hash 6d02d471b73289f17459918d17ee01d5020630acf044109e2fb1f42bb2036b46
```

