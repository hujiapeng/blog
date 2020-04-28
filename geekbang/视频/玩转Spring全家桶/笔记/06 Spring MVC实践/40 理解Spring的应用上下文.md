1. Spring的应用上下文，常用接口及实现。一般不用BeanFactory，常用ApplicationContext接口实现类
 - BeanFactory
    - DefaultListableBean Factory

 - ApplicationContext
    - ClassPathXmlApplicationContext
    - FileSystemXmlApplicationContext
    - AnnotationConfigApplicationContext

 - WebApplicationContext

2. Spring应用上下文是可以有父子关系的，如果在当前上下文中找不到对应的Bean，就会去父项上找
3. 如下代码就是两个不同的应用上下文，且有父子关系
```
 ApplicationContext fooContext = new AnnotationConfigApplicationContext(FooConfig.class);
 ApplicationContext barContext = new ClassPathXmlApplicationContext(new String[]{"applicationContext.xml"},fooContext);
```
4. 在使用AOP代码时要注意，目标Bean是否和AOP Bean在同一个上下文
5. 注意，ClassPathXmlApplicationContext和FileSystemXmlApplicationContext是可以设置父ApplicationContext的，而AnnotationConfigApplicationContext不可以；也就是使用Xml配置的Spring应用上下文可以作为子应用上下文；
6. 注意，在@Aspect所在类加上@Component注解，这样就把AOP代理应用在Spring Boot启动的应用上下文了，也就是只能对@Autowired(为了区别同类型不同Bean还要用@Qualifier)的注入Bean生效了
7. 有父子关系的应用上下文，如果想让AOP都生效，可以在xml中配置```<aop:aspectj-autoproxy/> ```
8. 下面提供一个获取应用上下文工具
```
@Component
public class SpringContextUtil implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringContextUtil.applicationContext = applicationContext;
    }

    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }

    public static <T> T getBean(String name) {
        return (T) applicationContext.getBean(name);
    }
}
```