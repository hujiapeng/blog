注意使代码生成工具生成的代码要和自己写的model、mapper等代码分开存放，以免再次使用代码生成工具生成代码时将自己的代码删除掉
1. 创建Spring Boot项目，新增依赖
```
        <dependency>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-core</artifactId>
            <version>1.3.7</version>
        </dependency>
```
2. resources下创建schema.sql
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
3. application.properties配置文件如下
```
#这种方式指定的,mapperlocation包含/mapper/*.xml
mybatis.mapper-locations=classpath*:/mapper/**/*.xml
mybatis.type-aliases-package=com.hujiapeng.mybatisgeneratordemo.model
mybatis.type-handlers-package=com.hujiapeng.mybatisgeneratordemo.handler
mybatis.configuration.map-underscore-to-camel-case=true
```
4. 配置generatorConfig.xml，用于MyBatis相关类及文件生成。注意要保证targetProject配置的目录存在
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <context id="H2Tables" targetRuntime="MyBatis3">
        <plugin type="org.mybatis.generator.plugins.FluentBuilderMethodsPlugin"/>
        <plugin type="org.mybatis.generator.plugins.ToStringPlugin"/>
        <plugin type="org.mybatis.generator.plugins.SerializablePlugin"/>
        <plugin type="org.mybatis.generator.plugins.RowBoundsPlugin"/>
<!--        数据库连接信息-->
        <jdbcConnection driverClass="org.h2.Driver" connectionURL="jdbc:h2:mem:testdb" userId="sa" password=""/>
<!--        用来生成Model对象的-->
        <javaModelGenerator targetPackage="com.hujiapeng.mybatisgeneratordemo.model"
                            targetProject="./src/main/java"></javaModelGenerator>
<!--        用来生成mapper.xml文件的-->
        <sqlMapGenerator targetPackage="com.hujiapeng.mybatisgeneratordemo.mapper"
                         targetProject="./src/main/resources/mapper">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>
<!--        用来生成mapper接口的，混合类型会带有MyBatis注解-->
        <javaClientGenerator type="MIXEDMAPPER" targetPackage="com.hujiapeng.mybatisgeneratordemo.mapper"
                             targetProject="./src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>
        <!--        用来自动生成相应实体接口和文件的表-->
        <table tableName="t_coffee" domainObjectName="Coffee">
            <generatedKey column="id" sqlStatement="CALL IDENTITY()" identity="true"/>
            <columnOverride column="price" javaType="org.joda.money.Money" jdbcType="BIGINT" typeHandler="com.hujiapeng.mybatisgeneratordemo.handler.MoneyTypeHandler"/>
        </table>
    </context>
</generatorConfiguration>
```
5. 主类中生成代码方式如下
```
    private void generateArtifacts() throws Exception {
        List<String> warnings = new ArrayList<>();
        ConfigurationParser cp = new ConfigurationParser(warnings);
        Configuration configuration = cp.parseConfiguration(this.getClass().getResourceAsStream("/generatorConfig.xml"));
        DefaultShellCallback callback = new DefaultShellCallback(true);
        MyBatisGenerator generator = new MyBatisGenerator(configuration, callback, warnings);
        generator.generate(null);
    }
```
6. 主类中如果使用，就可以注入mapper，使用方式如下
```
    @Autowired
	private CoffeeMapper coffeeMapper;
······
        Coffee espresso = new Coffee()
				.withName("espresso")
				.withPrice(Money.of(CurrencyUnit.of("CNY"), 20.0))
				.withCreateTime(new Date())
				.withUpdateTime(new Date());
		coffeeMapper.insert(espresso);
		Coffee latte = new Coffee()
				.withName("latte")
				.withPrice(Money.of(CurrencyUnit.of("CNY"), 30.0))
				.withCreateTime(new Date())
				.withUpdateTime(new Date());
		coffeeMapper.insert(latte);
		Coffee s = coffeeMapper.selectByPrimaryKey(1L);
		log.info("Coffee {}", s);
		CoffeeExample example = new CoffeeExample();
		example.createCriteria().andNameEqualTo("latte");
		List<Coffee> list = coffeeMapper.selectByExample(example);
		list.forEach(e -> log.info("selectByExample: {}", e));
```