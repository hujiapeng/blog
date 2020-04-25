一、 配置解析
1. 基于schema设计解析
 - Dubbo框架直接集成了Spring的能力，利用了Spring配置文件扩展出自定义的解析方式。用来解析xml中的配置，如```<dubbo:application name="demo-consumer"/> ```
 - Spring自定义解析方式，使用三个文件
    - dubbo.xsd用来约束XML配置时的标签和对应的属性
    - spring.schemas用来指定约束文件dubbo.xsd位置，多出来一行，是因为捐给Apache组织后，项目包名需要改动
  ```
http\://dubbo.apache.org/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/compat/dubbo.xsd
```
    - spring.handlers文件指明解析XML配置文件的类，如类DubboNamespaceHandler，多出来一行原因如上
 ```
http\://dubbo.apache.org/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
```
2. 基于XML配置原理解析
 - 通过spring.handlers知，主要解析逻辑入口为DubboNamespaceHandler，部分源码如下；主要实现接口NamespaceHandlerSupport，会调用init方法；然后根据不同的标签关联到解析类DubboBeanDefinitionParser中
 ```
    public class DubboNamespaceHandler extends NamespaceHandlerSupport {
        ······
        @Override
        public void init() {
            registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
            registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
            registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        ······
        }
    }
```
 - 解析类DubboBeanDefinitionParser实现了接口BeanDefinitionParser，会调用parse方法，对每个标签元素进行实际解析，然后注册成Bean，注册Bean代码如下
 ```
        RootBeanDefinition beanDefinition = new RootBeanDefinition();
        beanDefinition.setBeanClass(beanClass);
        beanDefinition.setLazyInit(false);
······
        parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
······
```
 - RuntimeBeanReference，在Spring Bean解析阶段，某些依赖Bean有可能还没创建，就要使用RuntimeBeanReference创建依赖Bean，存到当前Bean的属性中，正在创建Bean的时候，会对RuntimeBeanReference进行解析。如xml中的ref
 ```
reference = new RuntimeBeanReference(value);
beanDefinition.getPropertyValues().addPropertyValue("ref", reference);
```
3. 基于注解配置原理解析
 - 入口看注解@EnableDubbo
 ```
······
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {
······
```
 - 然后再看注解@EnableDubboConfig，使用了@Import(DubboConfigConfigurationRegistrar.class)，DubboConfigConfigurationRegistrar类实现接口ImportBeanDefinitionRegistrar，会自动触发方法```registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) ```，参数importingClassMetadata就是使用该@Import注解的注解对应参数信息。此处就是处理配置信息
 ```
@Import(DubboConfigConfigurationRegistrar.class)
public @interface EnableDubboConfig {
```
 - @DubboComponentScan也是类似，@Import(DubboComponentScanRegistrar.class)，DubboComponentScanRegistrar实现接口ImportBeanDefinitionRegistrar，自动触发方法registerBeanDefinitions，进行指定扫描包下的Bean处理
 ```
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);
        registerReferenceAnnotationBeanPostProcessor(registry);
    }
```
 - Dubbo的自定义注解@Service的扫描与注册，主要是ServiceAnnotationBeanPostProcessor类实现接口BeanDefinitionRegistryPostProcessor接口，可以对BeanDefinition做后置处理
    - 对Dubbo @Service注解的类注册成普通Bean
    - 对Dubbo @Service注解的类注册成Dubbo的ServiceBean，BeanName格式为ServiceBean:接口全路径名，用于Spring启动后的服务暴露

 - Dubbo的自定义注解@Reference的注入，主要是实现接口MergedBeanDefinitionPostProcessor和继承类InstantiationAwareBeanPostProcessorAdapter
    - 实现接口MergedBeanDefinitionPostProcessor会调用postProcessMergedBeanDefinition方法，可以在Spring处理完所有Bean后做最后处理，Dubbo中在此方法出用于缓存带有@Reference注解的属性信息
    - 继承类InstantiationAwareBeanPostProcessorAdapter，可以覆盖方法postProcessPropertyValues，对Bean属性做处理；此处的处理就是对@Reference注解的属性作注入

二、 服务暴露的实现原理
1. 配置承载初始化，就是配置优先级
2. 远程服务暴露机制
 - 如果有多个注册中心或要暴露多个协议，两层循环来暴露多次
 - 如果没有指定本地服务暴露，就对本地暴露服务，本地服务暴露就是把服务信息放在缓存中
 - 如果配置了监控地址，服务信息会上报
 - 没有注册中心的话就直接暴露服务，直接暴露服务就是本地直接通过Netty发布服务，不再和注册中心交互，下面展示有注册中心和无注册中心的URL
 ```
registry://www.ittutu.cn:2181/com.alibaba.dubbo.registry.RegistryService?application=echo-annotation-provider&client=zkclient&dubbo=2.0.2&pid=6628&registry=zookeeper&timestamp=1587811621523
dubbo://ip:port/xxx.Service?timeout=10000&······
```
3. 服务消费的实现原理
 - 多注册中心同时消费，则会在ReferenceConfig#createProxy中合并成一个Invoker
 - 直连服务消费原理
    - 生产环境不建议使用直连方式
    - 使用方法为在<dubbo:reference或@Reference注解中直接配置url，如果多个逗号分隔

4. 优雅停机原理解析
 - 停机原理就是注册JVM停止的钩子