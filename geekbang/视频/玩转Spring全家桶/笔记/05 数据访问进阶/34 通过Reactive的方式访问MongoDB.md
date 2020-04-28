1. MongoDB官方提供了支持Reactive的驱动：mongodb-driver-reactivestreams
2. Spring Data MongoDB中的主要支持
 - ReactiveMongoClientFactory
 - ReactiveMongoDatabaseFactory
 - ReactiveMongoTemplate

3. 创建测试项目
 - application.properties
```
spring.data.mongodb.uri=mongodb://springbucks:springbucks@www.ittutu.cn:27017/springbucks
```
 - model下Coffee.java
```
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public class Coffee {
        private String id;
        private String name;
        private Money price;
        private Date createTime;
        private Date updateTime;
    }
```
 - convert下MoneyReadConverter.java和MoneyWriteConverter.java
```
    public class MoneyReadConverter implements Converter<Long, Money> {
        @Override
        public Money convert(Long aLong) {
            return Money.ofMinor(CurrencyUnit.of("CNY"), aLong);
        }
    }
```
```
    public class MoneyWriteConverter implements Converter<Money, Long> {
        @Override
        public Long convert(Money money) {
            return money.getAmountMinorLong();
        }
    }
```
 - 主类代码如下
```
@SpringBootApplication
@Slf4j
public class MongoDemoApplication implements ApplicationRunner {

    @Autowired
    private ReactiveMongoTemplate mongoTemplate;
    private CountDownLatch downLatch;

    @Bean
    public MongoCustomConversions mongoCustomConversions() {
        return new MongoCustomConversions(
                Arrays.asList(new MoneyReadConverter(),
                        new MoneyWriteConverter()));
    }

    public static void main(String[] args) {
        SpringApplication.run(MongoDemoApplication.class, args);
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
//		startFromInsertion(() -> log.info("Runnable"));
        startFromInsertion(() -> {
            log.info("Runnable");
            decreaseHighPrice();
        });
        log.info("after starting");
        downLatch.await();
    }

    private void startFromInsertion(Runnable runnable) {
        mongoTemplate.insertAll(initCoffee())
                .publishOn(Schedulers.elastic())
                .doOnNext(c -> log.info("Next: {}", c))
                .doOnComplete(runnable)
                .doFinally(s -> {
                    downLatch.countDown();
                    log.info("Finnally 1, {}", s);
                })
                .count()
                .subscribe(c -> log.info("Insert {} records", c));
    }

    private List<Coffee> initCoffee() {
        Coffee espresso = Coffee.builder()
                .name("espresso")
                .price(Money.of(CurrencyUnit.of("CNY"), 20.0))
                .createTime(new Date())
                .updateTime(new Date())
                .build();
        Coffee latte = Coffee.builder()
                .name("latte")
                .price(Money.of(CurrencyUnit.of("CNY"), 30.0))
                .createTime(new Date())
                .updateTime(new Date())
                .build();
        return Arrays.asList(espresso, latte);
    }

    private void decreaseHighPrice() {
        mongoTemplate.updateMulti(query(where("price").gte(3000L)),
                new Update().inc("price", -500L)
                        .currentDate("updateTime"), Coffee.class)
                .doFinally(s -> {
                    downLatch.countDown();
                    log.info("Finnally 2, {}", s);
                })
                .subscribe(r -> log.info("Result is {}", r));
    }
}
```