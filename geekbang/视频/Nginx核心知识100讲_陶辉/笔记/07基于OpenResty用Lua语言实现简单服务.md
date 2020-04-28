1. 安装openResty。记得加上```--with-luajit```;server块下添加location块代码如下
```
        location /lua {
        
            default_type text/html;
            content_by_lua 'ngx.say("User-Agent:",ngx.req.get_headers()["User-Agent"])';
            
        }
```