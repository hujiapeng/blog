1. 高级特性概述
 - 服务分组和版本：支持同一个服务(Service接口)有多个分组和多个版本实现(一个接口多个实现，在@Service注解上添加组和版本)，用于服务强隔离、服务多个版本实现
 - 参数回调：当消费方调用服务提供方时，支持服务提供方能够异步回调到当前消费方，用于stub做热数据缓存等
 - 隐式参数：支持客户端隐式传递参数到服务端
 - 异步调用：并行发起多个请求，但只使用一个线程，用于业务请求非阻塞场景
 - 泛化调用：泛化调用主要用于消费端没有API接口的情况。不需要引入接口jar包，而是直接通过GenericService接口来发起服务调用。框架会自动把POJO对象转为Map，只要参数名能对应上即可。适合网关和跨框架集成等场景
 - 上下文信息：上下文中存放的是当前调用过程中所需的环境信息
 - Telnet操作：支持服务端调用、状态检查和跟踪服务调用统计等
 - Mock调用：用于方法调用失败时，构造Mock测试数据并返回
 - 结果缓存：结果缓存，用于加速热门数据的访问速度，Dubbo提供声明式缓存，以减少用户加缓存的工作量

2. 服务分组和版本
 - Dubbo中的服务分组和版本是**强隔离的**，如果服务方指定了服务分组和版本，则消费方调用也必须传递相同的分组名称和版本名称
 - 消费方会根据指定的分组和版本名称，对从注册中心拉取下来的所有服务列表进行一次过滤。源码在ZookeeperRegistry#toUrlsWithEmpty

3. 参数回调：当消费方调用服务方方法时，允许服务方在某个时间点回调消费方方法
 - 服务提供方之所以能回调服务消费方，那是因为服务消费方启动时，会拉取服务回调元数据，因为服务提供方配置了异步回调信息，所以这些信息会传给消费方。消费方会使用本地TCP暴露一个Dubbo协议的服务，供服务提供方回调
 - 准备两个接口，CallbackService用于提供正常服务，CallbackListener用于服务方回调
```
   public interface CallbackService {
       void addListener(String key, CallbackListener listener);
   }
   public interface CallbackListener {
       void change(String msg);
   }
```
 - 只提供CallbackService接口实现CallbackServiceImpl用于对外提供服务
```
   public class CallbackServiceImpl implements CallbackService {
       public void addListener(String key, CallbackListener listener) {
           listener.change("provider has");
       }
   }
```
 - provider xml配置如下
```
    <dubbo:service interface="com.hujiapeng.echo.api.CallbackService"
                   class="com.hujiapeng.echo.impl.CallbackServiceImpl" connections="1" callbacks="1000">
        <dubbo:method name="addListener">
            <dubbo:argument index="1" callback="true"/>
        </dubbo:method>
    </dubbo:service>
```
 - 消费方配置一个reference，然后就可以调用了，主类main方法中使用如下
```
    CallbackService callbackService = (CallbackService) context.getBean("callbackService");
    callbackService.addListener("testCallback", new CallbackListener() {
        public void change(String msg) {
            System.out.println(msg);
        }
    });
```
4. 隐式参数：在消费方使用RpcContext.getContext()#setAttachment方法设置隐式参数，在服务端RpcContext.getContext()#getAttachment方法中获取隐式参数。RpcContext是基于ThreadLocal实现的。
5. 异步调用(消费方)
 - 异步原理：消费方发起异步调用时，会在当前用户线程(RpcContext)中设置当前异步请求Future。在当前用户线程发起下一个远程调用前，会先保存在异步Future对象中。Dubbo会把异步请求对象保存在DefaultFuture类中，当服务端返回响应或者响应超时，被挂起的用户线程将被唤醒。可以在Futrue中获取到返回结果。
 - 只需要在消费方做配置
```
<dubbo:reference id="fooService" check="false" interface="com.hujiapeng.echo.api.FooService" async="true"/>
```
 - 代码实现
```
    fooService.getFoo(23);
    Future<Foo> fooFuture = RpcContext.getContext().getFuture();
    Foo foo = fooFuture.get();
```
6. 泛化调用：在不依赖消费方API包的场景下发起远程调用，这种特性适合框架集成和网关应用开发
 - 泛化调用必要参数包括应用名称、注册中心(或直连调用地址)、真实接口名称和泛化标识
 - 发起远程调用时，GenericService参数分别为方法名、方法参数类型签名和参数值
 - 注意每次动态创建的GenericService实例比较重，需要建立TCP连接，处理注册中心订阅和服务列表等计算，因此需要缓存ReferenceConfig对象
 - 服务端在处理泛化服务调用时，在GenericFilter拦截器中把RpcInvocation中传递过来的参数进行处理，找到真正的消费方方法，进行调用返回。注意返回的对象是Map类型
 - 泛化调用是消费方在没有服务提供方的API包时来实现服务调用的，所以只需要在消费方编写代码即可
```
//创建应用，指定应用名称
ApplicationConfig appConfig = new ApplicationConfig("demo-consumer");
//创建并配置注册中心
RegistryConfig registryConfig = new RegistryConfig();
registryConfig.setProtocol("zookeeper");
registryConfig.setAddress("www.ittutu.cn:2181");
//创建泛化ReferenceConfig对象
ReferenceConfig<GenericService> ref = new ReferenceConfig<GenericService>();
//设置应用
ref.setApplication(appConfig);
//设置注册中心
ref.setRegistry(registryConfig);
//设置协议
ref.setProtocol("dubbo");
//设置接口
ref.setInterface("com.hujiapeng.echo.api.FooService");
//标识泛化调用
ref.setGeneric(true);
//创建远程代理，注意对象创建比较重
GenericService genericService = ref.get();
//发起远程调用
Object result = genericService.$invoke("getFoo", new String[]{"java.lang.Integer"}, new Object[]{1});
```
 - 远程接口方法如下，用于远程调用时进行参考
```
   public interface FooService {
       Foo getFoo(Integer id);
   }
```
7. Telnet操作
 - ls 命令使用
    - ls 显示服务列表
    - ls -l 显示服务详细列表
    - ls HelloService 显示HelloService服务的方法列表
    - ls -l HelloService 显示HelloService服务的方法详细信息列表(包括参数类型和返回值)

 - ps 命令使用
    - ps 显示服务暴露的端口列表
    - ps -l 显示服务地址列表
    - ps 20880 显示端口上的连接信息
    - ps -l 20880 显示端口上的连接详细信息(客户端Ip和port，服务端Ip和port)

 - trace 命令使用
    - trace HelloService 跟踪1次HelloService服务任意方法的调用情况
    - trace HelloService 10 最多跟踪10次HelloService服务任意方法的调用情况
    - trace HelloService echo 跟踪1次HelloService服务echo方法的调用情况
    - trace HelloService echo 10 最多跟踪10次HelloService服务echo方法的调用情况

 - count 命令使用
    - count HelloService 统计1次HelloService服务任意方法的调用情况
    - count HelloService 10 最多统计10次HelloService服务任意方法的调用情况
    - count HelloService echo 统计1次HelloService服务echo方法的调用情况
    - count HelloService echo 10 最多统计10次HelloService服务echo方法的调用情况

8. Mock调用，当服务提供者调用抛出RpcException时，框架会降级到本地Mock伪装。如果不想发起RPC调用，直接使用Mock数据，则需要在配置中执行force:语法
 - ```<dubbo:reference mock="true" ······/> ```：mock类名称为接口名称+Mock
 - ```<dubbo:reference mock="com.foo.BarServiceMock" ······/> ```：mock到指定类
 - ```<dubbo:reference mock="return null" ······/> ```：触发mock时直接返回null，会吞掉异常信息
 - ```<dubbo:reference mock="throw com.alibaba.XXXException" ······/> ```：mock时抛出指定异常
 - ```<dubbo:reference mock="force:return fake" ······/> ```：不执行RPC调用，直接返回fake
 - ```<dubbo:reference mock="force:throw com.foo.MockException" ······/> ```：不执行RPC调用，直接抛出异常
 - ```<dubbo:reference mock="force:com.foo.BarServiceMock" ······/> ```：不执行RPC调用，直接mock到指定类

9. 结果缓存
```
<dubbo:reference cache="lru" ······/>
```