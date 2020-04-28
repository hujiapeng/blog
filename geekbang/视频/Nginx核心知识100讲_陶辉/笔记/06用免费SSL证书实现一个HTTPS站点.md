1. 先修改本地hosts文件，配置域名。路径在```C:\Windows\System32\drivers\etc```，其中IP为一个虚拟机IP。
```
192.168.2.210 www.hujiapeng.com
```
2. Nginx服务器安装一个python工具
```
[root@localhost ~]# yum -y install python2-certbot-nginx
```
3. 申请证书。需要先将nginx可执行文件路径加入到PATH或使用软链接将nginx加入到```/usr/sbin/```中。certbot命令指定为nginx服务器，且指定nginx服务配置路径，指定要生成证书的域名。（注意，要使用开启SSL模块的nginx二进制文件```--with-http_ssl_module```；还有就是域名必须时DNS认证的）
```
[root@localhost ~]# ln -s /root/nginx/sbin/nginx /usr/sbin/nginx
[root@localhost ~]# certbot --nginx --nginx-server-root=/root/nginx/conf/ -d www.hujiapeng.com
```