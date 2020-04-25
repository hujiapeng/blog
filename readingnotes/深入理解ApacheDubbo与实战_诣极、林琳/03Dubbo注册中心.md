一、注册中心概述
1. Dubbo通过注册中心实现了分布式环境中各服务之间的注册与发现，是各个分布式节点之间的纽带
 - 动态加入：一个服务提供者通过注册中心可以动态地把自己暴露给其他消费者，无须消费者逐个去更新配置文件
 - 动态发现：一个消费者可以动态的感知新的配置、路由规则和新的服务提供者，无须重启服务使之生效
 - 动态调整：注册中心支持参数的动态调整，新参数自动更新到所有相关服务节点
 - 统一配置：避免了本地配置导致每个服务的配置不一致问题

2. 注册中心源码在模块dubbo-registry中，各模块介绍如下：
 - dubbo-registry-api：包含了注册中心的所有API和抽象实现类
 - dubbo-registry-zookeeper：使用Zookeeper作为注册中心的实现
 - dubbo-registry-redis：使用Redis作为注册中心的实现
 - dubbo-registry-default：Dubbo基于内存的默认实现
 - dubbo-registry-multicast：multicast模式的服务注册与发现

3. 注册中心工作流程
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/03Dubbo%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83/%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83%E6%80%BB%E4%BD%93%E6%B5%81%E7%A8%8B.jpg)
 - 服务提供者启动时，会向注册中心写入自己的元数据信息，同时会订阅配置元数据信息
 - 消费者启动时，也会向注册中心写入自己的元数据信息，并订阅服务提供者、路由和配置元数据信息
 - 服务治理中心(dubbo-admin)启动时，会同时订阅所有消费者、服务提供者、路由和配置元数据信息
 - 当有服务提供者离开或有新的服务提供者加入时，注册中心服务提供者目录会发生变化，变化信息会动态通知给消费者、服务治理中心
 - 当消费方发起服务调用时，会异步将调用、统计信息等上报给监控中心(dubbo-monitor-simle)

4. Zookeeper注册中心原理概述
 Zookeeper是树形结构的注册中心，每个节点的类型分为持久节点、持久顺序节点、临时节点和临时顺序节点。Dubbo使用Zookeeper作为注册中心时，只会创建持久节点和临时节点两种，对创建的顺序并没有要求
 Dubbo服务提供者在Zookeeper中的树形结构分为四层，root(根节点)、service(接口名称)、四种服务目录(包括providers、consumers、routers、configurations)和服务分类节点下具体Dubbo服务URL。服务路径示例如/dubbo/com.foo.BarService/providers
 树形结构的关系(元数据包括IP、端口、权重和应用名等数据)
 - 树的根节点时注册中心分组，下面有多个服务接口，分组值来自用户配置<dubbo:registry>中的group属性，默认是/dubbo
 - 服务接口下包含4类子目录，分别是providers、consumers、routers、configurators，这个路径是持久节点
 - 服务提供者目录(/dubbo/service/providers)下面包含的接口有多个服务者URL元数据信息
 - 服务消费者目录(/dubbo/service/consumers)下面包含的接口有多个消费者URL元数据信息
 - 路由配置目录(/dubbo/service/routers)下面包含多个用于消费者路由策略URL元数据信息
 - 动态配置目录(/dubbo/service/configurators)下面包含多个用于服务者动态配置URL元数据信息

 在Dubbo框架进行服务调用时，用户可以通过服务治理平台(dubbo-admin)下发路由配置。如果要在运行时改变服务参数，则用户可以通过服务治理平台(dubbo-admin)下发动态配置。服务器端会通过订阅机制收到属性变更，并重新更新已经暴露的服务
 服务元数据中的所有参数都是以键值对形式存储的。以服务元数据为例：dubbo://192.168.0.1:20880/com.alibaba.demo.Service?category=provider&name=demo-provider&······
 Dubbo中启用注册中心，配置在`<dubbo:registry protocol="zookeeper" address="">`，配置address时，多个集群间用数线(|)分隔，一个集群多个节点时，节点间用逗号分隔
5. Redis注册中心原理概述(运行测试环境时，注意：maven中引入dubbo-registry-redis、注释掉/etc/redis.conf中的bind 127.0.0.1、修改redis配置中protected-mode为no)
 Redis注册中心也沿用了Dubbo抽象的Root、Service、Type、URL四层结构。Redis属于NoSQL数据库，使用key/Map结构存储数据。Root、Service、Type组合成Redis的key，如/dubbo/com.alibaba.dubbo.demo.DemoService/providers；Redis的value是一个Map结构，URL作为Map的key，超时时间作为Map的value。Redis使用的是hash数据结构存储数据
 ```
127.0.0.1:6379> keys *
1) "/dubbo/com.alibaba.dubbo.demo.DemoService/providers"
127.0.0.1:6379> hgetall /dubbo/com.alibaba.dubbo.demo.DemoService/providers
1) "dubbo://192.168.2.104:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bean.name=com.alibaba.dubbo.demo.DemoService&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=4972&side=provider&timestamp=1570888961820"
2) "1570889834047"
127.0.0.1:6379> hget /dubbo/com.alibaba.dubbo.demo.DemoService/providers "dubbo://192.168.2.104:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bean.name=com.alibaba.dubbo.demo.DemoService&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=4972&side=provider&timestamp=1570888961820"
"1570889864050"
127.0.0.1:6379> type /dubbo/com.alibaba.dubbo.demo.DemoService/providers
hash
127.0.0.1:6379> 
 ```

二、订阅/发布(通过注册中心的变化，触发监听事件，让服务端或消费端更新本地缓存)
1. 当一个已有服务提供者下线，或者一个新的服务提供者节点加入微服务环境时，订阅对应接口的消费者和服务治理中心都能及时收到注册中心的通知，并更新本地的配置信息。后续的服务调用者就能避免调用已经下线的节点，或者能调用到新的节点
2. Zookeeper的实现
 - 发布实现：服务提供者和消费者都需要把自己注册到注册中心。服务提供者的注册是为了让消费者感知服务的存在，从而发起远程调用；也让服务治理中心感知有新的服务提供者上线。消费者的发布是为了让服务治理中心可以发现自己。Zookeeper发布代码就是通过Zookeeper客户端在注册中心上创建一个目录。取消发布就是Zookeeper在注册中心上把对应的路径删除。源码在ZookeeperRegistry中
 ```
    @Override
    protected void doRegister(URL url) {
        try {
            zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
    @Override
    protected void doUnregister(URL url) {
        try {
            zkClient.delete(toUrlPath(url));
        } catch (Throwable e) {
            throw new RpcException("Failed to unregister " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
 - 订阅的实现：订阅通常有pull和push两种方式，目前Dubbo采用的是第一次启动拉取方式，后续接收事件重新拉取数据。在服务暴露时，服务端会订阅configurations用于监听动态配置，在消费端启动时，消费端会订阅providers、routers和configurators三个目录，分别对应服务提供者、路由和动态配置变更通知。
 两种不同的Zookeeper开源客户端库ApacheCurator和zkClient，可以在<dubbo:registry>或RegistryConfig Bean中的client属性中设置curator、zkclient使用不同客户端，默认使用Curator客户端。
 Zookeeper注册中心采用的是"事件通知"+"客户端拉取"的方式，客户端在第一次连接上注册中心时，会获取对应目录下全量的数据。并在订阅的节点上注册一个watcher，客户端与注册中心之间保持TCP长连接，后续每个节点有任何数据变化的时候，注册中心会根据watcher的回调主动通知客户端(事件通知)，客户端接到通知后，会把对应节点下的全量数据都拉取过来(客户端拉取)。
 Zookeeper每个节点都有一个版本号，某个节点数据发生变化(即事务操作)时，该节点对应的版本号就会发生变化，并触发watcher事件，推送数据给订阅方。版本号强调的是变更次数，即使节点值没变，只要有更新操作，版本号就会变。
    - Dubbo服务治理平台启动时会订阅全量接口，后续通过监听事件进行更新。服务治理中心全量订阅时会将service设置成特殊值*，处理所有service层的订阅
    - 普通消费者订阅逻辑：首先根据URL的类别得到一组需要订阅的路径，如果类别是*，则会订阅四种类型路径(providers、routers、consumers、configurators)，否则只订阅providers路径
    - 订阅类别服务就是对URL中的category属性值获取具体的类别(providers、routers、consumers、configurators)，然后拉取直接子节点数据进行通知。如果是providers类别，则更新本地Directory管理的Invoker服务列表；如果是routers，则更新本地路由规则列表；如果是configuators，则更新本地动态参数列表
    本地文件可通过如下代码知存放位置
 ```
String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + 
url.getParameter(Constants.APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
```
3. Redis的实现
 ![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/03Dubbo%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83/Redis%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.jpg)
 - 总体流程：Redis订阅发布使用的是过期机制和publish/subscribe通道。服务提供者发布服务，首先会在Redis中创建一个Key，然后在通道中发布一条register事件消息。服务的Key写入Redis后，发布者需要周期性地刷新key过期时间，在RedisRegistry构造方法中会启动一个expireExecutor定时调度线程池，不断调用deferExpired()方法去延续key的超时时间或删除过期key。如果服务提供者宕机，没有续期，则key会因为超时而被Redis删除，服务也就被认定为下线。
 服务列表变化通过publish/subscribe通道广播对应事件消息，订阅方收到后再从注册中心拉取数据，更新本地缓存服务列表。
 由于Redis的key超时不会有动态消息推送，和Redis的publish/subscribe通道消息不可靠，订阅方无法知道服务发布方已下线，所以需要依赖服务治理中心定时调度，触发清理Redis上的无效key，并在通道发起对应key的unregister事件，这样消费者监听到取消注册事件后会删除对应服务数据，从而保证数据最终一致。
 Redis客户端初始化时，要先初始化Redis连接池jedisPools，此时如果配置注册中心的集群模式为<dubbo:registry cluster="replicate"/>，则服务提供者在发布服务的时候，需要同时向Redis集群中所有节点都写入，是多写的方式。但是读取还是从一个节点中读取。这种模式下，Redis集群可以不配置数据同步，一致性由客户端的多写来保证。如果cluster设置为failover或不设置，则只会读取和写入任意一个Redis节点，失败的话再尝试下一个Redis节点。这种模式需要Redis集群配置数据同步。
 - 发布的实现，服务提供者和消费者都使用注册功能，部分关键代码如下：
 ```
for (Map.Entry<String, JedisPool> entry : jedisPools.entrySet()) {
            //遍历连接池中所有节点
            JedisPool jedisPool = entry.getValue();
            try {
                Jedis jedis = jedisPool.getResource();
                try {
                    //向Redis中注册，并在通道中发布注册事件
                    jedis.hset(key, value, expire);
                    jedis.publish(key, Constants.REGISTER);
                    success = true;
                    if (!replicate) {
                        //如果Redis使用非replicate模式，只需要写一个节点，因此可以直接"break"；否则遍历所有节点，依次写入注册信息
                        break; //  If the server side has synchronized data, just write a single machine
                    }
                } finally {
                    jedis.close();
                }
            } catch (Throwable t) {
                exception = new RpcException("Failed to register service to redis registry. registry: " + entry.getKey() + ", service: " + url + ", cause: " + t.getMessage(), t);
            }
        }
 ```
 - 订阅的实现：服务消费者、服务提供者和服务治理中心都会使用注册中心的订阅功能。在订阅时，如果是首次订阅，则会先创建一个Notifier内部类，这是一个线程类，在启动时会异步进行通道的订阅。在启动Notifier线程的同时，主线程会继续往下执行，全量拉一次注册中心上所有的服务信息。后续注册中心上的信息变更则通过Notifier线程订阅的通道推送事件来实现。

三、缓存机制
1. 消费者或治理中心获取注册信息后会做本地缓存。内存中会有一份，保存在Properties对象里，磁盘上也会持久化一份文件，通过file对象引用(我的缓存地址在```C:\Users\Administrator\.dubbo ```)。内存中的缓存notified是ConcurrentHashMap里面又嵌套了一个Map，外层Map的key是消费者URL，内层Map的key是分类，包含providers、consumers、routes、configurators四种。value则是对应的服务列表，对于没有服务提供者提供服务的URL，以特殊的empty://前缀开头。
2. 缓存的加载：在服务初始化的时候，AbstractRegistry构造函数里会从本地磁盘文件中把持久化的注册数据读到Properties对象里，并加载到内存缓存中。Properties保存了所有服务提供者的URL，使用URL#serviceKey()作为key，提供者列表、路由规则列表、配置规则列表等作为value。由于value是列表，当存在多个的时候，使用空格隔开。还有个特殊的key.registies，保存所有的注册中心地址。
 如果应用启动过程中，注册中心无法连接或宕机，则Dubbo框架会自动通过本地缓存加载Invokers。
3. 缓存的保存与更新：缓存的保存又同步和异步两种方式。异步会使用线程池异步保存，如果线程在执行过程中出现异常，则会再次调用线程池不断重试。AbstractRegistry#doSaveProperties方法中进行保存，使用AtomicLong的版本号保证数据是最新的，在catch中实现保存失败后重试保存
4. 重试机制：FailbackRegistry抽象类中定义了一个ScheduledExecutorService，每经过固定间隔(默认5秒)调用FailbackRegistry#retry()方法。此抽象类五个重要集合，Set<URL>集合failedRegistered存储发起注册失败的URL集合、Set<URL>集合failedUnregistered存储取消注册失败的URL集合、ConcurrentMap<URL, Set<NotifyListener>>集合failedSubscribed存储发起订阅失败的监听器集合、ConcurrentMap<URL, Set<NotifyListener>>集合failedUnsubscribed存储取消订阅失败的监听器集合、ConcurrentMap<URL, Map<NotifyListener, List<URL>>>集合failedNotified存储通知失败的URL集合。retry方法中会把这五个集合分别遍历和重试，重试成功则从集合中移除。FailbackRegistry实现了subscribe、unsubscribe等通用方法，里面调用了未实现的模板方法，会由子类实现。通用方法会调用这些模板方法，发生异常就会把URL添加到对应集合中，供定时器去重试。

四、重试机制
 FailbackRegistry类中有一个定时任务线程池ScheduledExecutorService，就是通过这个线程池轮询failedRegistered集合中失败的地址

五、设计模式
1. 模板模式：整个注册中心逻辑部分使用了模板模式。抽象类AbstractRegistry实现了Registry接口中注册、订阅、查询、通知等方法，还实现了磁盘文件持久化注册信息这一通用方法。但是这些方法只是简单地把URL加入到对应的集合，没有具体逻辑。FailbackRegistry又继承了AbstractRegistry，重写了父类的注册、订阅、查询和通知等方法(重写的方法中调用了super中对应方法)，并且添加了重试机制。此外还添加了四个未实现的抽象模板方法，如下：
```
    // ==== Template method ====
    protected abstract void doRegister(URL url);
    protected abstract void doUnregister(URL url);
    protected abstract void doSubscribe(URL url, NotifyListener listener);
    protected abstract void doUnsubscribe(URL url, NotifyListener listener);
```
 具体的业务逻辑就靠子类实现，如``` ZookeeperRegistry extends FailbackRegistry ```
2. 工厂模式：所有注册中心实现都是通过对应工厂创建的。AbstractRegistryFactory实现了RegistryFactory接口的getRegistry(URL url)方法，主要完成加锁以及调用抽象模板方法createRegistry(URL url)创建具体实现等操作，并缓存在内存中。createRegistry方法由具体的子类实现。具体使用哪个具体类调用是依靠接口上@Adaptive({"protocol"})注解，当url.protocol=zookeeper时，获得ZookeeperRegistryFactory实现类。