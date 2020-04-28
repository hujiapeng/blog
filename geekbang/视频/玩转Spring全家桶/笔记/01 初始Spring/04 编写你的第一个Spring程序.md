1. 使用的Spring boot版本为2.1.13，添加的依赖有Web和Actuator
2. 既然添加了Web依赖，就可以在@SpringBootApplication所在层级或下面层级添加@RestController注解的Controller类。Controller类中添加请求方法如下
```
    @RequestMapping("/hello")
    public String hello() {
        return "Hello Spring";
    }
```
 Terminal中测试请求如下
 ```
$ curl http://localhost:8080/hello
Hello Spring
 ```
3. 由于添加了Actuator这个依赖，那么可以检查系统允许状态，如检查系统健康状态.UP表示正常
 ```
$ curl http://localhost:8080/actuator/health
{"status":"UP"}
 ```
4. 通过maven工具将系统打包成jar包，打包完成后可以执行```java -jar XXX.jar```命令启动
 ```
$ mvn clean package -Dmaven.test.skip
 ```
5. 如果自己的Spring Boot项目有自己的maven parent，或者不想用maven parent，所以就不能使用初始化自动配置的parent了，但是又想使用Spring Boot约定优于配置的模式，那么可如下配置pom
 ```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.hujiapeng</groupId>
    <artifactId>hellospring</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>hellospring</name>
    <description>Demo project for Spring Boot</description>
	<properties>
		<java.version>1.8</java.version>
	</properties>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>2.1.1.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<version>2.1.1.RELEASE</version>
				<executions>
					<execution>
						<goals>
							<goal>repackage</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
</project>
 ```
 其实可以查看Spring Boot的maven parent，里面还有个parent，里面的parent依赖就是和上面dependencyManagement中一样的spring-boot-dependencies