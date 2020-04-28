1. Druid连接池是阿里巴巴开源的数据库连接池项目。Druid连接池为监控而生，内置强大的监控功能，监控特性不影响性能。功能强大，能防SQL注入，内置Logging能诊断Hack应用行为。--官方介绍
2. Druid连接池使用中常见问题地址[https://github.com/alibaba/druid/wiki/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98](https://github.com/alibaba/druid/wiki/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)
3. Druid使用功能：
 - 详细的监控
 - ExceptionSorter，针对主流数据库的返回码都支持
 - SQL防注入
 - 内置加密配置
 - 众多扩展点、方便进行定制

4. 创建Spring Boot2.0项目，添加Web、Actuator、JDBC、H2、Lombok依赖。
5. pom.xml中添加依赖```druid-spring-boot-starter ```，并且注意要在```spring-boot-starter-jdbc ```中排除HikariCP，如下：
 ```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jdbc</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.zaxxer</groupId>
                    <artifactId>HikariCP</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.21</version>
        </dependency>
 ```
6. 自定义Druild扩展点，继承FilterEventAdapter类。重写父类方法即可，如在数据库连接前后打印日志信息：
 ```
@Slf4j
public class ConnectionLogFilter extends FilterEventAdapter {
    @Override
    public void connection_connectBefore(FilterChain chain, Properties info) {
        log.info("BEFORE CONNECTION!");
    }
    @Override
    public void connection_connectAfter(ConnectionProxy connection) {
        log.info("AFTER CONNECTION!");
    }
}
 ```
7. 在resources目录下创建文件```META-INF/druid-filter.properties ```，添加内容如下：
 ```
druid.filters.conn=com.hujiapeng.druiddemo.ConnectionLogFilter
 ```
8. application.properties配置文件中配置的druid中，配置```spring.datasource.druid.filters=conn ```如果还有其他扩展点，可以继续在后面追加，逗号分隔，如```spring.datasource.druid.filters=conn,config,stat,slf4j ```，其中config,stat,slf4j为Druid内置的扩展点。注意druid-filter.properties中的druid.filters.**conn**要和application.properties中的spring.datasource.druid.filters=**conn**对应。
9. application.properties中关于Druid整体配置如下
 ```
spring.datasource.url=jdbc:h2:mem:foo
spring.datasource.username=sa
spring.datasource.password=n/z7PyA5cvcXvs8px8FVmBVpaRyNsvJb3X7YfS38DJrIg25EbZaZGvH4aHcnc97Om0islpCAPc3MqsGvsrxVJw==
spring.datasource.druid.initial-size=5
spring.datasource.druid.max-active=5
spring.datasource.druid.min-idle=5
spring.datasource.druid.filters=conn,config,stat,slf4j
spring.datasource.druid.connection-properties=config.decrypt=true;config.decrypt.key=${public-key}
spring.datasource.druid.filter.config.enabled=true
spring.datasource.druid.test-on-borrow=true
spring.datasource.druid.test-on-return=true
spring.datasource.druid.test-while-idle=true
public-key=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALS8ng1XvgHrdOgm4pxrnUdt3sXtu/E8My9KzX8sXlz+mXRZQCop7NVQLne25pXHtZoDYuMh3bzoGj6v5HvvAQ8CAwEAAQ==
 ```
10. 主类中注入DataSource，并通过日志方式输出datasource。启动后，可以看到初始化DruidDataSource的时候，获取连接前后就会输出Demo中的日志。由于配置的Druid连接数为5，所以会有5次日志输出：
```
2020-04-08 16:19:14.983  INFO 5148 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2020-04-08 16:19:15.747  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : BEFORE CONNECTION!
2020-04-08 16:19:15.966  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : AFTER CONNECTION!
2020-04-08 16:19:16.004  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : BEFORE CONNECTION!
2020-04-08 16:19:16.005  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : AFTER CONNECTION!
2020-04-08 16:19:16.006  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : BEFORE CONNECTION!
2020-04-08 16:19:16.006  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : AFTER CONNECTION!
2020-04-08 16:19:16.007  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : BEFORE CONNECTION!
2020-04-08 16:19:16.007  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : AFTER CONNECTION!
2020-04-08 16:19:16.007  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : BEFORE CONNECTION!
2020-04-08 16:19:16.008  INFO 5148 --- [           main] c.h.druiddemo.ConnectionLogFilter        : AFTER CONNECTION!
2020-04-08 16:19:16.013  INFO 5148 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
```
也可以看到datasource有5个连接信息输出：
```
2020-04-08 16:19:52.289  INFO 5148 --- [           main] c.h.druiddemo.DruidDemoApplication       : {
	CreateTime:"2020-04-08 16:19:14",
	ActiveCount:0,
	PoolingCount:5,
	CreateCount:5,
	DestroyCount:0,
	CloseCount:0,
	ConnectCount:0,
	Connections:[
		{ID:1521986562, ConnectTime:"2020-04-08 16:19:15", UseCount:0, LastActiveTime:"2020-04-08 16:19:15"},
		{ID:1253827612, ConnectTime:"2020-04-08 16:19:16", UseCount:0, LastActiveTime:"2020-04-08 16:19:16"},
		{ID:1336758691, ConnectTime:"2020-04-08 16:19:16", UseCount:0, LastActiveTime:"2020-04-08 16:19:16"},
		{ID:1836786457, ConnectTime:"2020-04-08 16:19:16", UseCount:0, LastActiveTime:"2020-04-08 16:19:16"},
		{ID:181097736, ConnectTime:"2020-04-08 16:19:16", UseCount:0, LastActiveTime:"2020-04-08 16:19:16"}
	]
}
```