1. Nginx进程间的通讯方式(主要是跨worker进程间的通讯)：
 - 基础同步工具：信号、共享内存，多个worker进程可以同时操作共享内存，包括读取和写入。
 - 高级通讯方式：为了保证共享内存数据安全，就需要使用锁，Nginx中使用的是自旋锁来保护共享内存区数据。Nginx使用Slab内存管理器来管理内存。
2. 共享内存使用者：
 - rbtree(红黑树)：使用红黑树数据结构的某些nginx模块
 - 单链表
 - Ngx_http_lua_api：这个模块是openResty核心模块。使用中可以使用lua_shared_dict分配一块指定大小的内存块。

3. openResty共享内存代码示例，其中使用到了红黑树和单链表，红黑树用来快速增删查数据，由于内存大小有限，使用单链表来实现LRU数据淘汰策略。
```
http {
    ······
    lua_shared_dict dogs 10m;
    ······
    server {
        ······
        location /set {
            default_type text/html;
            content_by_lua_block {
                local dogs=ngx.shared.dogs
                dogs:set("Jim",8)
                ngx.say("STORED")
         }    
        }

        location /get {
            default_type text/html;
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