1. goaccess安装(https://www.vultr.com/docs/how-to-install-goaccess-on-centos-7)
2. 注意log_format格式暂时使用默认，都开启
```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
```
3. 执行goaccess命令配置日志输出report.html到nginx/html下。log-format=COMBINED只对goaccess指定的标准日志格式生效。启动后会有websocket连接信息，否则没有启动成功，只是生成了静态report.html文件
```
[root@localhost nginx]# pwd
/root/nginx
[root@localhost nginx]# ../goaccess/bin/goaccess logs/hujp.access.log -o root/openresty/nginx/html/report.html --real-time-html --time-format='%H:%M:%S' --date-format='%d/%b/%Y' --log-format=COMBINED
```
4. 配置goaccess生成的report.html访问location。server块下添加一个location
```
        location /report.html {
            alias html/report.html;            
        }
```