1. 安装Nginx有两种方法，除了编译以外，还可以借助系统上的yum或apt-get等工具直接安装Nginx Binary。
但Nginx二进制安装会有个大问题，Nginx Binary会把模块直接编译进去，而Nginx官方模块并不会每一个都默认开启，如果想添加第三方Nginx模块，必须通过编译Nginx这种方式添加。
2. 下载Nginx并解压
```
[root@localhost ~]# wget http://nginx.org/download/nginx-1.14.2.tar.gz
[root@localhost ~]# tar -zxvf nginx-1.14.2.tar.gz
[root@localhost ~]# ll
total 996
drwxr-xr-x. 8 1001 1001     158 Dec  4  2018 nginx-1.14.2
-rw-r--r--. 1 root root 1015384 Dec  4  2018 nginx-1.14.2.tar.gz
```
3. 如果使用vim，安装vim方法，相关配置可以搜索，主要修改~/.vimrc文件
```
[root@localhost ~]# yum -y install vim-enhanced
```
如果要使用nginx自带的vim配置文件，就把nginx/contrib/vim下文件都拷贝到~/.vim文件夹下，如果不存在.vim文件夹就自己创建
```
[root@localhost ~]# cp -r nginx-1.14.2/contrib/vim/* ~/.vim
```
4. 在nginx目录下，查看configure帮助文档
```
[root@localhost nginx-1.14.2]# ./configure --help | more
```
--prefix=PATH表示安装目录；
--modules-path=PATH设置动态安装模块路径；
--with开头的命令说明模块在nginx默认不开启，此命令是要添加对应模块；
--without开头的命令说明模块在nginx默认开启，此命令是要移除对应模块
5. 编译nginx之前注意nginx需要有必要的编译环境
```
[root@localhost nginx-1.14.2]# yum -y install gcc gcc-c++ autoconf automake make
```
还需要一些模块必要的依赖，如果编译校验过程中还有报错，就可以根据需要安装
```
[root@localhost ~]# yum -y install pcre-devel openssl openssl-devel
```
6. 指定安装路径，执行configure命令，开始编译前配置，为编译安装做准备，还会生成中间文件
```
[root@localhost nginx-1.14.2]#  ./configure --prefix=/root/nginx 
······
Configuration summary
  + using system PCRE library
  + OpenSSL library is not used
  + using system zlib library

  nginx path prefix: "/root/nginx"
  nginx binary file: "/root/nginx/sbin/nginx"
  nginx modules path: "/root/nginx/modules"
  nginx configuration prefix: "/root/nginx/conf"
  nginx configuration file: "/root/nginx/conf/nginx.conf"
  nginx pid file: "/root/nginx/logs/nginx.pid"
  nginx error log file: "/root/nginx/logs/error.log"
  nginx http access log file: "/root/nginx/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
  nginx http fastcgi temporary files: "fastcgi_temp"
  nginx http uwsgi temporary files: "uwsgi_temp"
  nginx http scgi temporary files: "scgi_temp"
```
7. configure命令后新生成objs文件夹，其中的ngx_modules.c文件指定了哪些modules会被编译进nginx。
8. 进行make编译。注意，在执行nginx版本升级时，不再执行make install命令，需要把编译好的objs目录下的nginx二进制文件拷贝安装目录的sbin下，替换原有nginx文件。c语言编译时生成的中间文件会放到objs/src目录下。如果使用了动态模块，动态模块编译生成的so动态文件也会放到objs文件夹下。
```
[root@localhost nginx-1.14.2]# make
```
9. 首次安装使用make install命令安装，安装完成后就可以在prefix指定的路径下看到安装文件。conf为nginx配置文件目录，sbin是nginx命令文件目录。
```
[root@localhost nginx-1.14.2]# make install
```

