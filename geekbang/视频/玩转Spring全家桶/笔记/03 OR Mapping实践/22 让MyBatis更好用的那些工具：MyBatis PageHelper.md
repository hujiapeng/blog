[MyBatis-PageHelper Git地址](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/README_zh.md)，[官网地址](https://pagehelper.github.io/docs/)
1. 创建Spring Boot项目，添加依赖
```
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>1.2.10</version>
        </dependency>
```
2. resources下schema.sql文件
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
3. resources下data.sql
```
insert into t_coffee (name, price, create_time, update_time) values ('espresso', 2000, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('latte', 2500, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('capuccino', 2500, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('mocha', 3000, now(), now());
insert into t_coffee (name, price, create_time, update_time) values ('macchiato', 3000, now(), now());
```
4. application.properties配置如下
```
mybatis.type-handlers-package=geektime.spring.data.mybatisdemo.handler
mybatis.configuration.map-underscore-to-camel-case=true
#默认为false，只有使用RowBounds作为分页参数时有效，如果为true，RowBounds中的offset参数当成pageNum使用
pagehelper.offset-as-page-num=true
#默认为false，分页合理化参数，按说页码是大于0小于等于页数的，如果出现<=0页码，则当初0处理；如果大于总页数，则按最后一页处理
pagehelper.reasonable=true
#默认为false，当为true时，如果pageSize=0或RowBounds.limit=0就会查询全部，相当于没有分页但返回Page类型
pagehelper.page-size-zero=true
#默认为false，如果为true，分页插件会从查询方法的参数中，自动根据params配置的字段取值，查到合适的值就自动分页
pagehelper.support-methods-arguments=true
```
5. model下的Coffee
```
   @Data
   @AllArgsConstructor
   @NoArgsConstructor
   @Builder
   public class Coffee {
       private Long id;
       private String name;
       private Money price;
       private Date createTime;
       private Date updateTime;
   }
```
6. handler下的MoneyTypeHandler
```
   public class MoneyTypeHandler extends BaseTypeHandler<Money> {
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Money parameter, JdbcType jdbcType) throws SQLException {
        ps.setLong(i, parameter.getAmountMinorLong());
    }
    @Override
    public Money getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return parseMoney(rs.getLong(columnName));
    }
    @Override
    public Money getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return parseMoney(rs.getLong(columnIndex));
    }
    @Override
    public Money getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return parseMoney(cs.getLong(columnIndex));
    }
    private Money parseMoney(Long value) {
        return Money.of(CurrencyUnit.of("CNY"), value / 100.0);
    }
}
```
7. mapper下的CoffeeMapper
```
   //如果配置了MapperScan就可以不用配置@Mapper
   //@Mapper
   public interface CoffeeMapper {
       @Select("select * from t_coffee order by id")
       List<Coffee> findAllWithRowBounds(RowBounds rowBounds);
       @Select("select * from t_coffee order by id")
       List<Coffee> findAllWithParam(@Param("pageNum") int pageNum, @Param("pageSize") int pageSize);
   }
```
8. 主类下测试
```
   @SpringBootApplication
@Slf4j
@MapperScan("com.hujiapeng.mybatispagehelperdemo.mapper")
public class MybatisPagehelperDemoApplication implements ApplicationRunner {
    @Autowired
    private CoffeeMapper coffeeMapper;
    public static void main(String[] args) {
        SpringApplication.run(MybatisPagehelperDemoApplication.class, args);
    }
    @Override
    public void run(ApplicationArguments args) throws Exception {
        coffeeMapper.findAllWithRowBounds(new RowBounds(1, 3))
                .forEach(coffee -> log.info("Page(1) Coffee {}", coffee));
        coffeeMapper.findAllWithRowBounds(new RowBounds(2, 3))
                .forEach(coffee -> log.info("Page(2) Coffee {}", coffee));
        log.info("===================");
        //如果offset为0，则根据reasonable为true，offset相当于1；
        //如果limit为0，根据page-size-zero为true，则查询全部
        coffeeMapper.findAllWithRowBounds(new RowBounds(1, 0))
                .forEach(c -> log.info("Page(1) Coffee {}", c));
        log.info("===================");
        coffeeMapper.findAllWithParam(1, 3)
                .forEach(coffee -> log.info("Page(1) Coffee {}", coffee));
        List<Coffee> list = coffeeMapper.findAllWithParam(2, 3);
        PageInfo page = new PageInfo(list);
        log.info("PageInfo: {}", page);
    }
}
```