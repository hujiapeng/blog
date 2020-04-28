1. Spring Boot1.0中默认使用Tomcat的数据库连接池，而在Spring Boot2.0后默认使用HikariCP
2. HikariCP,A high-performance JDBC connection pool。Hikari来自日语“光”，表示快：
 - 字节码级别优化，很多方法都是通过JavaAssist生成
 - 大量的小改进，造成最终的快，如：
    - 用自己写的FastStatementList替换ArrayList
    - 借鉴.Net中的ConcurrentBag，自己写的ConcurrentBag，在并发情况下性能提升
    - 字节码级别，代理类的优化，如用invokestatic代替了invokevirtual
3. Spring Boot中配置HikariCP：
 - 如果使用的是2.0，默认使用HikariCP，那么直接在配置文件中使用```spring.datasource.hikari.* ```配置
 - 如果使用的是1.0，默认使用Tomcat连接池，那就需要移除tomcat-jdbc依赖，然后引入HikariCP依赖。配置文件中指定使用HikariDataSource，如```spring.datasource.type=com.zaxxer.hikari.HikariDataSource ``` 

4. Hikari在Spring Boot中DataSource的默认配置源码如下
 ```
	/**
	 * Hikari DataSource configuration.
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
			matchIfMissing = true)
	static class Hikari {
		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties) {
			HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}
	}
 ```
5. 其他关于Hikari可以参考官网[https://github.com/brettwooldridge/HikariCP](https://github.com/brettwooldridge/HikariCP)