1. 目前Spring Data Redis中只有Lettuce支持Reactive方式
2. Spring Data Redis中的主要支持，即可以直接注入的Bean
 - ReactiveRedisConnection
 - ReactiveRedisConnectionFactory
 - ReactiveRedisTemplate
3. 创建测试项目，项目中使用的是ReactiveStringRedisTemplate，由于Spring没有创建该Bean，所以要自己创建
 - pom.xml中依赖
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
        <dependency>
            <groupId>org.joda</groupId>
            <artifactId>joda-money</artifactId>
            <version>1.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.jadira.usertype</groupId>
            <artifactId>usertype.core</artifactId>
            <version>6.0.1.GA</version>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
```
 - resources下schema.sql文件
```
drop table t_coffee if exists;
create table t_coffee (
    id bigint auto_increment,
    create_time timestamp,
    update_time timestamp,
    name varchar(255),
    price bigint,
    primary key (id)
);
```
 - resources下data.sql文件
```
insert into t_coffee (name, price, create_time, update_time) values ('espresso', 2000, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('latte', 2500, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('capuccino', 2500, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('mocha', 3000, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('macchiato', 3000, now(), now());
```
 - resource下application.properties文件
```
spring.redis.host=www.ittutu.cn
```
 - Coffee.java代码
```
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public class Coffee {
        private Long id;
        private String name;
        private Long price;
    }
```
 - 主类中代码，注意注入的ReactiveStringRedisTemplate，还有创建Bean
```
    private static final String KEY = "COFFEE_MENU";
    @Autowired
    private JdbcTemplate jdbcTemplate;
    @Autowired
    private ReactiveStringRedisTemplate redisTemplate;
    @Bean
    public ReactiveStringRedisTemplate reactiveStringRedisTemplate(ReactiveRedisConnectionFactory factory) {
        return new ReactiveStringRedisTemplate(factory);
    }
······
    @Override
    public void run(ApplicationArguments args) throws Exception {
        ReactiveHashOperations<String, String, String> hashOps = redisTemplate.opsForHash();
        CountDownLatch downLatch = new CountDownLatch(1);
        List<Coffee> list = jdbcTemplate.query("select * from T_COFFEE", (rs, i) ->
                Coffee.builder()
                        .id(rs.getLong("id"))
                        .name(rs.getString("name"))
                        .price(rs.getLong("price"))
                        .build()
        );
        Flux.fromIterable(list)
                .publishOn(Schedulers.single())
                .doOnComplete(() -> log.info("list ok"))
                .flatMap(c -> {
                    log.info("try to put {},{}", c.getName(), c.getPrice());
                    return hashOps.put(KEY, c.getName(), c.getPrice().toString());
                })
                .doOnComplete(() -> log.info("set ok"))
                .concatWith(redisTemplate.expire(KEY, Duration.ofMinutes(1)))
                .doOnComplete(() -> log.info("expire ok"))
                .onErrorResume(e -> {
                    log.error("exception {}", e.getMessage());
                    return Mono.just(false);
                })
                .subscribe(b -> log.info("Boolean: {}", b),
                        e -> log.error("Exception {}", e.getMessage()),
                        () -> downLatch.countDown());
        log.info("Waiting");
        downLatch.await();
    }
```