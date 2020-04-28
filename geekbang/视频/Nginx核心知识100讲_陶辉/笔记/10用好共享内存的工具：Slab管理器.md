1. Slab首先将共享内存分为很多页面，每个页面会被切分为不同大小的Slot。当存储数据时，会找到第一个不小于数据大小的slot来存储，这样有个缺点就是会造成内存浪费，但这种内存管理方式适合小对象，避免了内存碎片，如果某对象数据结构是固定的，在使用时可以避免重复初始化。
2. 使用TEngine的slab_stat模块(http://tengine.taobao.org/document/ngx_slab_stat.html )，监控统计slab使用状态。注意slab_stat不是一个独立的工具，是tengine上的一个模块，所以需要先下载tengine，然后再在openResty上使用Tengine中的slab_stat模块。
 - 下载安装TEngine
```
[root@localhost ~]# wget http://tengine.taobao.org/download/tengine-2.3.2.tar.gz
[root@localhost ~]# tar -zxvf tengine-2.3.2.tar.gz 
#重新编译openResty
[root@localhost openresty-1.15.8.1]# ./configure --prefix=/root/openresty --add-module=/root/tengine-2.3.2/modules/ngx_slab_stat
[root@localhost openresty-1.15.8.1]# make
[root@localhost openresty-1.15.8.1]# cd
[root@localhost ~]# openresty/bin/openresty -s stop
[root@localhost ~]# cp openresty-1.15.8.1/build/nginx-1.15.8/objs/nginx openresty/nginx/sbin/nginx  
[root@localhost ~]# openresty/bin/openresty  
```
3. 修改openresty下的nginx.conf配置，添加slab_stat对应的location
```
http {
    ······
    lua_shared_dict dogs 10m;
    ······
    server {
        ······
        location = /slab_stat {
            slab_stat;    
        }

        location /set {
            content_by_lua_block {
                local dogs=ngx.shared.dogs
                dogs:set("Jim",8)
                ngx.say("STORED")
         }    
        }

        location /get {
            content_by_lua_block {
                 local dogs=ngx.shared.dogs
                 ngx.say(dogs:get("Jim"))
            }
        }
        ······
    }
    ······
}
```
 查看slab内存情况
```
[root@localhost ~]# curl http://localhost/set          
STORED
[root@localhost ~]# curl http://localhost/get          
8
[root@localhost ~]# curl http://localhost/slab_stat    
 * shared memory: hujp_cache
total:       10240(KB) free:       10168(KB) size:           4(KB)
pages:       10168(KB) start:00007F78AE851000 end:00007F78AF241000
slot:           8(Bytes) total:           0 used:           0 reqs:           0 fails:           0
slot:          16(Bytes) total:           0 used:           0 reqs:           0 fails:           0
```