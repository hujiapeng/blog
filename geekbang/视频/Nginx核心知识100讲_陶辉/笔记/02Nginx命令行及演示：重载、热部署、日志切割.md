1. 启动Nginx，在执行完安装后的Nginx的sbin目录下，指定如下命令
```
[root@localhost sbin]# pwd
/root/nginx/sbin
[root@localhost nginx]# ./nginx
```
2. 重载：修改了nginx.conf配置文件后，需要重新加载配置文件且不会停止服务。如果修改的是端口，就要先stop然后再启动。因为有时候端口不能彻底停掉。
```
[root@localhost sbin]# ./nginx -s reload 
```
3. 热部署：
 - 查看nginx应用进程
```
[root@localhost sbin]# ps -ef |grep nginx
root      9700     1  0 13:16 ?        00:00:00 nginx: master process ./nginx
root      9730  9700  0 13:36 ?        00:00:00 nginx: worker process
root      9735  9669  0 13:40 pts/1    00:00:00 grep --color=auto nginx
```
 - 将新版本编译好的nginx二进制文件(objs目录下)拷贝到当前nginx安装目录sbin下(拷贝前先做好备份cp nginx nginx.old)
 - 给Master进程发送版本升级信号。kill -USR2 master进程ID。
```
[root@localhost sbin]# kill -USR2 9700
```
 - 此时再用ps命令看nginx进程，可以看到两个master进程和对应的新旧worker进程，新旧两个版本的nginx。新旧worker都在运行，旧的worker进程会平滑的过度到新的worker进程，而旧worker进程不再监听web端口，全部有新worker进程监听，也就是新的请求会直接使用新worker进程服务。
```
[root@localhost sbin]# ps -ef |grep nginx
root      9700     1  0 13:16 ?        00:00:00 nginx: master process ./nginx
root      9730  9700  0 13:36 ?        00:00:00 nginx: worker process
root      9767  9700  0 14:20 ?        00:00:00 nginx: master process ./nginx
root      9768  9767  0 14:20 ?        00:00:00 nginx: worker process
root      9771  9669  0 14:23 pts/1    00:00:00 grep --color=auto nginx
```
 - 发送优雅关闭所有旧worker进程命令。kill -WINCH 旧版本master进程ID。
```
[root@localhost sbin]# kill -WINCH 9700
```
 - 查看版本升级后的nginx。旧的worker进程已经关闭，而旧的master进程还在运行，这个是为了做版本回退用的。
```
[root@localhost sbin]# ps -ef |grep nginx
root      9700     1  0 13:16 ?        00:00:00 nginx: master process ./nginx
root      9767  9700  0 14:20 ?        00:00:00 nginx: master process ./nginx
root      9768  9767  0 14:20 ?        00:00:00 nginx: worker process
root      9775  9669  0 14:33 pts/1    00:00:00 grep --color=auto nginx
```
4. 日志切割：
 - logs目录下日志文件变大后，需要使用新文件继续记录日志，那就用到了日志切割
```
[root@localhost logs]# ll
total 16
-rw-r--r--. 1 root root  444 Dec  1 13:37 access.log
-rw-r--r--. 1 root root 1062 Dec  1 14:20 error.log
-rw-r--r--. 1 root root    5 Dec  1 14:20 nginx.pid
-rw-r--r--. 1 root root    5 Dec  1 13:16 nginx.pid.oldbin
``` 
 - 如对记录访问请求的access.log(使用curl命令或其他访问请求日志会记录在这个文件中)进行分隔，先将原文件重命名，不要使用cp命令，这样会有数据信息丢失，使用mv重命名方式不会。
```
[root@localhost logs]# mv access.log access.log_bak 
[root@localhost logs]# ll
total 16
-rw-r--r--. 1 root root  616 Dec  1 14:43 access.log_bak
-rw-r--r--. 1 root root 1062 Dec  1 14:20 error.log
-rw-r--r--. 1 root root    5 Dec  1 14:20 nginx.pid
-rw-r--r--. 1 root root    5 Dec  1 13:16 nginx.pid.oldbin
```
 - 使用reopen命令进行日志切割
```
[root@localhost logs]#  ../sbin/nginx -s reopen
[root@localhost logs]# ll
total 16
-rw-r--r--. 1 root root    0 Dec  1 14:44 access.log
-rw-r--r--. 1 root root  616 Dec  1 14:43 access.log_bak
-rw-r--r--. 1 root root 1122 Dec  1 14:44 error.log
-rw-r--r--. 1 root root    5 Dec  1 14:20 nginx.pid
-rw-r--r--. 1 root root    5 Dec  1 13:16 nginx.pid.oldbin
```
 - 实际使用中，可以将mv和nginx -s reopen命令放到bash脚本中，然后通过crontab定时工具来处理定时日志切割。

