一、源码调试
1. 下载Dubbo 2.6.x分支源码。2.6.x代表2.6字版本中最新版本2.6.7
```
git clone -b 2.6.x https://github.com/apache/dubbo.git
```
2. Dubbo模块结构
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/01Dubbo-%E9%AB%98%E6%80%A7%E8%83%BDRPC%E9%80%9A%E4%BF%A1%E6%A1%86%E6%9E%B6/Dubbo%E6%A8%A1%E5%9D%97%E7%BB%93%E6%9E%84.jpg)
 - dubbo-common：通用逻辑模块，提供工具类和通用模型
 - dubbo-remoting：远程通信模块，为消费者和服务提供者提供通信能力
 - dubbo-rpc：容易和remote模块混淆，本模块抽象各种通信协议，以及动态代理
 - dubbo-cluster：集群容错模块，RPC只关心单个调用，本模块则包括负载均衡、集群容错、路由、分组聚合等
 - dubbo-registry：注册中心模块
 - dubbo-monitor：监控模块，监控dubbo接口的调用次数、时间等
 - dubbo-config：配置模块，实现了API配置、属性配置、XML配置、注解配置等功能
 - dubbo-container：容器模块，如果项目比较轻量，没用到Web特性，因此不想使用Tomcat等Web容器，则可以使用这个Main方法加载Spring的容器
 - dubbo-filter：过滤器模块，包含Dubbo内置的过滤器
 - dubbo-plugin：插件模块，提供内置的插件，如QoS
 - dubbo-demo：一个简单的远程调用示例模块
 - dubbo-test：测试模块，包含性能测试、兼容性测试等
3. 按照书本，本地先启动Provider，然后启动Consumer，可是启动Consumer报错
```
com.alibaba.dubbo.rpc.RpcException: Failed to invoke the method sayHello in the service com.alibaba.dubbo.demo.DemoService. No provider available for the service com.alibaba.dubbo.demo.DemoService from registry 224.5.6.7:1234 on the consumer 192.168.4.214 using the dubbo version . Please check if the providers have been started and registered.
```

二、基于XML配置实现案例
1. 先搭建一个单机的zookeeper(可参考[zookeeper部署](https://www.cnblogs.com/hujiapeng/p/9019103.html))，注意要关闭防火墙(可参考[CentOS7防火墙](https://www.cnblogs.com/hujiapeng/p/6511189.html))。下面启动Provider后，可通过登录zookeeper查看暴露的服务
```
zookeeper-3.4.12/bin/zkCli.sh -server 127.0.0.1:2181
ls /dubbo/com.hujiapeng.echo.api.EchoService/providers
```
2. Echo项目整体结构
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/02%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E6%AC%BEDubbo%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F/xml%E5%AE%9E%E7%8E%B0/echoDemo.jpg)
maven依赖配置
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.hujiapeng.dubbo-samples</groupId>
    <artifactId>echodubboxml</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    <modules>
        <module>echoapi</module>
        <module>echoservice</module>
        <module>echoclient</module>
    </modules>
    <dependencies>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.6.7</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo-qos</artifactId>
            <version>2.6.7</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo-registry-zookeeper</artifactId>
            <version>2.6.7</version>
        </dependency>
    </dependencies>
</project>
```
3. 创建EchoAPI接口项目
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/02%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E6%AC%BEDubbo%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F/xml%E5%AE%9E%E7%8E%B0/echoAPI.jpg)
EchoService接口
```
public interface EchoService {
    String echo(String message);
}
```
4. 创建Echo服务端项目
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/02%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E6%AC%BEDubbo%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F/xml%E5%AE%9E%E7%8E%B0/echoService.jpg) 
1)、maven依赖配置
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.hujiapeng.dubbo-samples</groupId>
        <artifactId>echodubboxml</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <artifactId>echoservice</artifactId>
    <dependencies>
        <dependency>
            <groupId>com.hujiapeng.dubbo-samples</groupId>
            <artifactId>echoapi</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
</project>
```
2)、EchoService接口实现
```
public class EchoServiceImpl implements EchoService {
    public String echo(String message) {
        String now = new SimpleDateFormat("HH:mm:ss").format(new Date());
        System.out.println("[" + now + "] Hello " + message + ", request from consumer: " +
                RpcContext.getContext().getRemoteAddress());
        return message;
    }
}
```
3)、echo-provider.xml配置
```
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
http://dubbo.apache.org/schema/dubbo
http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <!--服务提供方应用名称，方便用于依赖跟踪-->
    <dubbo:application name="echo-provider"/>
    <!--使用zookeeper作为注册中心-->
    <dubbo:registry address="zookeeper://192.168.6.47:2181"/>
    <!--使用dubbo协议并且指定监听20880端口-->
    <dubbo:protocol name="dubbo" port="20880"/>
    <!--通过XML方式把实现配置为Bean，让Spring托管和实例化-->
    <bean id="echoService" class="com.huiapeng.echo.impl.EchoServiceImpl"/>
    <!--声明要暴露的接口-->
    <dubbo:service interface="com.huiapeng.echo.api.EchoService" ref="echoService"/>
</beans>
```
4)、启动Dubbo服务
```
public class EchoProvider {
    public static void main(String[] args) throws Exception {
        //指定服务暴露配置文件
        String[] configFiles = new String[]{"spring/echo-provider.xml"};
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(configFiles);
        //启动Spring容器并暴露服务
        context.start();
        System.in.read();
    }
}
```
5. 终端使用Telnet模拟客户端调用，格式为[接口.方法]
```
telnet localhost 20880
```
回车一下会出现dubbo>命令提示符
```
dubbo>invoke com.huiapeng.echo.api.EchoService.echo("hello world!")
"hello world!"
elapsed: 8 ms.
dubbo>
```
6. 编写Echo客户端
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/02%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E6%AC%BEDubbo%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F/xml%E5%AE%9E%E7%8E%B0/echoClient.jpg)
1)、maven依赖配置
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.hujiapeng.dubbo-samples</groupId>
        <artifactId>echodubboxml</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <artifactId>echoclient</artifactId>
    <dependencies>
        <dependency>
            <groupId>com.hujiapeng.dubbo-samples</groupId>
            <artifactId>echoapi</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
</project>
```
2)、echo-consumer.xml配置
```
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
http://dubbo.apache.org/schema/dubbo
http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <!--服务消费方应用名称，方便于依赖跟踪-->
    <dubbo:application name="echo-consumer"/>
    <!--zookeeper作为注册中心-->
    <dubbo:registry address="zookeeper://192.168.6.47:2181"/>
    <!--指定要消费的服务-->
    <dubbo:reference id="echoService" check="false" interface="com.hujiapeng.echo.api.EchoService"/>
</beans>
```
3)、消费服务代码
```
public class EchoConsumer {
    public static void main(String[] args) {
        //配置文件
        String[] configFiles = new String[]{"spring/echo-consumer.xml"};
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(configFiles);
        context.start();
        EchoService echoService = (EchoService) context.getBean("echoService");
        String result = echoService.echo("hello world!");
        System.out.println("echo result: " + result);
    }
}
```

三、基于注解实现案例
1. 整体项目结构
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/02%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E6%AC%BEDubbo%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F/%E6%B3%A8%E8%A7%A3%E5%AE%9E%E7%8E%B0/echo%E6%95%B4%E4%BD%93%E7%BB%93%E6%9E%84.jpg)
echoAPI项目结构
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/02%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E6%AC%BEDubbo%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F/%E6%B3%A8%E8%A7%A3%E5%AE%9E%E7%8E%B0/echoAPI.jpg)
echoService项目结构
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/02%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E6%AC%BEDubbo%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F/%E6%B3%A8%E8%A7%A3%E5%AE%9E%E7%8E%B0/echoServer.jpg)
echoClient结构
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/02%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E6%AC%BEDubbo%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F/%E6%B3%A8%E8%A7%A3%E5%AE%9E%E7%8E%B0/echoClient.jpg)
2. 所有项目的pom.xml配置以及echoAPI项目和XML配置实现中的一样
3. echoService项目代码
1)、接口实现类，使用的@Service注解，注意@Service注解是dubbo的注解不是spring的
```
//通过注解方式，依靠Spring初始化来暴露服务
//注意注解路径(com.alibaba.dubbo.config.annotation.Service;)
@Service
public class EchoServiceImpl implements EchoService {
    public String echo(String message) {
        String now = new SimpleDateFormat("HH:mm:ss").format(new Date());
        System.out.println("[" + now + "] Hello " + message + ", request from consumer: " +
                RpcContext.getContext().getRemoteAddress());
        return message;
    }
}
```
2)、Provider启动类代码，其中包含对应xml中的配置
```
public class AnnotationProvider {
    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ProviderConfiguration.class);
        context.start();
        System.in.read();
    }
    @Configuration
    //指定扫描服务所在包
    @EnableDubbo(scanBasePackages = "com.hujiapeng.echo")
    static class ProviderConfiguration {
        //此刻还不知道这个Bean作用，注释掉也可正常使用
        @Bean
        public ProviderConfig providerConfig() {
            return new ProviderConfig();
        }
        @Bean
        public ApplicationConfig applicationConfig() {
            ApplicationConfig applicationConfig = new ApplicationConfig();
            applicationConfig.setName("echo-annotation-provider");
            return applicationConfig;
        }
        @Bean
        public RegistryConfig registryConfig() {
            RegistryConfig registryConfig = new RegistryConfig();
            registryConfig.setProtocol("zookeeper");
            registryConfig.setAddress("192.168.6.47");
            registryConfig.setPort(2181);
            return registryConfig;
        }
        @Bean
        public ProtocolConfig protocolConfig() {
            ProtocolConfig protocolConfig = new ProtocolConfig();
            protocolConfig.setName("dubbo");
            protocolConfig.setPort(20880);
            return protocolConfig;
        }
    }
}
```
4. echoClient项目代码
```
@Component
public class EchoConsumer {
    @Reference
    private EchoService echoService;
    public String echo(String name) {
        return echoService.echo(name);
    }
}
```
```
public class AnnotationConsumer {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ConsumerConfiguration.class);
        context.start();
        EchoConsumer echoService = context.getBean(EchoConsumer.class);
        String result = echoService.echo("hello world!");
        System.out.println("result: " + result);
    }
    @Configuration
    @EnableDubbo(scanBasePackages = "com.hujiapeng.echo")
    @ComponentScan(value = {"com.hujiapeng.echo"})
    static class ConsumerConfiguration {
        @Bean
        public ConsumerConfig consumerConfig() {
            return new ConsumerConfig();
        }
        @Bean
        public ApplicationConfig applicationConfig() {
            ApplicationConfig applicationConfig = new ApplicationConfig();
            applicationConfig.setName("echo-annotation-consumer");
            return applicationConfig;
        }
        @Bean
        public RegistryConfig registryConfig() {
            RegistryConfig registryConfig = new RegistryConfig();
            registryConfig.setProtocol("zookeeper");
            registryConfig.setAddress("192.168.6.47");
            registryConfig.setPort(2181);
            return registryConfig;
        }
    }
}
```

三、基于API实现案例
1. 整体项目结构，除了echoClient有点区别外，其他都一样
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/02%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E6%AC%BEDubbo%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F/API%E5%AE%9E%E7%8E%B0/echoClient.jpg)
2. echoService接口实现类
```
public class EchoServiceImpl implements EchoService {
    public String echo(String message) {
        String now = new SimpleDateFormat("HH:mm:ss").format(new Date());
        System.out.println("[" + now + "] Hello " + message + ", request from consumer: " +
                RpcContext.getContext().getRemoteAddress());
        return message;
    }
}
```
3. Provider启动类
```
public class EchoProvider {
    public static void main(String[] args) throws IOException {
        ServiceConfig<EchoService> service = new ServiceConfig<EchoService>();
        service.setApplication(new ApplicationConfig("java-echo-provider"));
        service.setRegistry(new RegistryConfig("zookeeper://192.168.6.47:2181"));
        //指定服务暴露接口
        service.setInterface(EchoService.class);
        //指定真实服务对象
        service.setRef(new EchoServiceImpl());
        //暴露服务
        service.export();
        System.out.println("java-echo-provider is running.");
        System.in.read();
    }
}
```
4. echoClient项目就一个EchoConsumer类
```
public class EchoConsumer {
    public static void main(String[] args) {
        ReferenceConfig<EchoService> reference = new ReferenceConfig<EchoService>();
        //设置消费方应用名称
        reference.setApplication(new ApplicationConfig("java-echo-consumer"));
        //设置注册中心地址和协议
        reference.setRegistry(new RegistryConfig("zookeeper://192.168.6.47:2181"));
        //指定要消费的服务接口
        reference.setInterface(EchoService.class);
        //创建远程连接并做动态代理转换
        EchoService echoService = reference.get();
        String result = echoService.echo("hello world!");
        System.out.println(result);
    }
}
```

