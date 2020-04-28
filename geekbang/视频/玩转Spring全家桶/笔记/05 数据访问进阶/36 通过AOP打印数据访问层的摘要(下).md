1. 如何打印SQL
 - HikariCP：可以借助P6Spy，来辅助SQL输出 ```https://github.com/p6spy/p6spy ```  ```https://p6spy.readthedocs.io/en/latest/integration.html ```
 - Alibaba Druid内置SQL输出 ```https://github.com/alibaba/druid/wiki/Druid%E4%B8%AD%E4%BD%BF%E7%94%A8log4j2%E8%BF%9B%E8%A1%8C%E6%97%A5%E5%BF%97%E8%BE%93%E5%87%BA ```

2. 借助p6spy，使用HikariCP输出SQL
 - 使用SpringBucks项目
 - pom.xml添加依赖
```
        <dependency>
            <groupId>p6spy</groupId>
            <artifactId>p6spy</artifactId>
            <version>3.8.1</version>
        </dependency>
```
 - 修改配置文件application.properties
```
#spring.datasource.type默认还是HikariCP
#spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.driver-class-name=com.p6spy.engine.spy.P6SpyDriver
spring.datasource.url=jdbc:p6spy:h2:mem:testdb
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=none
```
 - 创建配置文件spy.properties，这个是p6spy默认的配置文件
```
# 单行日志
logMessageFormat=com.p6spy.engine.spy.appender.SingleLineFormat
# 使用Slf4J记录sql
appender=com.p6spy.engine.spy.appender.Slf4JLogger
# 是否开启慢SQL记录
outagedetection=true
# 慢SQL记录标准，单位秒
outagedetectioninterval=2
```
 - aspect包下添加AOP类PerformanceAspect.java；注意添加注解@Component和@Aspect
```
    @Component
    @Aspect
    @Slf4j
    public class PerformanceAspect {
        //    @Around("execution(* com.hujiapeng.performanceaspectdemo.repository..*(..))")
        @Around("repositoryOps()")
        public Object logPerformance(ProceedingJoinPoint pjp) throws Throwable {
            long startTime = System.currentTimeMillis();
            String name = "-";
            String result = "Y";
            try {
                name = pjp.getSignature().toShortString();
                return pjp.proceed();
            } catch (Throwable ex) {
                result = "N";
                ex.printStackTrace();
                throw ex;
            } finally {
                long endTime = System.currentTimeMillis();
                log.info("{};{};{}ms", name, result, endTime - startTime);
            }
        }
        @Pointcut("execution(* com.hujiapeng.performanceaspectdemo.repository..*(..))")
        private void repositoryOps() {
        }
    }
```
 - 主类中也要添加注解@EnableAspectJAutoProxy，然后运行可看测试效果 