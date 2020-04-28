1. 使用Spring缓存
 - 还是借用Spring Bucks项目
 - 修改CoffeeService类，使用缓存，代码如下
```
    @Service
    @Slf4j
    @CacheConfig(cacheNames = "coffee")
    public class CoffeeService {
        @Autowired
        private CoffeeRepository coffeeRepository;
        @Cacheable
        public List<Coffee> findAllCoffee() {
            return coffeeRepository.findAll();
        }
        @CacheEvict
        public void reloadCoffee() {
        }
        public Optional<Coffee> findOneCoffee(String name) {
            ExampleMatcher matcher = ExampleMatcher.matching().withMatcher("name", exact().ignoreCase());
            Optional<Coffee> coffee = coffeeRepository.findOne(Example.of(Coffee.builder().name(name).build(), matcher));
            log.info("Coffee Found: {}", coffee);
            return coffee;
        }
    }
```
 - 主类中配置，及测试，注意启用缓存注解@EnableCaching(proxyTargetClass = true)表示使用Cglib代理作AOP缓存拦截
```
    @SpringBootApplication
@Slf4j
@EnableCaching(proxyTargetClass = true)
//@EnableTransactionManagement
//@EnableJpaRepositories
public class CacheDemoApplication implements ApplicationRunner {
    @Autowired
    private CoffeeService coffeeService;
    public static void main(String[] args) {
        SpringApplication.run(CacheDemoApplication.class, args);
    }
    @Override
    public void run(ApplicationArguments args) throws Exception {
        log.info("Count: {}", coffeeService.findAllCoffee().size());
        for (int i = 0; i < 10; i++) {
            log.info("Reading from cache.");
            coffeeService.findAllCoffee();
        }
        coffeeService.reloadCoffee();
        log.info("Reading after refresh.");
        coffeeService.findAllCoffee().forEach(c -> log.info("Coffee {}", c.getName()));
    }
}
```
2. 通过Spring Boot配置Redis缓存
 - 借用上面Spring缓存项目
 - pom.xml中引入依赖
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```
 - application.properties中配置如下
```
    spring.jpa.hibernate.ddl-auto=none
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.format_sql=true
management.endpoints.web.exposure.include=*
spring.cache.type=redis
spring.cache.cache-names=coffee
spring.cache.redis.time-to-live=5000
#是否缓存null值
spring.cache.redis.cache-null-values=false
#默认端口号6379，如果其他端口号，直接在后面写如，:8888
spring.redis.host=www.ittutu.cn
```
 - 主类中主要测试方法如下
```
    @Override
    public void run(ApplicationArguments args) throws Exception {
        log.info("Count: {}", coffeeService.findAllCoffee().size());
        for (int i = 0; i < 5; i++) {
            log.info("Reading from cache.");
            coffeeService.findAllCoffee();
        }
        Thread.sleep(5000);
        //下面的清除缓存方法依然生效
        //coffeeService.reloadCoffee();
        log.info("Reading after refresh.");
        coffeeService.findAllCoffee().forEach(c -> log.info("Coffee {}", c.getName()));
    }
```