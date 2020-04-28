1. 要使用Nginx的模块，首先是要编译进Nginx二进制文件的。有的模块编译进去后就可以被使用，有的模块需要配置才能被使用。
2. Nginx模块的使用方式可以看Nginx官网。```http://nginx.org/en/docs/```
3. 在编译完源码后的objs目录下，查看ngx_modules.c文件，ngx_modules[]中包含了编译进Nginx二进制文件的模块。
```
[root@localhost objs]# pwd
/root/nginx-1.14.2/objs
[root@localhost objs]# cat  ngx_modules.c
```
4. 通过查看源代码，看模块提供的指令，如ngx_http_gzip_filter_module，找到ngx_command_t结构体，这个结构体是每一个模块都必须具备的。ngx_command_t结构体对应的是一个数组，数组中每一个成员都是一个指令名及其所需参数。如果某第三方模块说明文档不具体，可以到这块源码查看使用方式。(src/模块/下的文件是加载模块所必须的，src/模块/modules/下文件是可有可无的)
```
[root@localhost modules]# pwd
/root/nginx-1.14.2/src/http/modules
[root@localhost modules]# more ngx_http_gzip_filter_module.c 
```
4. Nginx模块之间是有顺序关系的，如果有冲突模块，先加载的模块后阻碍后加载的模块。模块有master进程和worker进程启动、执行时的回调方法，所以有些定制化的功能可以通过模块来实现。
