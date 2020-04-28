1. 拉取MongoDB镜像
 ```[root@ittutu ~]# docker pull mongo ```
2. 启动MongoDB镜像，并指定端口及文件路径映射，通过环境变量指定用户名密码
 ```[root@ittutu ~]# docker run --name mongo -p 27017:27017 -v ~/docker-data/mongo:/data/db -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=admin -d mongo ```
3. 进入mongodb容器
```[root@ittutu ~]# docker exec -it mongo bash ```
4. 登录mongodb
 ```root@a8f379cc791b:/# mongo -u admin -p admin ```
5. 查看mongodb当前所在库和拥有库
```
> db
test
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
```
6. 创建一个springbucks库(库名不能带有短横)，直接use，如果有继续操作，mongodb会自动创建。在当前库执行``` db.dropDatabase() ```就会删除当前库；查看当前库中用户，执行```show users ```；删除当前库中用户执行```db.dropUser("user") ```。
```
> use springbucks
switched to db springbucks
>  db.createUser(
... ... {
... ... user:"springbucks",
... ... pwd:"springbucks",
... ... roles:[
... ... {role:"readWrite",db:"springbucks"}
... ... ]
... ... }
... ... )
Successfully added user: {
        "user" : "springbucks",
        "roles" : [
                {
                        "role" : "readWrite",
                        "db" : "springbucks"
                }
        ]
}
> 
```
7. Mongodb基本使用，先使用use db切换到要操作的库
 - show collections：查看当前库下集合(文档)
 - db.coltest.insert({"name":"hjp"})：在集合coltest下插入一条记录
 - db.coltest.find().pretty()：查询当前集合下所有数据，并格式化
 - db.coltest.remove({"name":"espresso"})：根据name删掉集合中数据
 - db.coltest.drop()：删掉集合coltest
8. 创建Spring Boot，添加主要依赖如下
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
        <dependency>
            <groupId>org.joda</groupId>
            <artifactId>joda-money</artifactId>
            <version>RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
```
9. application.properties配置如下
 ```
spring.data.mongodb.uri=mongodb://springbucks:springbucks@www.ittutu.cn:27017/springbucks
```
10. model下添加Coffee类
```
   @Document
   @Data
   @NoArgsConstructor
   @AllArgsConstructor
   @Builder
   public class Coffee {
       @Id
       private String id;
       private String name;
       private Money price;
       private Date createTime;
       private Date updateTime;
   }
```
11. convert下创建MoneyReadConverter类，用于做mongodb中获取到的money数据转换为Money类型对象，之所以没有做Money类型对象转换为mongodb所需数据类型，是因为Money对象序列化为bson类型数据后可以完整存储到Mongodb中。
```
   public class MoneyReadConverter implements Converter<Document, Money> {
    @Override
    public Money convert(Document document) {
        Document money = (Document) document.get("money");
        double amount = Double.parseDouble(money.getString("amount"));
        String currency = ((Document) money.get("currency")).getString("code");
        return Money.of(CurrencyUnit.of(currency), amount);
    }
  }
```
12. 主类中注入MongoTemplate属性，并配置Bean且设置Money类型转换器，如下
```
@Autowired
private MongoTemplate mongoTemplate;
    @Bean
    public MongoCustomConversions mongoCustomConversions(){
        return new MongoCustomConversions(Arrays.asList(new MoneyReadConverter()));
    }
```
13. 主类中增删改查操作如下
 ```
       public void run(ApplicationArguments args) throws Exception {
        Coffee espresso = Coffee.builder()
                .name("espresso")
                .price(Money.of(CurrencyUnit.of("CNY"), 20.0))
                .createTime(new Date())
                .updateTime(new Date()).build();
        Coffee saved = mongoTemplate.save(espresso);
        log.info("Coffee {}", saved);

        List<Coffee> list = mongoTemplate.find(Query.query(where("name").is("espresso")), Coffee.class);
        log.info("Find {} Coffee", list.size());
        list.forEach(c -> log.info("Coffee {}", c));

        Thread.sleep(1000); // 为了看更新时间
        UpdateResult result = mongoTemplate.updateFirst(Query.query(where("name").is("espresso")),
                new Update().set("price", Money.ofMajor(CurrencyUnit.of("CNY"), 30)).currentDate("updateTime"), Coffee.class);
        log.info("Update Result: {}", result.getModifiedCount());

        Coffee updateOne = mongoTemplate.findById(saved.getId(), Coffee.class);
        log.info("Update Result: {}", updateOne);

        mongoTemplate.remove(updateOne);
    }
```
14. 通过源码查看Spring Boot是如何配置mongo的，以及如何想到添加MongoCustomConversions
 - 在spring-boot-autoconfigure对应的jar包下，找到MongoDataAutoConfiguration类，该类注解引入MongoDbFactoryDependentConfiguration类
 - 进入MongoDbFactoryDependentConfiguration类，找到MongoTemplate Bean的创建，里面要传入MongoConverter
 - 然后找到创建MappingMongoConverter Bean方法，MappingMongoConverter是MongoConverter的子类，创建该Bean的时候要传入MongoCustomConversions
 - 找到MongoCustomConversions Bean创建方法，在MongoDataConfiguration类中，如下
 ```
	@Bean
	@ConditionalOnMissingBean
	MongoCustomConversions mongoCustomConversions() {
		return new MongoCustomConversions(Collections.emptyList());
	}
```
 代码意思是，如果不存在MongoCustomConversions Bean就创建一个空集合对应的MongoCustomConversions Bean；所以如果自己代码中创建了该Bean，就用自己的了。
 - 通过如上案例，要多看下Spring Boot的自动配置，便于开发

15. 使用MongoRepository操作mongodb
 - application.properties、Coffee和MoneyReadConverter和之前一样
 - 创建Repository
```
    public interface CoffeeRepository extends MongoRepository<Coffee, String> {
        List<Coffee> findByName(String name);
    }
```
 - 主类中注入CoffeeRepository，并且加入MoneyReadConverter类型转换器
```
    @Autowired
    private CoffeeRepository coffeeRepository;
    @Bean
    public MongoCustomConversions mongoCustomConversions() {
        return new MongoCustomConversions(Arrays.asList(new MoneyReadConverter()));
    }
```
 - 主要增删改操作
```
    Coffee espresso = Coffee.builder()
                .name("espresso")
                .price(Money.of(CurrencyUnit.of("CNY"), 20.0))
                .createTime(new Date())
                .updateTime(new Date()).build();
        Coffee latte = Coffee.builder()
                .name("latte")
                .price(Money.of(CurrencyUnit.of("CNY"), 30.0))
                .createTime(new Date())
                .updateTime(new Date()).build();
        coffeeRepository.insert(Arrays.asList(espresso, latte));
        coffeeRepository.findAll(Sort.by("name")).forEach(coffee -> log.info("Saved Coffee {}", coffee));

        Thread.sleep(1000);
        latte.setPrice(Money.of(CurrencyUnit.of("CNY"), 35.0));
        latte.setUpdateTime(new Date());
        coffeeRepository.save(latte);
        coffeeRepository.findByName("latte").forEach(c -> log.info("Coffee {}", c));
        coffeeRepository.deleteAll();
```