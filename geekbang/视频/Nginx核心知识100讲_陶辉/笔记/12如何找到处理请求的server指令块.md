1. 当对nginx发起一个http请求时，nginx根据请求中的host与server块的server_name进行匹配，找到对应的server块进行处理。
如果server块下没有listen监听端口，那么默认是80；一个http块下的多个server块下，listen可以监听不同端口，使用查看端口命令查询结果如下
```
[root@localhost ~]#  netstat -tunlp 
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1013/master         
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1255/nginx: master  
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      1255/nginx: master  
tcp        0      0 0.0.0.0:8081            0.0.0.0:*               LISTEN      1255/nginx: master  
```
2. server_name支持三种三种域名
 - 指令后跟多个域名，第一个域名为主域名；
 - 泛域名，仅支持在最前面或最后面加星号(*),来匹配请求中的域名
 - 正则表达式，需要使用波浪线(~)做前缀

 当访问请求执行跳转的时候，可以通过配置server_name_in_redirect开关配置是否启用跳转时用主域 名进行跳转。server_name_in_redirect可以出现在http、server、location指令块下。
3. server_name_in_redirect指令演示。创建一个单独的nginx配置文件hjp.conf，在nginx.conf配置文件中通过include命令加载hjp.conf配置文件。在hjp.conf配置文件中配置server块。
```
http {
    ······
    include /root/nginx/conf/hjp/hjp.conf;
    ······
}
```
hjp.conf配置如下
```
server {
    server_name www.hjp1.cn www.hjp2.cn;
    #默认是关闭的
    server_name_in_redirect on;
    return 302 /;
}
```
默认情况下执行```curl http://www.hjp2.cn/ -I```命令，如下，Location不指向主域名
```
[root@localhost ~]# curl http://www.hjp2.cn/ -I
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.14.2
Date: Sat, 21 Dec 2019 09:33:04 GMT
Content-Type: text/html
Content-Length: 161
Location: http://www.hjp2.cn/
Connection: keep-alive
```
开启后执行```curl http://www.hjp2.cn/ -I```命令，如下，Location指向主域名
```
[root@localhost ~]# curl http://www.hjp2.cn/ -I
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.14.2
Date: Sat, 21 Dec 2019 09:34:24 GMT
Content-Type: text/html
Content-Length: 161
Location: http://www.hjp1.cn/
Connection: keep-alive
```
4. server_name中使用正则表达式变量案例如下，使用"$索引"方式和"$变量名"方式。请求方式分别为```http://www.hjp2.cn:8081/```和```http://www.hjp2.cn:8082/```
```
server {
    listen 8081;
    server_name ~^(www\.)?(.+)\.cn$;
    location / {
        index $2.html;
    }
}
server {
    listen 8082;
    server_name ~^(www\.)?(?<varName>.+).cn$;
    location / {
        index $varName.html;
    }
}
```
5. server_name其他使用方式
 - .hjp.cn可以匹配hjp.cn和*.hjp.cn；
 - _匹配所有;
 - "" 匹配没有传递host头部。

6. server匹配顺序，根据server_name来匹配，和server块所在位置及其顺序无关
 - 首先精确匹配；
 - 如匹配不上，再匹配星号(*)在前的泛域名；
 - 如匹配不上，再匹配星号(*)在后的泛域名；
 - 如匹配不上，再匹配正则表达式域名，如果正则匹配出多个，就选择第一个；
 - 如匹配不上，就使用server块下listen指令后标记default的server块；
 - 如没有listen为default的server块，就使用第一个server块。