1. 开发中使用HTTP请求时，要考虑如下三点：
 - 框架设置的默认超时是否合理；
 - 由于网络不稳定，可以使用重试，但要考虑到服务端接口的幂等性是否允许重试；
 - 考虑是否要像浏览器那样限制最大并发数，以免在服务并发量大的情况下，HTTP调用的并发数限制成为瓶颈。

2. 几乎所有网络框架都会提供如下两个超时参数：
 - 连接超时参数ConnectionTimeout，用来配置建连阶段的最长等待时间；一般来说，TCP三次握手建立连接需要的时间非常短，通常毫秒级，多的也就秒级，可以配置短些，如1~5秒即可，如果内网可以配置的更短，太长了没有意义，如果几秒内都连不上，那就是连不上了，要排查网络、防火墙及系统问题。
 连接超时要理清连接的是哪里，尤其对负载均衡这样的代理，要明白是客户端到代理超时，还是代理到服务端超时。
 - 读取超时参数ReadTimeout，用来控制从Socket上读取数据的最长等待时间。
    - 误区一：读取超时就是服务端异常。读取超时报错不能简单的任务服务端错误，有可能是客户端设置的读取超时参数太短；
    - 误区二：读取时间是Socket网络层面时间。不能简单的认为读取超时就是Socket网络上消耗的时间，还包括服务端业务逻辑处理时间；
    - 误区三：超时时间设置越长越好。一般对于定时任务或异步任务来说，读取超时配置的长些问题不大。但是面向用户响应的请求或微服务短平快的同步接口调用，并发量一般很大，应该设置一个比较短的读取超时时间，以防止被下游服务拖慢，通常设置不会超过30秒。

3. Spring Boot中Feign和Ribbon配置使用中的问题
 - 先通过一个案例测试服务端超时的问题
    - 创建Spring Cloud项目，Spring Cloud版本为```<spring-cloud.version>Greenwich.SR4</spring-cloud.version> ```，Spring Boot版本为```2.2.6.RELEASE ```,引入的主要依赖
```
       <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>fluent-hc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-httpclient</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>
```
     - 