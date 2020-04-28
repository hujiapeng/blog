1. 注意403Forbidden页面出现时，要注意两个地方
 - nginx.conf配置文件的第一行user使用root或者高权限用户
 - 关闭或设置服务器防火墙

2. location块下指定文件路径有两种方式，root和alias，主要区别是如何解释location后面的uri，nginx服务器分别以不同方式返回请求地址。注意如果不指定访问文件，且不做其他默认页面配置，访问路径下要有index.html文件
 - root： 处理结果是，root路径+location路径；
 - alias：处理结果是，使用alias路径替换location路径。（推荐）

3. 修改完配置文件，执行./nginx -s reload命令
4. http块下开启gzip压缩，并配置相关gzip参数
```
#gzip off;
 gzip  on;
#配置压缩的最小文件大小，也就是小于该值的不再压缩，为了演示效果设置为1
 gzip_min_length 1;
#gzip压缩级别
 gzip_comp_level 2;
 gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml application/x-httpd-php image/jpeg image gif image/png;
```
5. 显示目录结构共享静态资源的配置，注意alias依然生效，而且访问目录时候以斜杆"/"结尾。访问的时候如果出错，就到logs/error.log文件中查看。
```
#要访问目录的结构
[root@localhost nginx]# tree hujp
hujp
├── hujp
│   ├── controller
│   │   └── test.html
│   ├── repository
│   └── service
└── index.html
4 directories, 2 files
[root@localhost nginx]# 
```
location块中开启autoindex，访问：```http://192.168.7.53/hujp/```此处的hujp是第二个文件夹。
```
location / {
            alias hujp/;
            autoindex on;
        }
```
6. 限制某些请求的访问速度，通过设置变量$limit_rate。参考地址```http://nginx.org/en/docs/http/ngx_http_core_module.html#variables```
```
location / {
       alias hujp/;
       autoindex on;
      # 设置nginx向浏览器返回数据的速度，每秒传输字节数
       set $limit_rate 1k;
      # root   html;
      # index  index.html index.htm;
   }
```
7. 日志格式设置及日志输出位置，日志中还可以使用变量，如$remote_addr。http块下通过log_format设置日志格式，并给日志格式命名。如下
```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
```
然后可以在相应的块下设置日志输出位置及使用格式。如server块下配置日志并使用main日志格式
```
 server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        access_log  logs/hujp.access.log  main;
}
```
