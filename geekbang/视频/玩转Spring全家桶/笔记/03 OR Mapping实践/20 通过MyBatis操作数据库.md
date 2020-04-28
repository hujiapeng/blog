1. 搭建Spring Boot项目后，准备schema.sql如下
```
   create table t_coffee (
       id bigint not null auto_increment,
       name varchar(255),
       price bigint not null,
       create_time timestamp,
       update_time timestamp,
       primary key (id)
   );
```
2. application.properties配置如下
```
#注意下面配置的是handler所在包
mybatis.type-handlers-package=com.hujiapeng.mybatisdemo.handler
#下划线转驼峰格式，这样的表字段如果使用的是下划线，对应的实体类属性可以转成驼峰格式
mybatis.configuration.map-underscore-to-camel-case=true
#可以在控制台查看执行的SQL
mybatis.configuration.log-impl=org.apache.ibatis.logging.stdout.StdOutImpl
```
3. model下准备Coffee类
```
   @Data
   @Builder
   @NoArgsConstructor
   @AllArgsConstructor
   public class Coffee {
       private Long id;
       private String name;
       private Money price;
       private Date createTime;
       private Date updateTime;
   }
```
4. handler下配置类型转换器MoneyTypeHandler
```
   /**
    * 在 Money 与 Long 之间转换的 TypeHandler，处理 CNY 人民币
    */
   public class MoneyTypeHandler extends BaseTypeHandler<Money> {
       @Override
       public void setNonNullParameter(PreparedStatement preparedStatement, int i, Money money, JdbcType jdbcType) throws SQLException {
           preparedStatement.setLong(i, money.getAmountMinorLong());
       }
       @Override
       public Money getNullableResult(ResultSet resultSet, String columnName) throws SQLException {
           return parseMoney(resultSet.getLong(columnName));
       }
       @Override
       public Money getNullableResult(ResultSet resultSet, int i) throws SQLException {
           return parseMoney(resultSet.getLong(i));
       }
       @Override
       public Money getNullableResult(CallableStatement callableStatement, int i) throws SQLException {
           return parseMoney(callableStatement.getLong(i));
       }
       private Money parseMoney(Long value) {
   //        return Money.ofMinor(CurrencyUnit.of("CNY"), value);
           return Money.of(CurrencyUnit.of("CNY"), value / 100.0);
       }
   }
```
5. mapper下创建Mapper接口
```
   @Mapper
   public interface CoffeeMapper {
       @Insert("insert into t_coffee (name, price, create_time, update_time)"
               + "values (#{name}, #{price}, now(), now())")
       //用于返回主键，在对象中。直接的返回值是影响行数
       @Options(useGeneratedKeys = true)
       Integer save(Coffee coffee);
       @Select("select * from t_coffee where id = #{id}")
       @Results({
               @Result(id = true, column = "id", property = "id"),
               //和配置map-underscore-to-camel-case = true 可以实现一样的效果
   //            @Result(column = "create_time", property = "createTime"),
   //            @Result(column = "update_time", property = "updateTime")
       })
       Coffee findById(@Param("id") Long id);
   }
```
6. 主类中测试使用
```
   @SpringBootApplication
   @Slf4j
   @MapperScan("com.hujiapeng.mybatisdemo.mapper")
   public class MybatisDemoApplication implements ApplicationRunner {
       @Autowired
       private CoffeeMapper coffeeMapper;
       public static void main(String[] args) {
           SpringApplication.run(MybatisDemoApplication.class, args);
       }
       @Override
       public void run(ApplicationArguments args) throws Exception {
           Coffee c = Coffee.builder().name("espresso")
                   .price(Money.of(CurrencyUnit.of("CNY"), 20.0)).build();
           //注意返回值是影响行数
           Integer count = coffeeMapper.save(c);
           log.info("Save {} Coffee => {}", count, c);
           c = Coffee.builder().name("latte")
                   .price(Money.of(CurrencyUnit.of("CNY"), 25.0)).build();
           coffeeMapper.save(c);
           Long id = c.getId();
           c = coffeeMapper.findById(id);
           log.info("Coffee {}", c);
       }
   }
```