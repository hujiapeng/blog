1. 创建一个Spring Boot2.0项目，加入Web、Actuator、H2、JDBC、Lombok依赖
2. 准备schema.sql和data.sql初始化数据
3. 定义一个Dao，里面有JdbcTemplate字段，通过Spring依赖注入
4. 可以通过JdbcTemplate封装对数据进行插入和查询操作的方法
5. 主类中注入Dao字段，然后调用测试
6. 由于没有配置数据源，Spring Boot默认就使用本地内存数据库H2，数据库连接池为HikariCP
7. 在Dao中还有个SimpleJdbcInsert，这个是Spring简化的数据插入操作类，需要在配置类中生成Bean，并注入DataSource或JdbcTemplate，如下
 ```
    @Bean
    @Autowired
    public SimpleJdbcInsert simpleJdbcInsert(JdbcTemplate jdbcTemplate) {
        return new SimpleJdbcInsert(jdbcTemplate);
    }
 ```
 使用的时候要指定操作的表，传入的参数是一个表字段和值的Map，需要返回主键的话要指定主键，如下
 ```
  HashMap<String, String> row = new HashMap<>();
  row.put("BAR", "d");
  Number id = simpleJdbcInsert.withTableName("foo").usingGeneratedKeyColumns("id").executeAndReturnKey(row);
  log.info("ID of d: {}", id.longValue());
 ```
8. SpringJDBC批量操作，可以使用JdbcTemplate或NamedParameterJdbcTemplate的batchUpdate方法，如下
 ```
  jdbcTemplate.batchUpdate("insert into foo(bar) values(?)", new BatchPreparedStatementSetter() {
       @Override
       public void setValues(PreparedStatement ps, int i) throws SQLException {
           ps.setString(1, "b-" + i);
       }
       @Override
       public int getBatchSize() {
           return 5;
       }
   });
 ```
 ```
   List<Foo> list = new ArrayList<>();
   list.add(Foo.builder().id(100L).bar("b-100").build());
   list.add(Foo.builder().id(101L).bar("b-101").build());
   namedParameterJdbcTemplate.batchUpdate("insert into foo(id,bar) values(:id,:bar)",
           SqlParameterSourceUtils.createBatch(list));
 ```