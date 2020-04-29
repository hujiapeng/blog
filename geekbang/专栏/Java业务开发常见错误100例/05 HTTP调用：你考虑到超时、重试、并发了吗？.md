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
        <dependency>
            <groupId>org.springframework.retry</groupId>
            <artifactId>spring-retry</artifactId>
        </dependency>
```
     - 添加一个配置文件default.properties，这个是Ribbon用来做负载均衡的
```
#如果多个服务器，用逗号分隔
clientsdk.ribbon.listOfServers=localhost:8080
```
     - 编写一个加载配置文件的工具
```
@Slf4j
public class Utils {
    public static void loadPropertySource(Class clazz,String fileName){
        try {
            Properties properties=new Properties();
            System.out.println(clazz.getResource("/"));
            properties.load(clazz.getResourceAsStream(fileName));
            properties.forEach((k,v)->{
                log.info("{}={}",k,v);
                System.setProperty(k.toString(),v.toString());
            });
        } catch (Exception ex) {
            ex.printStackTrace();
            throw new RuntimeException(ex);
        }
    }
}
```
     - 编写一个使用FeignClient的接口，注意注解中使用的name是服务的名字，即default.properties中的clientsdk
```
@FeignClient(name = "clientsdk")
public interface Client {
    @GetMapping("/feignandribbon/server")
    void server();
}
```
     - 直接使用Spring Boot Web，即作为服务端，也作为客户端
```
   @RestController
   @RequestMapping("feignandribbon")
   @Slf4j
   public class FeignAndRibbonController {
       @Autowired
       private Client client;
       @GetMapping("client")
       public void timeout(){
           long begin = System.currentTimeMillis();
           try{
               client.server();
           } catch (Exception ex) {
               log.warn("执行耗时：{}ms 错误：{}", System.currentTimeMillis() - begin, ex.getMessage());
           }
       }
       @GetMapping("server")
       public void server() throws Exception{
           System.out.println("has reached server");
           TimeUnit.SECONDS.sleep(10);
       }
   }
```
    - Spring Boot启动类
```
   @SpringBootApplication
   @EnableFeignClients(basePackages = "com.hujiapeng.httptimeoutdemo.feignandribbon")
   public class ApplicationDefault {
       public static void main(String[] args) {
           Utils.loadPropertySource(ApplicationDefault.class, "/default.properties");
           SpringApplication.run(ApplicationDefault.class, args);
       }
   }
```
    - 启动后访问：``` curl http://localhost:8080/feignandribbon/client ```发现如下问题
```
has reached server
has reached server
2020-04-29 12:32:17.106  WARN 17068 --- [nio-8080-exec-8] c.h.h.f.FeignAndRibbonController         : 执行耗时：2009ms 错误：Read timed out executing GET
```
 - 两个问题，一个是重复请求问题，另一个是网络超时问题
 - 先处理网络超时问题
    - 在default.properties中添加配置如下，即超过服务端处理所消耗时间。如果想测试配置是否生效，可以配置的时间短点，观察客户端异常的日志
```
#注意对于feign来说要同时设置readTimeout和connectTimeout才可以生效
#使用clientsdk，说明只对名称为clientsdk的服务生效，如果设置全局的要用default
feign.client.config.clientsdk.readTimeout=15000
feign.client.config.clientsdk.connectTimeout=15000
#feign.client.config.default.readTimeout=15000
#feign.client.config.default.connectTimeout=15000
```
    - FeignClient在执行```client.server() ```方法调用的时会调用LoadBalancerFeignClient#execute方法。其实带有@FeignClient注解的Client接口对应的@Autowired注解的属性client已经被Feign借助Spring注入了代理了。其中获取配置的代码就在execute方法中如下
```
IClientConfig requestConfig = getClientConfig(options, clientName);
······
	IClientConfig getClientConfig(Request.Options options, String clientName) {
		IClientConfig requestConfig;
		if (options == DEFAULT_OPTIONS) {
			requestConfig = this.clientFactory.getClientConfig(clientName);
		}
		else {
			requestConfig = new FeignOptionsClientConfig(options);
		}
		return requestConfig;
	}
```
    - 当options == DEFAULT_OPTIONS时，获取到的配置，是来自实现接口IClientConfig的一个Bean DefaultClientConfigImpl中。clientFactory.getClientConfig方法中就是通过Spring获取Bean的方法。DefaultClientConfigImpl Bean来自RibbonClientConfiguration类，RibbonClientConfiguration的加载可以追踪的RibbonAutoConfiguration的注解@RibbonClients。下面看DefaultClientConfigImpl Bean的生成。此处的DEFAULT_CONNECT_TIMEOUT和DEFAULT_READ_TIMEOUT就是在没有自定义配置的时候的默认时间，为1000
```
	@Bean
	@ConditionalOnMissingBean
	public IClientConfig ribbonClientConfig() {
		DefaultClientConfigImpl config = new DefaultClientConfigImpl();
		config.loadProperties(this.name);
		config.set(CommonClientConfigKey.ConnectTimeout, DEFAULT_CONNECT_TIMEOUT);
		config.set(CommonClientConfigKey.ReadTimeout, DEFAULT_READ_TIMEOUT);
		config.set(CommonClientConfigKey.GZipPayload, DEFAULT_GZIP_PAYLOAD);
		return config;
	}
```
    - 当配置了Feign的readTimeout和connectTimeout时，就到了方法requestConfig = new FeignOptionsClientConfig(options)，那就要追踪参数options是从哪来的了。options是来自创建FeignClient Bean实例时，可参考源码FeignClientFactoryBean#configureUsingProperties。从此处也可以看出同时设置readTimeout和connectTimeout才可以生效
```
······
		if (config.getConnectTimeout() != null && config.getReadTimeout() != null) {
			builder.options(new Request.Options(config.getConnectTimeout(),
					config.getReadTimeout()));
		}
······
```
     - 上面的配置是配置的Feign，下面通过配置Ribbon来测试超时问题，修改配置文件default.properties，注意配置的大小写，Ribbon和Feign有区别，Ribbon配置参考类CommonClientConfigKey中属性。
```
clientsdk.ribbon.ReadTimeout=12000
clientsdk.ribbon.ConnectTimeout=12000
#全局配置如下
#ribbon.ReadTimeout=12000
#ribbon.ConnectTimeout=12000
```
     - 开始以为最终获取配置还是LoadBalancerFeignClient#execute中的方法，但是获取不到Ribbon的配置，进入方法executeWithLoadBalancer，最后到方法FeignLoadBalancer#execute。其实参数IClientConfig已经将配置参数加载到dynamicProperties，是在创建Bean DefaultClientConfigImpl时完成的。下面方法做的就是，如果dynamicProperties里面不包含配置参数，就使用默认值，override.connectTimeout方法调用就是如此。
```
	@Override
	public RibbonResponse execute(RibbonRequest request, IClientConfig configOverride)
			throws IOException {
		Request.Options options;
		if (configOverride != null) {
			RibbonProperties override = RibbonProperties.from(configOverride);
			options = new Request.Options(override.connectTimeout(this.connectTimeout),
					override.readTimeout(this.readTimeout));
		}
······
```
     - 如果配置文件中Feign和Ribbon都配置了，最终会使用Feign的，原因可参考LoadBalancerFeignClient#getClientConfig，使用了new FeignOptionsClientConfig(options)
 - 接下来处理重复请求的问题，那是由于Ribbon重试机制导致的，先看触发重试的配置条件吧。实际使用中，应该配置clientsdk.ribbon.listOfServers为多个服务器，重试也是根据多服务器去重试
     - DefaultClientConfigImpl类中搜索MaxAutoRetriesNextServer可以追踪到默认配置为1，然后再看重试策略类RibbonLoadBalancedRetryPolicy中的重试条件如下
```
	public boolean canRetry(LoadBalancedRetryContext context) {
		HttpMethod method = context.getRequest().getMethod();
		return HttpMethod.GET == method || lbContext.isOkToRetryOnAllOperations();
	}
	@Override
	public boolean canRetrySameServer(LoadBalancedRetryContext context) {
		return sameServerCount < lbContext.getRetryHandler().getMaxRetriesOnSameServer()
				&& canRetry(context);
	}
	@Override
	public boolean canRetryNextServer(LoadBalancedRetryContext context) {
		return nextServerCount <= lbContext.getRetryHandler().getMaxRetriesOnNextServer()
				&& canRetry(context);
	}
```
     - 经过上面条件可知，禁止重试的方式有两个，第一种方式，是将请求方式改为HttpMethod.POST方式，即
```
@FeignClient(name = "clientsdk")
public interface Client {
    @PostMapping("/feignandribbon/server")
//    @GetMapping("/feignandribbon/server")
    void server();
}
······
    @PostMapping("server")
//    @GetMapping("server")
    public void server() throws Exception {
        System.out.println("has reached server");
        TimeUnit.SECONDS.sleep(10);
    }
```
     - 第二种方式为，配置MaxAutoRetriesNextServer为0
```
ribbon.MaxAutoRetriesNextServer=0
```
     - 参数OkToRetryOnAllOperations也是可以配置的，默认为false，自己猜测是因为根据REST服务规范，Get请求可以保证幂等，所以重试没有问题。POST、Put等非幂等的，如果开启这个参数，就要考虑到幂等性问题
     - Ribbon的重试机制用了Spring Cloud的Reactive模式，响应式编程，Observable发布消费模式，可参考LoadBalancerCommand中的如下代码
```
        if (maxRetrysNext > 0 && server == null) 
            o = o.retry(retryPolicy(maxRetrysNext, false));
```
4. 并发限制了爬虫的抓取能力
