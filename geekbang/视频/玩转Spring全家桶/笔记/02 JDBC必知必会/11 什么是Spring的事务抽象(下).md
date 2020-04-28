1. 编码方式控制事务，事务管理器TransactionTemplate和JDBC JdbcTemplate通过依赖注入，Demo如下：
 ```
public void run(String... args) throws Exception {
        log.info("COUNT BEFORE TRANSACTION: {}", getCount());
        Object execute = transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
                jdbcTemplate.execute("INSERT INTO FOO (ID, BAR) VALUES (1, 'aaa')");
                log.info("COUNT IN TRANSACTION: {}", getCount());
                transactionStatus.setRollbackOnly();
            }
        });
        log.info("COUNT AFTER TRANSACTION: {}", getCount());
    }
    private long getCount() {
        return (long) jdbcTemplate.queryForList("select count(*) as CNT from foo").get(0).get("CNT");
    }
 ```
 如果需要有返回值，则使用TransactionCallback
2. 注解方式控制事务，首先要在配置类或启动类上加注解@EnableTransactionManagement，开启事务管理器；如果使用XML方式，在Spring的配置文件上加上```<tx:annotation-drivern/> ```开启事务管理器。事务管理器中的属性有，proxyTargetClass表示是否代理指向类，也就是使用Java代理对接口还是Cglib代理对类增强；mode方式是Java模式还是AspectJ模式，order是AOP拦截顺序，默认最低级别，这样可以最后执行。
 在需要执行事务的类或方法上添加@Transaction注解，注解中可以指定transactionManager，如果系统中只有一个，也可以不指定；还可以指定事务传播特性、隔离级别、超时，是否只读以及根据特定异常回滚。
 ```
@SpringBootApplication
@EnableTransactionManagement(mode = AdviceMode.PROXY)
 ```
 ```
    @Transactional(rollbackFor = RollbackException.class)
    public void insertRecord() {
        jdbcTemplate.execute("INSERT INTO FOO (BAR) VALUES ('AAA')");
    }
 ```