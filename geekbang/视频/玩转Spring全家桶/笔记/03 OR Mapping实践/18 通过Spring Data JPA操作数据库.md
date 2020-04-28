需要接上一节
1. model下类
 - CoffeeOrder，@ManyToMany注解表示多对多，默认为fetch = FetchType.LAZY，注意在获取实体对象时，如果使用懒加载，要在调用获取实体的方法或类上加注解@Transactional，否则报异常``` org.hibernate.LazyInitializationException: failed to lazily initialize a collection of role ```这是因为Hibernate在加载完实体后就把session关闭了，如果开启了事务，那么整个方法session直到事务提交再退出
 ```
     @Entity
     @Table(name = "T_ORDER")
     @Data
     @ToString(callSuper = true)
     @NoArgsConstructor
     @AllArgsConstructor
     @Builder
     public class CoffeeOrder extends BaseEntity implements Serializable {
         private String customer;
         @ManyToMany
         @JoinTable(name = "T_ORDER_COFFEE")
         @OrderBy("id")
         private List<Coffee> items;
         @Enumerated
         @Column(nullable = false)
         private OrderState state;
     }
 ```
 - OrderState
 ```
    public enum OrderState {
        INIT, PAID, BREWING, BREWED, TAKEN, CANCELLED
    }
```
2. repository下的类
 - CoffeeOrderRepository
 ```
public interface CoffeeOrderRepository extends JpaRepository<CoffeeOrder, Long> {
}
```
3. 主类
 ```
    @SpringBootApplication
    @EnableJpaRepositories
    @EnableTransactionManagement
    @Slf4j
    public class SpringbucksApplication implements ApplicationRunner {
        @Autowired
        private CoffeeOrderRepository orderRepository;
        public static void main(String[] args) {
            SpringApplication.run(SpringbucksApplication.class, args);
        }
        @Override
        @Transactional
        public void run(ApplicationArguments args) throws Exception {
            log.info("All CoffeeOrder: {}",orderRepository.count());
            orderRepository.findAll().forEach(order -> log.info(order.toString()));
        }
    }
```