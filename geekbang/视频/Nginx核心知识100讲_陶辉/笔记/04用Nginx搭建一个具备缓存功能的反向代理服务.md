1. 搭建openResty作为反向代理服务器，之前的nginx作为上游服务器(产生内容的服务器)。搭建openResty步骤如下，和nginx差不多。先停掉当前的nginx服务```nginx -s stop```。
```
#安装必要的编译环境
[root@localhost ~]# yum install pcre-devel openssl-devel gcc
#下载openResty源码并解压
[root@localhost ~]# wget https://openresty.org/download/openresty-1.15.8.1.tar.gz
[root@localhost ~]# tar -zxvf openresty-1.15.8.1.tar.gz
#编译并安装，注意编译前的configure要指定加载luajip模块
[root@localhost ~]# cd openresty-1.15.8.1
[root@localhost openresty-1.15.8.1]# ./configure --prefix=/root/openresty --with-luajit
[root@localhost openresty-1.15.8.1]# make
[root@localhost openresty-1.15.8.1]# make install
#启动openResty，其实就是启动nginx下的nginx文件。记得修改nginx/conf/nginx.conf文件第一行user为root
[root@localhost openresty-1.15.8.1]# cd /root/openresty
[root@localhost openresty]# bin/openresty  
[root@localhost openresty]# ps -ef | grep nginx
root     26487     1  0 16:05 ?        00:00:00 nginx: master process bin/openresty
root     26501 26487  0 16:07 ?        00:00:00 nginx: worker process
root     26510  1017  0 16:40 pts/0    00:00:00 grep --color=auto nginx
```
2. 修改nginx上游服务器配置server块，只允许本机进程访问8080端口。(如果此时的nginx没有停掉，要先停掉```nginx -s stop```再启动，防止之前的端口没有彻底关掉)
```
listen       127.0.0.1:8080;
```
3. 修改反向代理服务器openResty配置。http块下添加upstream块，然后添加上游服务器；location块添加proxy_pass配置对应的upstream。
```
http{
    ······
    upstream local {
        server 127.0.0.1:8080;
    }
    server {
        ······
        location / {
            proxy_pass http://local;
        }
        ······

    }
}
```
4. 由于浏览器与反向代理是一个TCP连接，反向代理与上游服务器是一个TCP连接，上游服务器获取的信息并不是浏览器的，而是反向代理的。上游服务器想要获得真实客户端信息，需要在代理服务器的location块下配置proxy_set_header，使得上游服务器获取到浏览器信息，如浏览器地址输入的域名，就可以通过$host获取到。
```
 location / {
     proxy_pass http://local;
     proxy_set_header Host $host;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 }
```
5. 反向代理配置代理缓存。会把上游服务器内容，缓存到一个文件内，使用缓存时，不再请求上游服务器。先在http块下配置缓存主要参数，并命名缓存Key等信息。在lcoation块下使用缓存。
```
http {
     ······
     #配置缓存文件存储位置，缓存级别，缓存Key值及缓存大小等信息
     proxy_cache_path /tmp/hujp/nginxcache levels=1:2 keys_zone=hujp_cache:10m max_size=10g inactive=60m use_temp_path=off;  
     ······
     server {
       ······
        location {
            ······
            proxy_pass http://local;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            #使用hujp_cache这个缓存
            proxy_cache hujp_cache;
            #设置缓存Key命名方式，可以根据不同请求设置相应缓存(如下cache_key配置导致无法使用缓存，只保留$host，再reload就可以使用缓存了，想要知道原因，需要等后面再学学)
            proxy_cache_key $host$uri$is_args$args;
            #设置缓存生效策略及时间，仅对如下返回码生效缓存
            proxy_cache_valid 200 304 302 1d;
            ·····
        }
       ······
    }
}
```