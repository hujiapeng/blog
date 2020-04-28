1. Spring Boot2.0，新版中已经使用Lettuce作为Redis客户端，替换了之前的Jedis；
2. 与Redis建立连接，配置连接工厂LettuceConnectionFactory或JedisConnectionFactory，根据不同连接模式，做相应配置
 - 单机模式下：用RedisStandaloneConfiguration，主要是配置hostName、port、database和password；
 - 哨兵模式下：用RedisSentinelConfiguration，在配置文件中配置如
 ```
    spring.redis.sentinel.master=myMaster
	spring.redis.sentinel.nodes=127.0.0.1:23679,127.0.0.1:23680,127.0.0.1:23681
```
 - 集群模式下：用RedisClusterConfiguration，配置文件中配置如下
 ```
     spring.redis.cluster.nodes=127.0.0.1:23679,127.0.0.1:23680,127.0.0.1:23681
	 spring.redis.cluster.max-redirects=3
```
 只要连接上一个节点就可以获取到其他节点信息；注意最大尝试次数至少为3，默认即为3，因为集群中有些操作是需要重定向的。
3. Spring Boot中Redis的自动配置是用类RedisProperties，可以看到配置的前缀为spring.redis
4. Lettuce内置读写分离，支持操作有：
 - 只读主
 - 只读从
 - 优先读主
 - 优先读从

5. LettuceClient配置有以下三种，经常使用的是第三个
 - LettuceClientConfiguration
 - LettucePoolingClientConfiguration
 - LettuceClientConfigurationBuilderCustomizer

6. Spring Boot中操作Redis的RedisTemplate有两种
 - RedisTemplate<K,V>方式，这种是操作的对象，存到Redis里Key、Value是二进制数据
 - StringRedisTemplate方式，这种是操作的字符串，Key、Value都是字符串

7. 注意，对Redis的操作一定要设置一个**过期时间**。
8. RedisTemplate操作示例：
 - 借用Spring Bucks项目
 - pom.xml添加的依赖
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
```
 - application.properties配置如下
```
spring.jpa.hibernate.ddl-auto=none
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.format_sql=true
management.endpoints.web.exposure.include=*
spring.redis.host=www.ittutu.cn
spring.redis.lettuce.pool.maxActive=5
spring.redis.lettuce.pool.maxIdle=5
```
 - CoffeeService中引入RedisTemplate
```
@Service
@Slf4j
public class CoffeeService {
    private static final String CACHE = "springbucks-coffee";
    @Autowired
    private CoffeeRepository coffeeRepository;
    @Autowired
    private RedisTemplate<String, Coffee> redisTemplate;
    public Optional<Coffee> findOneCoffee(String name) {
        HashOperations<String, String, Coffee> hashOperations = redisTemplate.opsForHash();
        if (redisTemplate.hasKey(CACHE) && hashOperations.hasKey(CACHE, name)) {
            log.info("Get coffee {} from Redis.", name);
            return Optional.of(hashOperations.get(CACHE, name));
        }
        ExampleMatcher exampleMatcher = ExampleMatcher.matching().withMatcher("name", exact().ignoreCase());
        Optional<Coffee> coffee = coffeeRepository.findOne(Example.of(Coffee.builder().name(name).build(), exampleMatcher));
        log.info("Coffee Found: {}", coffee);
        if (coffee.isPresent()) {
            log.info("Put coffee {} to Redis.", name);
            hashOperations.put(CACHE, name, coffee.get());
            redisTemplate.expire(CACHE, 2, TimeUnit.MINUTES);
        }
        return coffee;
    }
}
```
 - 主类中要配置RedisTemplate Bean，及其调用测试方法如下
```
    @Bean
    public RedisTemplate<String, Coffee> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Coffee> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
······
    @Override
    public void run(ApplicationArguments args) throws Exception {
        Optional<Coffee> c = coffeeService.findOneCoffee("mocha");
        log.info("Coffee {}", c);

        for (int i = 0; i < 5; i++) {
            c = coffeeService.findOneCoffee("mocha");
        }

        log.info("Value from Redis: {}", c);
    }
```
 - 如果想自定义客户端调用方式，可配置LettuceClientConfigurationBuilderCustomizer Bean，如想优先读取主节点
```
    @Bean
    public LettuceClientConfigurationBuilderCustomizer customizer() {
        return builder -> builder.readFrom(ReadFrom.MASTER_PREFERRED);
    }
```
9. Redis Repository的使用：
 - 对实体使用注解：类上使用@RedisHash标识Redis实体，配置缓存名称及失效时间；实体就要有Id，注意Redis注解的Id为org.springframework.data.annotation.Id；如果想开启二级索引，可以使用注解@Indexd，其实二级索引就是在Redis里把缓存名称、Id、二级索引对应字段名称做成几个Key，然后根据各个Key里的值再查出结果
 - Spring Boot如何区分这些Repository？
    - 根据实体的注解，如JPA用@Entity，MongoDB用@Document，Reids用@RedisHash
    - 根据继承的接口类型，如JPA用JpaRepository接口，MongoDB用MongoRepository接口，Redis用CrudRepository接口
    - 也可以在主类或配置类上启用Repository注解时配置扫描包，如@EnableJpaRepositories("包路径")

 - 借用SpringBucks项目
 - pom.xml中添加依赖以及application.properties中配置如同RedisTemplate项目
 - model下创建Redis实体CoffeeCache
```
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    @RedisHash(value = "springbucks-coffee",timeToLive = 60)
    public class CoffeeCache {
        //注意@Id注解为springframework.data.annotation下的
        @Id
        private Long id;
        //开启另一个索引
        @Indexed
        private String name;
        private Money price;
    }
```
 - repository下创建CoffeeCacheRepository
```
    public interface CoffeeCacheRepository extends CrudRepository<CoffeeCache, Long> {
        Optional<CoffeeCache> findOneByName(String name);
    }
```
 - 类似mongodbRepository中也要有对Money类型转换，Redis中两个类型都要有转换器，在converter包下。RedisCustomConversions转换器的添加源码在RedisRepositoryConfigurationExtension.registerBeansForRoot中的```registerIfNotAlreadyRegistered customConversions,registry,REDIS_CUSTOM_CONVERSIONS_BEAN_NAME,configurationSource); ```
    - MoneyToBytesConverter转换器如下,注解@WritingConverter的意思是往Redis中写的时候做转换
```
    @WritingConverter
    public class MoneyToBytesConverter implements Converter<Money, byte[]> {
        @Override
        public byte[] convert(Money money) {
            String value = Long.toString(money.getAmountMinorLong());
            return value.getBytes(StandardCharsets.UTF_8);
        }
    }
```
    - BytesToMoneyConverter转换器如下，注解@ReadingConverter表示从Redis中读数时做转换
```
    @ReadingConverter
    public class BytesToMoneyConverter implements Converter<byte[], Money> {
        @Override
        public Money convert(byte[] bytes) {
            String value = new String(bytes, StandardCharsets.UTF_8);
            return Money.of(CurrencyUnit.of("CNY"), Long.parseLong(value));
        }
    }
```
 - 主类中添加注解@EnableRedisRepositories，配置Bean RedisCustomConversions，加入两个转换器，及其测试方法调用如下
```
    @Bean
    public RedisCustomConversions redisCustomConversions(){
        return new RedisCustomConversions(Arrays.asList(new BytesToMoneyConverter(),new MoneyToBytesConverter()));
    }
······
    @Override
    public void run(ApplicationArguments args) throws Exception {
        Optional<Coffee> c = coffeeService.findSimpleCoffeeFromCache("mocha");
        log.info("Coffee {}", c);
        for (int i = 0; i < 5; i++) {
            c = coffeeService.findSimpleCoffeeFromCache("mocha");
        }
        log.info("Value from Redis: {}", c);
    }
```
