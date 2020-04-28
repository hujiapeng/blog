1. Jedis客户端使用：
 - Jedis不是线程安全的；
 - 通过JedisPool获得Jedis实例；
 - 通过JedisPool取得Jedis实例后，使用Jedis中方法操作

2. 通过Docker启动Redis
 - 获取镜像：```docker pull redis ```
 - 启动Redis：```docker run --name redis -d -p 6379:6379 redis ```
 - 进入Redis容器：```docker exec -it redis bash ```
 - 启动Redis客户端：```redis-cli ```
 - 注意：如果key包含空格等特殊字符，要使用双引号包括下
 - keys * 查出所有Key后，使用type Key命令查看Key类型，便于使用相应命令查询Key Value

3. 在SpringBucks项目上继续扩展
 - application.properties中的配置：
 ```
spring.jpa.hibernate.ddl-auto=none
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.format_sql=true
redis.host=www.ittutu.cn
redis.maxTotal=5
redis.maxIdle=5
redis.testOnBorrow=true
#驼峰式变量和短横分隔变量配置一样
#redis.max-total=3
#redis.max-idle=3
#redis.test-on-borrow=true
```
 - 主类中代码如下：
 ```
@SpringBootApplication
@Slf4j
@EnableTransactionManagement
@EnableJpaRepositories
public class RedisDemoApplication implements ApplicationRunner {

    @Autowired
    private CoffeeService coffeeService;
    @Autowired
    private JedisPool jedisPool;
    @Autowired
    private JedisPoolConfig jedisPoolConfig;

    @Bean
    @ConfigurationProperties("redis")
    public JedisPoolConfig jedisPoolConfig() {
        return new JedisPoolConfig();
    }

    @Bean(destroyMethod = "close")
    public JedisPool jedisPool(@Value("${redis.host}") String host) {
        return new JedisPool(jedisPoolConfig(), host);
    }

    public static void main(String[] args) {
        SpringApplication.run(RedisDemoApplication.class, args);
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        log.info(jedisPoolConfig.toString());
        try (Jedis jedis = jedisPool.getResource()) {
            coffeeService.findAllCoffee().forEach(coffee -> {
                jedis.hset("springbucks-menu", coffee.getName(),
                        Long.toString(coffee.getPrice().getAmountMinorLong()));
            });
            Map<String, String> menu = jedis.hgetAll("springbucks-menu");
            log.info("Menu: {}", menu);
            String price = jedis.hget("springbucks-menu", "espresso");
            log.info("espresso - {}",
                    Money.ofMinor(CurrencyUnit.of("CNY"), Long.parseLong(price)));
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
}
```