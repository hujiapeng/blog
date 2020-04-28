1. schema.sql
 ```
   drop table t_coffee if exists;
   drop table t_order if exists;
   drop table t_order_coffee if exists;
   create table t_coffee (
       id bigint auto_increment,
       create_time timestamp,
       update_time timestamp,
       name varchar(255),
       price bigint,
       primary key (id)
   );
   create table t_order (
       id bigint auto_increment,
       create_time timestamp,
       update_time timestamp,
       customer varchar(255),
       state integer not null,
       primary key (id)
   );
   create table t_order_coffee (
       coffee_order_id bigint not null,
       items_id bigint not null
   );
insert into t_coffee (name, price, create_time, update_time) values ('espresso', 2000, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('latte', 2500, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('capuccino', 2500, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('mocha', 3000, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('macchiato', 3000, now(), now());
```
2. application.properties
 ```
spring.jpa.hibernate.ddl-auto=none
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.format_sql=true
```
3. model下的类
 - BaseEntity，@MappedSuperclass注解是表示该类不是完整的实体类，但它的属性作为子类的数据库字段；@GeneratedValue注解表示主键生成；@Column(updatable = false)表示字段不可修改
 ```
     @MappedSuperclass
     @Data
     @NoArgsConstructor
     @AllArgsConstructor
     public class BaseEntity implements Serializable {
         @Id
         @GeneratedValue(strategy = GenerationType.IDENTITY)
         private Long id;
         @Column(updatable = false)
         @CreationTimestamp
         private Date createTime;
         @UpdateTimestamp
         private Date updateTime;
     }
 ```
 - Coffee，@Type表示该字段或对应get方法返回对象映射到的Hibernate对应类(实现接口ParameterizedType)
 ```
@Entity
@Table(name = "T_COFFEE")
@Builder
@Data
@NoArgsConstructor
@AllArgsConstructor
@ToString(callSuper = true)
public class Coffee extends BaseEntity {
    private String name;
    @Type(type = "org.jadira.usertype.moneyandcurrency.joda.PersistentMoneyMinorAmount",
    parameters = {@Parameter(name = "currencyCode",value = "CNY")})
    private Money price;
}
```
4. repository下的类
 - CoffeeRepository，父类JpaRepository泛型
 ```
public interface CoffeeRepository extends JpaRepository<Coffee, Long> {
}
```
5. service下的类
 - CoffeeService，Optional泛型可以简化一些判空等操作，如coffee!=null ，可以用coffeeOptional.isPresent()
 ```
    @Service
    @Slf4j
    public class CoffeeService {
        @Autowired
        private CoffeeRepository coffeeRepository;
        public Optional<Coffee> findOneCoffee(String name) {
            // exact().ignoreCase()表示完全匹配并且忽略大小写
            ExampleMatcher matcher = ExampleMatcher.matching().withMatcher("name", exact().ignoreCase());
            Optional<Coffee> coffee = coffeeRepository.findOne(Example.of(Coffee.builder().name(name).build(), matcher));
            log.info("Coffee Found: {}", coffee);
            return coffee;
        }
    }
```
6. 主类下使用，其实可以省略注解@EnableJpaRepositories和@EnableTransactionManagement，因为Spring Boot默认会通过JpaRepositoriesAutoConfiguration和TransactionAutoConfiguration类根据条件启用
 ```
    @SpringBootApplication
    @EnableJpaRepositories
    @EnableTransactionManagement
    @Slf4j
    public class SpringbucksApplication implements ApplicationRunner {
        @Autowired
        private CoffeeRepository coffeeRepository;
        @Autowired
        private CoffeeService coffeeService;
        public static void main(String[] args) {
            SpringApplication.run(SpringbucksApplication.class, args);
        }
        @Override
        public void run(ApplicationArguments args) throws Exception {
            log.info("All Coffee: {}",coffeeRepository.count());
            coffeeRepository.findAll().forEach(coffee -> log.info(coffee.toString()));
            Optional<Coffee> latte = coffeeService.findOneCoffee("Latte");
            if (latte.isPresent()) {
                
            }
        }
    }
```