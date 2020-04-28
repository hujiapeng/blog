1. Spring会将各种数据库常见的异常转换为自己的异常DataAccessException。Spring会收集各种数据库的异常错误码，进行分门别类，默认文件路径为org/springframework/jdbc/support/sql-error-codes.xml，通过SQLErrorCodeSQLExceptionTranslator转换器进行解析。
 自己可以在Classpath下创建sql-error-codes.xml文件，来覆盖默认的配置，定制错误码解析逻辑。如果想要根据某些错误码，抛出自定义的异常，可以通过添加属性customTranslations，在CustomSQLErrorCodesTranslation Bean中指定错误码和对应的异常类。注意异常类必须继承自DataAccessException。
2. 自定义一个异常类，继承自DuplicateKeyException(最终继承自DataAccessException)
 ```
public class CustomDuplicatedKeyException extends DuplicateKeyException {
    public CustomDuplicatedKeyException(String msg) {
        super(msg);
    }
    public CustomDuplicatedKeyException(String msg, Throwable cause) {
        super(msg, cause);
    }
}
 ```

3. classpath下创建sql-error-codes.xml，是对H2数据库错误码定制的解析逻辑
 ```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN 2.0//EN" "http://www.springframework.org/dtd/spring-beans-2.0.dtd">
<beans>
    <bean id="H2" class="org.springframework.jdbc.support.SQLErrorCodes">
        <property name="badSqlGrammarCodes">
            <value>42000,42001,42101,42102,42111,42112,42121,42122,42132</value>
        </property>
        <property name="duplicateKeyCodes">
            <value>23001,23505</value>
        </property>
        <property name="dataIntegrityViolationCodes">
            <value>22001,22003,22012,22018,22025,23000,23002,23003,23502,23503,23506,23507,23513</value>
        </property>
        <property name="dataAccessResourceFailureCodes">
            <value>90046,90100,90117,90121,90126</value>
        </property>
        <property name="cannotAcquireLockCodes">
            <value>50200</value>
        </property>
        <property name="customTranslations">
            <bean class="org.springframework.jdbc.support.CustomSQLErrorCodesTranslation">
                <property name="errorCodes" value="23001,23505" />
                <property name="exceptionClass"
                          value="com.hujiapeng.errorcodedemo.CustomDuplicatedKeyException" />
            </bean>
        </property>
    </bean>
</beans>
 ```
4. 测试类中测试
 ```
@SpringBootTest
@RunWith(SpringRunner.class)
public class ErrorcodeDemoApplicationTests {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    @Test(expected = CustomDuplicatedKeyException.class)
    public void testThrowingCustomException() {
        jdbcTemplate.execute("INSERT INTO FOO (ID, BAR) VALUES (1, 'a')");
        jdbcTemplate.execute("INSERT INTO FOO (ID, BAR) VALUES (1, 'b')");
    }
}
 ```