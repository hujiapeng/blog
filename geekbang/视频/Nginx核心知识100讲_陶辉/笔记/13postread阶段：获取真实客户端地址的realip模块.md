1. TCP连接四元组有src ip、src port、dst ip、dst port。如果只是单纯的客户端与服务器连接，中间不存在代理，那么src ip就是用户IP。
2. HTTP请求如果中间有代理层，如CDN，nginx获取真实客户端IP的话，那就需要借助HTTP头部两个字段X-Forward-For和X-Real-IP。
 - X-Real-IP字段是nginx独有的，也就是说使用nginx代理的才会通过X-Real-IP获取用户真实IP，X-Real-IP字段始终存储真实客户端IP，没有代理层IP；
 - X-Forward-For字段是RFC规范中的，也就是只要按照规范的代理服务器都会将请求IP(即，直接客户端IP)拼接到X-Forward-For字段中，X-Forward-For字段中存储的是从客户端到最后一层代理服务器IP的集合。

3. nginx通过变量remote_addr获取客户端真实IP，需要引入real-ip模块```--with-http_realip_module```，这个模块作用就是根据配置使用X-Forward-For还是X-Real-IP字段来设置remote_addr变量值。模块中相关指令如下
 - real_ip_header：默认值是X-Real-IP，即从HTTP头部X-Real-IP中获取真实用户IP；还可以设置为X-Forward-For，即从HTTP头部X-Forward-For中获取真实用户IP。
 - set_real_ip_from：这个是设置可信IP，也就是代理层的IP或者集群中的IP，如果是多层，可设置多个；如果是区间的，可以设置子网掩码代表。
 - real_ip_recursive：是否开启环回，默认off，关闭，如果打开设置为on。如果关闭，则从HTTP头部X-Forward-For字段中最右边开始取第一个作为用户真实IP；如果开启，则从X-Forward-For字段中最右边开始遍历查找，找到第一个没有在set_real_ip_from中设置过的IP就是用户真实IP。

4. 配置案例如下，
```
server {
    listen 8081;
    server_name localhost;
    set_real_ip_from 192.168.2.210;
    set_real_ip_from 2.2.2.2;
    #real_ip_header X-Real_IP;
    real_ip_header X-Forward-For;
    real_ip_recursive off;
    #real_ip_recursive on;
    location /{
        return 200 "Client real ip: $remote_addr\n";    
    }
}
```
real_ip_header为X-Real_IP时，不管下面什么配置，就是获取域名对应的IP，即真实用户IP，只是这个是nginx独有的。
real_ip_header为X-Forward-For时，请求地址如``` curl -H 'X-Forward-For:1.1.1.1,2.2.2.2,192.168.2.210' hjp:8081/ ```real_ip_recursive为off时，remote_addr为192.168.2.210，real_ip_recursive为on时，remote_addr为1.1.1.1。注意X-Forward-For比较依赖set_real_ip_from