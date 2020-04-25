一、 Dubbo发布服务入口是借助Spring的事件机制，监听ContextRefreshedEvent事件
1. 当所有Spring Bean加载完成，AbstractApplicationContext.finishRefresh中会发布ContextRefreshedEvent事件。finishRefresh方法的调用其实来自AnnotationConfigApplicationContext对象的创建，会调用其中的refresh方法
 ```
// Publish the final event.
publishEvent(new ContextRefreshedEvent(this));
```
2. ServiceBean实现了接口ApplicationListener<ContextRefreshedEvent>，监听了ContextRefreshedEvent事件，export()就开始发布服务了。根据@Servivce可以有多个ServiceBean，也就是会执行多次下面事件方法，注册的ServiceBean如```ServiceBean:com.hujiapeng.echo.api.EchoService ```，格式为ServiceBean:接口引用路径
 ```
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (isDelay() && !isExported() && !isUnexported()) {
            if (logger.isInfoEnabled()) {
                logger.info("The service ready on spring started. service: " + getInterface());
            }
            export();
        }
    }
```
3. export()代码如下，主要看super.export()，publishExportEvent()是发出一个事件，到了ReferenceAnnotationBeanPostProcessor.initReferenceBeanInvocationHandler，没看出有什么用，就不解析了。
 ```
    @Override
    public void export() {
        super.export();
        // Publish ServiceBeanExportedEvent
        publishExportEvent();
    }
```
4. super.export()调用的是ServiceConfig下的export，该方法前面的属性信息是在ServiceBean初始化完后，通过实现接口InitializingBean，执行了afterPropertiesSet方法处理的。这里主要看export中的doExport方法，doExport方法前一部分是做配置校验，直接进入doExportUrls()方法
 ```
    private void doExportUrls() {
        //获取注册中心
        List<URL> registryURLs = loadRegistries(true);
        for (ProtocolConfig protocolConfig : protocols) {
            //向注册中心发布该协议的服务
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```
5. loadRegistries(true)方法就是获取配置中的注册中心，注意如下代码，即系统属性配置优于代码或配置文件的配置地址
 ```
 for (RegistryConfig config : registries) {
     String address = config.getAddress();
     if (address == null || address.length() == 0) {
         address = Constants.ANYHOST_VALUE;
     }
     String sysaddress = System.getProperty("dubbo.registry.address");
     if (sysaddress != null && sysaddress.length() > 0) {
         address = sysaddress;
     }
```
6. 进入doExportUrlsFor1Protocol(protocolConfig, registryURLs)方法，有三个地方进行发布服务，都是包装成Invoker对外发布
 - 如果没有配置remote，就发布本地服务
 ```
 // export to local if the config is not remote (export to remote only when config is remote)
 if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
     exportLocal(url);
}
```
 - 如果没配置发布本地服务，就发布其他服务
 ```
     // export to remote if the config is not local (export to local only when config is local)
     if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
         if (registryURLs != null && !registryURLs.isEmpty()) {
             //如果有注册中心，就发布到注册中心
             Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
             DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
             Exporter<?> exporter = protocol.export(wrapperInvoker);
             exporters.add(exporter);
          ·······
          }else{
             //如果没有注册中心，就直接发布
             Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
             DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
             Exporter<?> exporter = protocol.export(wrapperInvoker);
             exporters.add(exporter);
          }
```
7. 先看发布本地服务，其中就是把URL协议、Host和端口进行修改成本地，然后通过proxyFactory获取到对应invoker，最后通过protocol把invoker发布出去
 ```
    private void exportLocal(URL url) {
        if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
            URL local = URL.valueOf(url.toFullString())
                    .setProtocol(Constants.LOCAL_PROTOCOL)
                    .setHost(LOCALHOST)
                    .setPort(0);
            StaticContext.getContext(Constants.SERVICE_IMPL_CLASS).put(url.getServiceKey(), getServiceClass(ref));
            Exporter<?> exporter = protocol.export(
                    proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
            exporters.add(exporter);
            logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
        }
    }
```
8. 看非本地服务发布代码，也是通过proxyFactory获取到对应invoker，最后通过protocol把invoker发布出去
 ```
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
Exporter<?> exporter = protocol.export(wrapperInvoker);
```
9. 暂时理解invoker里封装的是服务接口和接口实现类等基本信息，获取invoker主要是靠proxyFactory，这个代理工厂是靠Dubbo自定义的SPI实现的
 ```
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```
10. protocol也是靠Dubbo自定义的SPI来实现的
 ```
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```
