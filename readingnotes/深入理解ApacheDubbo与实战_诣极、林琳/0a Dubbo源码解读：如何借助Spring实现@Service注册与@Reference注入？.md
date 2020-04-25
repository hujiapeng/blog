一、Spring Dubbo启动原理解析
1. 如果使用的是注解方式用Dubbo，解析自定义注解及类的入口类是DubboComponentScanRegistrar；如果使用的是xml方式用Dubbo，解析自定义标签的入口类在DubboNamespaceHandler。考虑到注解使用方便，下面就直接分析注解方式的源码
2. 进入DubboComponentScanRegistrar源码
 - 实现接口ImportBeanDefinitionRegistrar，该接口配合@Configuration和@Import使用。使用方式如下，下面例子是源码中的，所以例子复杂了点，多了一层注解。
 - 引入DubboComponentScanRegistrar的注解：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan
```
 引用注解DubboComponentScan的注解：
 ```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo
```
 对以上注解的使用：
 ```
    @Configuration
    @EnableDubbo(scanBasePackages = "com.hujiapeng.echo")
    class ConsumerConfiguration 
```
 - 引入DubboComponentScanRegistrar后进入方法registerBeanDefinitions，这里是三件事开启Dubbo，1）、获取@EnableDubbo注解中的扫描包路径；2）、启用自定义注解@Service，是用在服务提供接口实现类上的；3）、启用自定义注解@Reference，是用在服务消费者中属性上的，用来自动注入
```
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);
        registerReferenceAnnotationBeanPostProcessor(registry);
    }
```
    - registerBeanDefinitions方法中AnnotationMetadata参数就是引入DubboComponentScanRegistrar类对应的注解元数据；getPackagesToScan(importingClassMetadata)方法就是获取到@EnableDubbo注解中的packages；
    - BeanDefinitionRegistry是用来注册和获取BeanDefinition的

 - getPackagesToScan(importingClassMetadata)方法获取到Dubbo扫描的包；
 - registerServiceAnnotationBeanPostProcessor(packagesToScan, registry)是用来注册ServiceAnnotationBeanPostProcessor的，该类实现接口BeanDefinitionRegistryPostProcessor Bean后置处理器，Spring构造完Bean定义后，创建Bean实例前会进入postProcessBeanDefinitionRegistry方法；
    - registerServiceAnnotationBeanPostProcessor源码如下，创建Bean ServiceAnnotationBeanPostProcessor，并通过BeanDefinitionBuilder为该Bean设置属性；ServiceAnnotationBeanPostProcessor中使用扫描包的构造函数内容就是从这设置好的；
```
    private void registerServiceAnnotationBeanPostProcessor(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceAnnotationBeanPostProcessor.class);
        //为构造函数设置扫描包
        builder.addConstructorArgValue(packagesToScan);
        builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        //registerWithGeneratedName方法是Spring中的方法，就是根据类获取到BeanName，注册Bean
        BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);
    }
```
    - ServiceAnnotationBeanPostProcessor其实就是解析Dubbo中使用的自定义@Service
 - registerReferenceAnnotationBeanPostProcessor(registry)是用来注册ReferenceAnnotationBeanPostProcessor的，即@Reference Bean；用Dubbo的BeanRegistrar工具类registerInfrastructureBean方法注册基础Bean，代码如下：
 ```
    private void registerReferenceAnnotationBeanPostProcessor(BeanDefinitionRegistry registry) {
        // Register @Reference Annotation Bean Processor
        BeanRegistrar.registerInfrastructureBean(registry,
                ReferenceAnnotationBeanPostProcessor.BEAN_NAME, ReferenceAnnotationBeanPostProcessor.class);
    }
```
 - registerInfrastructureBean方法如下
 ```
    public static void registerInfrastructureBean(BeanDefinitionRegistry beanDefinitionRegistry,
                                                  String beanName,
                                                  Class<?> beanType) {
        if (!beanDefinitionRegistry.containsBeanDefinition(beanName)) {
            RootBeanDefinition beanDefinition = new RootBeanDefinition(beanType);
            beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
            beanDefinitionRegistry.registerBeanDefinition(beanName, beanDefinition);
        }
    }
```

二、 下面解读，解析Dubbo自定义注解@Service的类ServiceAnnotationBeanPostProcessor
1. 先看该类实现的接口，源码如下，BeanDefinitionRegistryPostProcessor是BeanFactory后处理器，
```
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware 
```
2. BeanDefinitionRegistryPostProcessor，注册BeanDefinition后处理器，上面对ServiceAnnotationBeanPostProcessor做了Bean注册，所以就会进入实现相应后处理器的postProcessBeanDefinitionRegistry方法
```
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);
        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }
    }
```
 - resolvePackagesToScan(packagesToScan)方法是用来解析字符串中的${XXX}占位符的，进行替换，主要是Environment.resolvePlaceholders方法，Environment对象是通过接口EnvironmentAware注入的，没有其他逻辑，最后返回解析完后的包集合，代码如下
```
    private Set<String> resolvePackagesToScan(Set<String> packagesToScan) {
        Set<String> resolvedPackagesToScan = new LinkedHashSet<String>(packagesToScan.size());
        for (String packageToScan : packagesToScan) {
            if (StringUtils.hasText(packageToScan)) {
                String resolvedPackageToScan = environment.resolvePlaceholders(packageToScan.trim());
                resolvedPackagesToScan.add(resolvedPackageToScan);
            }
        }
        return resolvedPackagesToScan;
    }
```
 packagesToScan参数来自该类构造方法传入，即注册该类Bean时，靠BeanDefinitionBuilder.addConstructorArgValue传入
 - registerServiceBeans(resolvedPackagesToScan, registry)方法用来注册带有Dubbo自定义注解@Service的Bean，代码如下：
```
private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
        //DubboClassPathBeanDefinitionScanner继承ClassPathBeanDefinitionScanner，根本是构造ClassPathBeanDefinitionScanner，
        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);
        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
        scanner.setBeanNameGenerator(beanNameGenerator);
        //注意：如下scanner是过滤出带有@Service注解的类
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));
        for (String packageToScan : packagesToScan) {
            // Registers @Service Bean first
            //注意：根据scanner过滤出的类，注册成BeanDefinition
            //注意：此处已经注册成BeanDefinition了，可以使用FactoryBean根据BeanDefinition中beanClass初始化对象
            scanner.scan(packageToScan);
            // Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
            //如下方法是获取BeanDefinitionHolder
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);
            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    //注意：下面注册的Bean是ServiceBean，根据不同接口注册不同ServiceBean，
                    //如果一个接口多个@Service注解实现类，则取第一个
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }
                if (logger.isInfoEnabled()) {
                    logger.info(beanDefinitionHolders.size() + " annotated Dubbo's @Service Components { " +
                            beanDefinitionHolders +
                            " } were scanned under package[" + packageToScan + "]");
                }
            } else {
                if (logger.isWarnEnabled()) {
                    logger.warn("No Spring Bean annotating Dubbo's @Service was found under package["
                            + packageToScan + "]");
                }
            }
        }
    }
```
   由于代码量的原因，只说重点的几个：
    - ClassPathBeanDefinitionScanner根据scanner设置的条件，通过scan方法注册符合条件的Bean，即将带有注解@Service的类注册成Bean
    - 通过代码得知带有@Service注解的类是符合scanner注册Bean所需条件的。第一层是isCandidateComponent(MetadataReader metadataReader)，过滤出带有@Service注解的类；第二层是isCandidateComponent(AnnotatedBeanDefinition beanDefinition)，带有@Service注解的类是对立类，并且可直接实例化
    - 自定义实现ClassPathBeanDefinitionScanner类时，可通过覆盖相应的isCandidateComponent方法实现获取BeanDefinition，用于判断的代码如下：
    ```
	protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
		AnnotationMetadata metadata = beanDefinition.getMetadata();
        //isIndependent是否为独立的(一般的类，即非内部类，或者静态类都是可直接实例化的独立类)
        //isConcrete是否为具体类，即非接口和抽象类
        //对于Lookup注解，就是Spring通过该注解调用相应方法
		return (metadata.isIndependent() && (metadata.isConcrete() ||
				(metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));
	}
```
    - findServiceBeanDefinitionHolders方法中是通过scanner获取到对应包下的BeanDefinition，然后返回```Set<BeanDefinitionHolder> ```就是包装了BeanDefinition，带有BeanName的BeanDefinitionHolder
    - 对BeanDefinitionHolder进行操作，注册成ServiceBean，每一个@Service实现类对应的接口会构造成一个ServiceBean，如果一个接口有多个实现，那么就使用第一个；如果一个@Service注解的类没有接口，并且@Service注解参数中也没有指定接口，那么就抛出异常
    - 解读下方法registerServiceBean，真正来注册ServiceBean的。
    ```
    private void registerServiceBean(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                     DubboClassPathBeanDefinitionScanner scanner) {
        //下面两个方法就是获取Bean Class和Service注解
        Class<?> beanClass = resolveClass(beanDefinitionHolder);
        Service service = findAnnotation(beanClass, Service.class);
        //如下方法是获取Bean对应的接口，如果@Service中指定了接口，就用@Service中的，
        //否则就使用Bean实现的接口，如果没有实现就抛出异常，如果有实现多个就使用第一个接口
        Class<?> interfaceClass = resolveServiceInterfaceClass(beanClass, service);
        //获取Bean的名称，就是@Service对应实现类的名称
        String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();
        //获取ServiceBean的BeanDefinition，下面方法就是构造ServiceBean的BeanDefinition，并且添加属性，
        //如ref对应的annotatedServiceBeanName，interface对应interfaceClass等其他属性
        //方法内直接通过BeanDefinitionBuilder builder = rootBeanDefinition(ServiceBean.class);先创建一个ServiceBean作为RootBean
        AbstractBeanDefinition serviceBeanDefinition =
                buildServiceBeanDefinition(service, interfaceClass, annotatedServiceBeanName);
        // ServiceBean Bean name，这个ServiceBean的BeanName格式就是ServiceName:接口引用路径，如ServiceBean:com.hujiapeng.echo.api.EchoService
        String beanName = generateServiceBeanName(service, interfaceClass, annotatedServiceBeanName);
        //如下的校验就是对一个接口多个实现的过滤，多个的话使用第一个，Spring中相同名称的Bean只能有一个
        if (scanner.checkCandidate(beanName, serviceBeanDefinition)) { // check duplicated candidate bean
            //注册ServiceBean
            registry.registerBeanDefinition(beanName, serviceBeanDefinition);
            if (logger.isInfoEnabled()) {
                logger.info("The BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean has been registered with name : " + beanName);
            }
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("The Duplicated BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean[ bean name : " + beanName +
                        "] was be found , Did @DubboComponentScan scan to same package in many times?");
            }
        }
    }
```
    
 - scanner.scan得到的@Service注解下的Bean，可以直接通过Spring自动注入使用
 - registerServiceBean方法是用来处理对外发布的ServiceBean的

三、 下面解读，解析Dubbo自定义注解@Reference的类ReferenceAnnotationBeanPostProcessor
1. 先看下类继承关系
```
public class ReferenceAnnotationBeanPostProcessor extends AnnotationInjectedBeanPostProcessor<Reference>
        implements ApplicationContextAware, ApplicationListener
```
 好多实现逻辑要从AnnotationInjectedBeanPostProcessor说起，注意该泛型类引入的是注解@Reference；实现ApplicationListener接口就是监听Spring发布的事件，触发方法onApplicationEvent，但是看了这个类监听到事件后的处理逻辑，没什么用，就不说这个事件的代码了
2. 进入抽象类AnnotationInjectedBeanPostProcessor，看该类继承关系
```
public abstract class AnnotationInjectedBeanPostProcessor<A extends Annotation> extends
        InstantiationAwareBeanPostProcessorAdapter implements MergedBeanDefinitionPostProcessor, PriorityOrdered,
        BeanFactoryAware, BeanClassLoaderAware, EnvironmentAware, DisposableBean 
```
 - 实现接口MergedBeanDefinitionPostProcessor，会调用postProcessMergedBeanDefinition方法，该方法是所有相关BeanDefinition各自合并后，可以作为RootBeanDefinition来后置处理。方法中主要获取InjectionMetadata对象，并进行缓存，同时执行```metadata.checkConfigMembers(beanDefinition); ```将要进行注入调用的方法或字段InjectedElement存在metadata对象的Collection中，在进行metadata.inject时候调用。注意获取的InjectionMetadata对象是经过泛型类的泛型Reference注解过滤过的，即InjectedElement中的对象只有被标注过@Reference注解的。
 - 实现接口MergedBeanDefinitionPostProcessor，会让每个合并后的BeanDefinition来调用postProcessMergedBeanDefinition方法
 ```
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        if (beanType != null) {
            InjectionMetadata metadata = findInjectionMetadata(beanName, beanType, null);
            metadata.checkConfigMembers(beanDefinition);
        }
    }
```
 - 实现抽象类InstantiationAwareBeanPostProcessorAdapter，这个是Bean实例化前后要经过的，如Bean实例化前方法postProcessBeforeInstantiation，实例化后方法postProcessAfterInstantiation，BeanDefinition属性信息设置完后执行方法postProcessPropertyValues；Dubbo中就是在BeanDefinition属性信息设置完后，对Bean中带有@Reference注解的属性字段进行注入
 - Dubbo启动后，每个BeanDefinition先经过MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition，然后再执行InstantiationAwareBeanPostProcessorAdapter.postProcessPropertyValues
 ```
    @Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
        //获取类元数据信息，里面有字段和方法存在Collection<InjectedElement>中
        InjectionMetadata metadata = findInjectionMetadata(beanName, bean.getClass(), pvs);
        try {
            //下面方法是执行Spring内的方法，里面就是循环InjectedElement集合，
            //循环中调用InjectedElement.inject，就是执行类中字段的注入或方法的执行
            metadata.inject(bean, beanName, pvs);
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of @" + getAnnotationType().getName()
                    + " dependencies is failed", ex);
        }
        return pvs;
    }
```
 InjectionMetadata.inject源码如下，注意是Spring中的，不是Dubbo
 ```
	public void inject(Object target, String beanName, PropertyValues pvs) throws Throwable {
		Collection<InjectedElement> elementsToIterate =
				(this.checkedElements != null ? this.checkedElements : this.injectedElements);
		if (!elementsToIterate.isEmpty()) {
			boolean debug = logger.isDebugEnabled();
			for (InjectedElement element : elementsToIterate) {
				if (debug) {
					logger.debug("Processing injected element of bean '" + beanName + "': " + element);
				}
				element.inject(target, beanName, pvs);
			}
		}
	}
```
 - InjectedElement是InjectionMetadata中的抽象类，子类通过实现inject方法来达到字段注入和方法调用。Dubbo中继承了AnnotatedInjectionMetadata类AnnotatedInjectionMetadata，类中分别定义了字段和方法的InjectedElement集合。InjectedElement类型的字段是Dubbo实现抽象类InjectionMetadata.InjectedElement的AnnotatedFieldElement类型，方法是AnnotatedMethodElement类型。两种类型分别实现了InjectedElement的inject方法
 字段注入源码如下
 ```
        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
            Class<?> injectedType = field.getType();
            Object injectedObject = getInjectedObject(annotation, bean, beanName, injectedType, this);
            ReflectionUtils.makeAccessible(field);
            field.set(bean, injectedObject);
        }
```
 方法调用源码如下
 ```
        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
            Class<?> injectedType = pd.getPropertyType();
            Object injectedObject = getInjectedObject(annotation, bean, beanName, injectedType, this);
            ReflectionUtils.makeAccessible(method);
            method.invoke(bean, injectedObject);
        }
```
 - 简单的可以理解：Dubbo就是通过实现InstantiationAwareBeanPostProcessorAdapter.postProcessPropertyValues方法，处理每个Bean的字段，根据自定义注解进行属性注入，类似@Autowired的实现，下面先贴出一段完整案例代码，只是案例
 ```
@Component
public class TestInstantiationAwareBeanPostProcessorAdapter extends InstantiationAwareBeanPostProcessorAdapter {
    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
        if (beanName.contains("Demo")) {
            List<InjectionMetadata.InjectedElement> injectedElements = getInJectedElement(bean, pvs);
            InjectionMetadata metadata = new InjectionMetadata(bean.getClass(), injectedElements);
            try {
                metadata.inject(bean.getClass(), beanName, pvs);
            } catch (Throwable ex) {
                ex.printStackTrace();
            }
        }
        return super.postProcessProperties(pvs, bean, beanName);
    }
    private List<InjectionMetadata.InjectedElement> getInJectedElement(Object bean, PropertyValues pvs) {
        List<InjectionMetadata.InjectedElement> injectedElements = new LinkedList<>();
        ReflectionUtils.doWithFields(bean.getClass(), new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
                Annotation annotation = getAnnotation(field, Autowired.class);
                if (annotation != null) {
                    if (Modifier.isStatic(field.getModifiers())) {
                        return;
                    }
                    injectedElements.add(new InjectionMetadata.InjectedElement(field, null) {
                        @Override
                        protected void inject(Object target, String requestingBeanName, PropertyValues pvs) throws Throwable {
                            //根据Cglib或JDK获取代理对象
                            TestInterface testInterface=new TestBeanFactory<TestInterface>(TestInterface.class).getObject();
                            field.set(bean, testInterface);
                        }
                    });
                }
            }
        });
        return injectedElements;
    }
}
```
 - Dubbo中执行@Reference注解的属性字段注入时，要先获取代理对象，那就从方法getInjectedObject(annotation, bean, beanName, injectedType, this)进入，最终又到ReferenceAnnotationBeanPostProcessor.doGetInjectedBean方法
 ```
    @Override
    protected Object doGetInjectedBean(Reference reference, Object bean, String beanName, Class<?> injectedType,
                                       InjectionMetadata.InjectedElement injectedElement) throws Exception {
        String referencedBeanName = buildReferencedBeanName(reference, injectedType);
        ReferenceBean referenceBean = buildReferenceBeanIfAbsent(referencedBeanName, reference, injectedType, getClassLoader());
        cacheInjectedReferenceBean(referenceBean, injectedElement);
        Object proxy = buildProxy(referencedBeanName, referenceBean, injectedType);
        return proxy;
    }
```
 buildProxy(referencedBeanName, referenceBean, injectedType)源码如下，使用的是JDK动态代理
 ```
    private Object buildProxy(String referencedBeanName, ReferenceBean referenceBean, Class<?> injectedType) {
        InvocationHandler handler = buildInvocationHandler(referencedBeanName, referenceBean);
        Object proxy = Proxy.newProxyInstance(getClassLoader(), new Class[]{injectedType}, handler);
        return proxy;
    }
```
 再继续跟踪，就是获取JDK动态代理的Handler了，这个Handler就是@Reference注解的对象调用方法时，实际调用的就是这个代理的Handler里的invoke方法
 ```
@Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Object result = null;
            try {
                if (bean == null) { // If the bean is not initialized, invoke init()
                    // issue: https://github.com/apache/incubator-dubbo/issues/3429
                    init();
                }
                result = method.invoke(bean, args);
            } catch (InvocationTargetException e) {
                // re-throws the actual Exception.
                throw e.getTargetException();
            }
            return result;
        }
```
 上面invoke中的bean其实还是一个代理，也就是说还会进入另一个代理的hanlder.invoke方法，bean对象可以从init()中得到来源，一路跟踪，是在ReferenceConfig中获取到的对象，ReferenceConfig中createProxy方法就是根据Dubbo配置信息得到代理对象
 ```
        // create service proxy
        //invoker会根据不同协议得到不同对象
        return (T) proxyFactory.getProxy(invoker);
```
 上面的proxyFactory对象来自SPI
 ```
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```