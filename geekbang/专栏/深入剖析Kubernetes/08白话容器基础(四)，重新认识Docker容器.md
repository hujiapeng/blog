一、用Docker部署一个用Python编写的Web应用
1. app.py代码中使用Flask框架启动一个Web服务，功能是，如果当前环境有"NAME"环境变量，就打印出来，否则就打印"Hello World"，最后再打印出当前环境的hostname
```
from flask import Flask
import socket
import os
app = Flask(__name__)
@app.route('/')
def hello():
    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>"           
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname())
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```
2. 这个应用依赖一个文件requirements.txt，在同目录下
```
[root@ittutu ~]# ls app.py requirements.txt 
app.py  requirements.txt
[root@ittutu ~]# 
```
3. 编写Dockerfile，Dockerfile的设计思想，是使用一些标准的原语(即大写高亮的词语)，描述我们所要构建的Docker镜像。并且这些原语是按顺序处理的。注意，Dockerfile里的原语并**不是都**是指对容器内部的操作，如ADD，表示把当前目录里文件，复制到容器的目录中。
```
[root@ittutu dockerfiles]# cat Dockerfile 
# 使用官方提供的Python开发镜像作为基础镜像
FROM python:2.7-slim
# 将工作目录切换为 /app，进入容器后，先cd / ，后ls命令可看到app
WORKDIR /app
# 将当前目录下的所有内容复制到/app下
ADD . /app
# 使用pip命令安装这个应用所需要的依赖，
# 详细可查pip安装依赖命令，其中requirements.txt中为需要的依赖包名
RUN pip install --trusted-host pypi.python.org -r requirements.txt
# 允许外界访问容器的80端口
EXPOSE 80
# 设置环境变量
ENV NAME World
# 设置容器进程为：python app.py，即这个Python应用的启动命令
CMD ["python","app.py"]
[root@ittutu dockerfiles]# 
```
4. 使用Dockerfile制作容器镜像。-t作用是给这个镜像加一个Tag，即起一个名字。Dockerfile中的每个原语执行后，都会生成一个对应的镜像层。即使原语本身并没有明显地修改操作，也会产生一个对应层，只不过在外界看这个层是空的
```
[root@ittutu dockerfiles]# docker build -t helloworld .
Sending build context to Docker daemon 4.096 kB
Step 1/7 : FROM python:2.7-slim
Trying to pull repository docker.io/library/python ... 
fc7181108d40: Downloading [=======================>                           ] 10.55 MB/22.49 MB
8c60b810a35a: Download complete 
8c60b810a35a: Downloading [=========================================>         ] 2.096 MB/2.528 MB
63184f224d60: Download complete 
63184f224d60: Waiting 
```
5. docker build完成后，可以通过docker images命令查看结果。还可以用docker inspect(或 docker image inspect)查看镜像详细信息
```
[root@ittutu dockerfiles]# docker images       
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
helloworld                  latest              7dd18f7ef99a        2 minutes ago       130 MB
```
6. 启动容器，启动后，docker会自动执行Dockerfile中指定的CMD中命令。其中-p 参数是告诉Docker把容器内的80端口映射在宿主机的4000端口上，否则要先用docker inspect命令查看容器的IP地址，访问"http://<容器IP地址>:80"才能看到容器内应用返回结果
```
[root@ittutu dockerfiles]# docker run -p 4000:80 helloworld
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```
docker inspect 容器ID 查看到容器中的IP，也可以如下访问
```
[root@ittutu ~]# curl http://172.18.0.3:80
<h3>Hello World!</h3><b>Hostname:</b> a644cbd7b07e<br/>[root@ittutu ~]# 
```
7. 新开一客户端，查看运行中的容器
```
[root@ittutu ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS                  NAMES
a644cbd7b07e        helloworld          "python app.py"     About a minute ago   Up About a minute   0.0.0.0:4000->80/tcp   adoring_shaw
[root@ittutu ~]# 
```
8. 查看容器内应用返回值
```
[root@ittutu ~]# curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> a644cbd7b07e<br/>[root@ittutu ~]# 
```
使用如下命令查看容器的hostname，结果是一致的
```
[root@ittutu ~]# docker inspect a644cbd | grep Hostname
        "HostnamePath": "/var/lib/docker/containers/a644cbd7b07eba5ec3bc240158920c76561cf0cc3cc41569c4aaade7f5ff2a03/hostname",
            "Hostname": "a644cbd7b07e",
[root@ittutu ~]# 
```

二、将镜像上传至Docker Hub
1. 注册Docker Hub 账号，然后使用docker login 登录
```
[root@ittutu ~]# docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: hujiapeng
Password: 
Login Succeeded
[root@ittutu ~]# 
```
2. 用docker tag 给镜像起一个完整的名字
```
[root@ittutu ~]# docker tag helloworld hujiapeng/helloworld:v1
[root@ittutu ~]# docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
helloworld                  latest              7dd18f7ef99a        53 minutes ago      130 MB
hujiapeng/helloworld        v1                  7dd18f7ef99a        53 minutes ago      130 MB
```
注意：hujiapeng是我在Docker Hub上的用户名，它的"学名"叫镜像仓库(Repository)；"/"后面的helloworld是这个镜像的名字，而"v1"是这个镜像的版本号。在Docker Hub上一个用户只能有一个免费的公开镜像仓库，仓库名就是用户的登录名
3. 上传镜像，上传成功后，可以到Docker Hub仓库看到helloworld镜像
```
[root@ittutu ~]# docker push hujiapeng/helloworld:v1
The push refers to a repository [docker.io/hujiapeng/helloworld]
e3888241a730: Pushed 
f7e28b2179db: Pushed 
1fe4d1304244: Pushed 
a212ef9c5ee1: Mounted from library/python 
a47fa5565167: Mounted from library/python 
658556256f47: Mounted from library/python 
cf5b3c6798f7: Mounted from library/python 
v1: digest: sha256:a0695e5cfb442c8373af65cdeb13f0225732cea2893a78effe84d2d3b955d5f4 size: 1787
[root@ittutu ~]# 
```
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/geekbang/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90Kubernetes/%E4%B8%8A%E4%BC%A0%E7%9A%84%E9%95%9C%E5%83%8F.jpg)
4. 此外，可以使用docker commit指令，把一个正在运行的容器，直接提交为一个镜像。如做了如下操作
```
[root@ittutu ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                  NAMES
a644cbd7b07e        helloworld          "python app.py"     58 minutes ago      Up 58 minutes       0.0.0.0:4000->80/tcp   adoring_shaw
[root@ittutu ~]# docker exec -it a644cbd7b07e /bin/sh
# touch test.txt
# echo aaaa > test.txt
# exit
[root@ittutu ~]# docker commit a644cbd7b07e hujiapeng/helloworld:v2
sha256:f664d21346a575a7c71640972bc00c9acb7952a623a030374100460379da6a5b
[root@ittutu ~]# docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
hujiapeng/helloworld        v2                  f664d21346a5        5 seconds ago       131 MB
helloworld                  latest              7dd18f7ef99a        About an hour ago   130 MB
hujiapeng/helloworld        v1                  7dd18f7ef99a        About an hour ago   130 MB
```
5. docker commit实际上就是在容器运行起来后，把最上层的"可读写层"，加上原先容器镜像的只读层，打包组成一个新的镜像。
新镜像和原镜像中相同的只读层是在宿主机上共享的，这就是使用层来保存镜像的好处，不会额外占用空间

三、docker exec 原理
1. Linux Namespace创建的隔离空间虽然看不到，但是一个进程的Namespace信息在宿主机上确确实实存在的，并且以一个文件的方式，通过如下指令可以看到正在运行的容器进程ID
```
[root@ittutu ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                  NAMES
a644cbd7b07e        helloworld          "python app.py"     About an hour ago   Up About an hour    0.0.0.0:4000->80/tcp   adoring_shaw
[root@ittutu ~]# docker inspect --format '{{.State.Pid}}' a644cbd7b07e
10969
[root@ittutu ~]# 
```
这时，可以通过查看宿主机的proc文件，看到这个10969进程的所有Namespace对应的文件
```
[root@ittutu ~]# ls -l /proc/10969/ns
total 0
lrwxrwxrwx 1 root root 0 Jun 26 14:40 ipc -> ipc:[4026532224]
lrwxrwxrwx 1 root root 0 Jun 26 14:19 mnt -> mnt:[4026532222]
lrwxrwxrwx 1 root root 0 Jun 26 14:19 net -> net:[4026532227]
lrwxrwxrwx 1 root root 0 Jun 26 14:40 pid -> pid:[4026532225]
lrwxrwxrwx 1 root root 0 Jun 26 16:00 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Jun 26 14:40 uts -> uts:[4026532223]
[root@ittutu ~]# 
```
2. 一个进程的每种Linux Namespace，都在它对应的/proc/[进程号]/ns下有一个对应的虚拟文件，并且链接到一个真实的Namespace文件上。
一个进程，可以选择加入到某个进程已有的Namespace当中，从而达到"进入"这个进程所在容器的目的，这就是docker exec的实现原理。这个操作依赖系统调用setns()。
```
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)
int main(int argc, char *argv[]) {
    int fd;
    
    fd = open(argv[1], O_RDONLY);
    if (setns(fd, 0) == -1) {
        errExit("setns");
    }
    execvp(argv[2], &argv[2]); 
    errExit("execvp");
}
```
上述代码，接收两个参数，第一个参数argv[1]表示当前进程要加入的Namespace文件的路径，如/proc/10969/ns/net；第二个参数argv[2]表示要在这个Namespace里运行的进程，如/bin/sh。核心代码是通过open()系统调用打开指定的Namespace文件，并把这个文件的描述符fd交给setns()使用。在setns()执行后，当前进程就加入了这个文件对应的Linux Namespace当中了。

四、Docker Volume(数据卷)
1. Volume机制：允许将宿主机上指定的目录或文件，挂载到容器里进行读取和修改操作
2. 挂载方式有两种：(1)、-v /test，这种方式没有显示声明宿主机目录，那么Docker会默认在宿主机上创建一个临时目录/var/lib/docker/volumes/[VOLUME_ID]/_data，然后把它挂载到容器/test目录上。(2)、-v /home:/test，这种方式是把宿主机上的/home目录挂载到容器的/test目录上
3. 前面讲到，虽然使用Mount Namespace进行了文件声明挂载，没有执行chroot(或pivot_root)前，容器进程还是可以看到系统所有文件的；
容器启动后，镜像的各个层(/var/lib/docker/aufs/diff)会被联合挂载在/var/lib/docker/aufs/mnt/目录中，这样容器所需的footfs就准备好了；
所以，只需在rootfs准备好之后，在执行chroot(或pivot_root)之前，把Volume指定的宿主机目录(如/home)，挂载到指定的容器目录(如/test)在宿主机上对应的目录(即，/var/lib/docker/aufs/mnt[可读写层ID]/test)上，这个Volume的挂载工作就完成了
4. Linux的绑定挂载(bind mount)机制：允许你将一个目录或文件，而不是整个设备，挂载到一个指定的目录上。并且这时，你在该挂载点上进行的任何操作，只是发生在被挂载的目录或文件上，而原挂载点的内容则会被隐藏起来且不受影响
如上例子，-v /home:/test，就是宿主机的/home挂载到容器的/test上，此时对test上的任何操作，实际上是在对/home操作，而原来/test上原有的数据被隐藏起来且不受影响。注意：容器中/test中数据也会在宿主机/home下(Docker的copyData功能)，对/test数据修改后，重新启动容器，数据又恢复到原始。此时对宿主机/home的操作也会显示在容器/test下
docker commit是发生在宿主机上的，由于Mount Namespace的隔离作用(相当于容器内进行的操作)，宿主机并不知道这个绑定挂载的存在，所以对于宿主机看来，容器中可读写层的/test目录还是原来的目录，没有变化
5. 容器数据卷操作
(1)、列出当前镜像
```
[root@ittutu ~]# docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
hujiapeng/helloworld        v2                  2deb3e799b02        About an hour ago   130 MB
```
(2)、创建数据卷，如果宿主机或容器中的指定目录不存在，会自动创建。如果宿主机使用相对路径，如-v hjpDVo:/test ，那么hjpDVo会在Docker默认路径(/var/lib/docker/volumes/)下创建hjpDVo；同时注意，容器中的/test，是相对容器根(/)目录说的，跟工作目录(如，/app)无关，容器目录必须是绝对路径，否则无法启动，报错如(```invalid mount path: 'hjpTest' mount path must be absolute.```)。另一个注意，如果宿主机使用绝对路径，使用docker volume ls 命令是查不到数据卷的；而且也不会把容器中/test下内容在/home下显示
```
[root@ittutu ~]# docker run -it -v /root/hjpDVo:/app/hjpTest 2deb3e799
# 
```
(3)、查看数据卷，及数据卷详细信息
```
[root@ittutu ~]# docker volume ls
DRIVER              VOLUME NAME
local               213f7fa2ff67df089224e1ff1313a22016816f376efe8d51e4186335bac77214
local               36c94053cab82d4d30ac8f289ca290052d381170876ffa07a6cf93e18f9cf974
local               hjpDVo
[root@ittutu ~]# docker volume inspect hjpDVo
[
    {
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/hjpDVo/_data",
        "Name": "hjpDVo",
        "Options": {},
        "Scope": "local"
    }
]
[root@ittutu ~]# 
```
(4)、exit命令退出容器后，想要删除所有停止的容器，执行命令如下
```
docker rm $(docker ps -aq)
```
(5)、删除所有数据卷
```
docker volume rm $(docker volume ls -qf dangling=true)
```
