一、加载机制概述
Dubbo中几乎所有功能组件都是基于扩展机制(SPI)实现的。Dubbo SPI没有直接使用Java SPI，而是在它思想上做了改进。Dubbo有自己的配置规范和特性，同时又兼容Java SPI。
1. Java SPI(Service Provider Interface)使用了策略模式，一个接口多种实现。只声明接口，而具体实现并不在程序中直接确定，而是在程序之外配置。步骤如下
 - 定义一个接口及其对应方法
 - 编写接口实现类，可以有多个
 - 在META-INF/services目录下，创建一个以接口全路径命名的文件
 - 文件内容为具体实现类的全路径名，多个用换行符分隔
 - 在代码中通过java.util.ServiceLoader来加载具体的实现类

2. 扩展点机制改进，Dubbo SPI，改进点如下：
 - JDK SPI会一次性实例化扩展点所有实现，如果有扩展点初始化很耗时，或者没有用也加载，则很浪费资源，Dubbo SPI只是加载配置文件中的类，缓存起来，并不立即全部初始化
 - JDK SPI多扩展实现导致的异常，不能明确的给出提示
 - Dubbo SPI增加了对扩展IoC和AOP的支持

 Dubbo SPI实现Demo
 - 接口添加SPI注解，并指定接口默认实现类Key
 ```
@SPI("impl")
public interface PrintService {
    void printInfo();
}
```
 - META-INF/services下接口全路径名文件内内容
 ```
impl=com.spiinterface.PrintServiceImp
```
 - 主类中调用测试
 ```
    public static void main(String[] args) {
        PrintService printService = ExtensionLoader.getExtensionLoader(PrintService.class)
                .getDefaultExtension();
        printService.printInfo();
    }
```
3. Dubbo SPI扩展点配置规范
 - SPI配置文件名称：默认扫描/META-INF/services/、/META-INF/dubbo/、/META-INF/dubbo/internal/三个目录
 - SPI配置文件名称：全路径接口名称
 - 文件内容格式：key=value方法，key为SPI注解中传入参数，value为扩展点实现类全路径名，多个用换行符分隔

4. Dubbo SPI可以分为Class缓存、实例缓存
 - Class缓存：获取扩展类时，先从缓存中获取，如果没有就再根据配置文件加载，并将Class缓存起来
 - 实例缓存：基于性能考虑，Dubbo还会缓存Class实例化后的对象。每次获取时候，先从缓存中获取，没有就重新加载并缓存起来

 根据扩展类的种类还可以分为如下类：
 - 普通扩展类：就是配置在SPI配置文件中的普通扩展实现类
 - 包装扩展类：这种Wrapper类没有具体实现，只做了通用逻辑抽象，需要在构造方法中传入一个具体的扩展接口的实现
 - 自适应扩展类：一个扩展接口会有多个实现类，具体使用哪个，可以不写死在配置文件或代码中，通过传入URL中的某些参数来动态确定
 - 其他缓存，如扩展类加载器缓存、扩展名缓存等

5. 扩展点特性
 - 自动包装：ExtensionLoader在加载扩展时，如果发现这个扩展类包含其他扩展点作为构造函数的参数，则这个扩展类就会被认为是Wrapper类，其实就是一种装饰模式，对同接口类的一种增强。继续使用Dubbo SPI Demo
    - 创建一个Wrapper类实现扩展点接口PrintService，注意类中要有扩展点接口为参数的构造函数
 ```
    public class PrintServiceWrapper implements PrintService {
        private PrintService printService;
        public PrintServiceWrapper(PrintService printService) {
            this.printService = printService;
        }
        @Override
        public void printInfo() {
            System.out.println("前");
            printService.printInfo();
            System.out.println("后");
        }
    }
```
    - 对应的SPI配置文件添加上该Wrapper类的全路径名，key使用不使用都可以，只要有该包装类配置，Dubbo SPI就会加载使用
 ```
impl=com.spiinterface.PrintServiceImp
#filter=com.hujiapeng.PrintServiceWrapper
com.hujiapeng.PrintServiceWrapper
```
     - 测试结构如下
 ```
前
printservice
后
```
     - 如果有多个包装类，那多个都会执行，其原理就是根据构造函数参数类型来获取构造函数，同时传入参数创建实例；Dubbo中就把新的实例对象返回
 ```
      Set<Class<?>> wrapperClasses = cachedWrapperClasses;
      if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
          for (Class<?> wrapperClass : wrapperClasses) {
              //多个wrapperClass会循环创建实例，每次传入的就是上一次创建的
              //所以最终返回的是最后一个被包装的实例，而构造函数传入的参数是上一个实例
              instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
          }
      }
      return instance;
```
 - 自动加载：如果某个扩展点(带有@SPI注解的接口)是另一个扩展点实现类的成员属性，并且该实现类有另一个扩展点的setter方法，那么框架会自动注入对应的实例。个人对这个单独做实验失败，总是依赖@Adaptive，但是这个属于自适应的，最终还是练习了自适应，没有用这个
 - 自适应：Dubbo SPI中，使用@Adaptive注解，可以动态地通过URL中的参数来确定要使用哪个具体实现类，来解决自动加载中的实例注入问题。注意，自适应的扩展点对应的方法要带有URL参数```com.alibaba.dubbo.common.URL ```
    - @Adaptive如果在类上，表示在其他扩展点注入当前扩展点类型属性时，特定使用的扩展点实现类。所以一个扩展点只能有一个带有@Adaptive注解的类
    - @Adaptive如果用在扩展点(接口)方法上，要使用方法参数中的URL参数，当@Adaptive没有参数时，就会把接口名转化为参数传入(如接口名为SimpleExt，转化为simple.ext)，如果无法找到扩展点实现类，就使用@SPI注解中参数，@Adaptive和@SPI中都没有参数时，就会抛异常；如果@Adaptive中有参数，一个或多个，那就会根据参数顺序依次查找对应的扩展点实现类，直到找到一个即可，如果都没有，那就使用@SPI中的参数，如果都无法匹配则抛异常
    - 下面展示Demo
       定义扩展点
 ```
     @SPI("adaptimpl")
     public interface AdaptivePrintService {
         @Adaptive({"adaptiveKey"})
         void printInfo(URL url);
     }
```
      其他扩展点实现类中使用
 ```
    public class PrintServiceImp implements PrintService {
       private AdaptivePrintService anotherPrintService;
        public void setAnotherPrintService(AdaptivePrintService anotherPrintService) {
            this.anotherPrintService = anotherPrintService;
        }
        @Override
        public void printInfo() throws MalformedURLException {
            Map<String,String> params=new HashMap<>(1);
            params.put("adaptiveKey","adaptive");
            URL url=new URL("dubbo","localhost",0,params);
            anotherPrintService.printInfo(url);
            System.out.println("printservice");
        }
    }
```
      SPI配置文件com.hujiapeng.AdaptivePrintService
 ```
adaptimpl=com.hujiapeng.AdaptivePrintServiceImpl
adaptive=com.hujiapeng.AdaptivePrintServiceImplAnother
```

 - 自动激活：使用@Activate注解，可以标记对应的扩展点默认被激活启用。该注解还可以通过传入不同参数，设置扩展点在不同条件下被自动激活。主要使用场景是某个扩展点的多个实现类需要同时启用

二、扩展点注解
1. @SPI：可以标记接口为Dubbo SPI接口，注解中的参数可以为接口指定默认实现类
2. @Adaptive：用在类上，是指定接口对应的自适应类，用来在其他扩展点实现类中的属性自动注入；用在扩展点接口方法上，则可以通过参数动态获得实现类。方法级别的注解，在第一次调用getExtension时，会自动生成和编译一个动态的Adaptive类，从而达到动态实现类的效果。
如Dubbo初始化扩展点时，会生成一个Transport$Adaptive类，这个动态类里会实现接口中带有@Adaptive注解的方法，方法中就是使用ExtensionLoader.getExtensionLoader方法获取解析到的扩展点实现类，然后再调用接口方法。Dubbo中使用cacheAdaptiveClass缓存Adaptive具体实现类，cacheAdaptiveInstance缓存Class的具体实例化对象。
3. @Activate：一个扩展点的多个实现，同时都要使用，类似Java SPI获取到多个扩展点实现。Dubbo SPI的@Activate，如果没有参数，就默认激活，如果有参数，就根据参数条件判断是否激活。注意扩展点要加注解@SPI、还要有SPI配置文件
 - @Activate注解在类上，如果没有参数则默认激活；如果有参数就按照URL中使用的Key来判断
 ```
        Map<String, String> params = new HashMap<>(1);
        params.put("activateKey", "activate");
        URL url = new URL("dubbo", "localhost", 0, params);
        List<ActivatePrintService> activateExtension = ExtensionLoader.getExtensionLoader(ActivatePrintService.class)
                .getActivateExtension(url, "");
```
 - getActivateExtension方法中如果指定了第二个参数，那就先根据第二个参数查找SPI配置中key对应的扩展点实现类，如果第二个参数不为空字符串，那么SPI配置中要有对应的key，否则抛异常；如果根据第二个参数指定的实现类不符合@Activate条件，那么就会加载扩展点多有实现类直到找到符合条件的，如果依然没有，则返回空集合

三、ExtensionLoader的工作原理
1. 工作流程，如下图
![](https://raw.githubusercontent.com/hujiapeng/imgs/master/readingnotes/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ApacheDubbo%E4%B8%8E%E5%AE%9E%E6%88%98/04Dubbo%E6%89%A9%E5%B1%95%E7%82%B9%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/ExtensionLoader%E6%80%BB%E4%BD%93%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.jpg)
2. ExtensionFactory实现原理
 - ExtensionFactory接口源码如下
 ```
@SPI
public interface ExtensionFactory {
    <T> T getExtension(Class<T> type, String name);
}
```
 - ExtensionFactory接口有三个实现类，AdaptiveExtensionFactory、SpiExtensionFactory和SpringExtensionFactory
    - AdaptiveExtensionFactory类上有@Adaptive注解，所以该类为接口默认实现类，这个类会把其他两个类加入集合List<ExtensionFactory> factories中，源码如下
 ```
      public AdaptiveExtensionFactory() {
          ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
          List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
          for (String name : loader.getSupportedExtensions()) {
              list.add(loader.getExtension(name));
          }
          factories = Collections.unmodifiableList(list);
      }
```
    - loader.getSupportedExtensions()可以获取到接口对应SPI配置中的普通扩展点实现类的key，wrapper类的key获取不到

四、扩展点动态编译实现
1. Dubbo SPI的自适应中，如果@Adaptive在方法上，Dubbo内部会自动生成获取扩展点实现类的代码，这就用到了动态编译，Dubbo中有三种，分别是JDK编译器、Javassist编译器和AdaptiveCompiler编译器，都是实现接口com.alibaba.dubbo.common.compiler.Compiler
2. AdaptiveCompiler类上有注解@Adaptive，所以这个类会是默认实现类，先使用该类，该类compile方法中会使用扩展点实现类的key获取对应的扩展点实现类，注意该类属性DEFAULT_COMPILER，首先使用<dubbo:application compiler="jdk"/>或ApplicationConfig Bean中的配置，如果没有则使用接口上的@SPI("javassist")
3. Javassist动态代码编译，字节码生成工具有CGLIB、ASM、Javassist等
 ```
    public static void main(String[] args) throws Exception {
        //初始化Javassist的类池
        ClassPool classPool = ClassPool.getDefault();
        //创建一个Hello World类
        CtClass ctClass = classPool.makeClass("Hello World");
        //添加一个test方法，会打印Hello World
        CtMethod ctMethod = CtNewMethod.make("" +
                "public static void test(){" +
                "System.out.println(\"Hello World\");" +
                "}", ctClass);
        ctClass.addMethod(ctMethod);
        //生成类
        Class aClass = ctClass.toClass();
        //反射生成类实例
        Object object = aClass.newInstance();
        Method method = aClass.getDeclaredMethod("test", null);
        method.invoke(object, null);
    } 
```
4. JDK动态代码编译，主要使用JavaFileObject接口、ForwardingJavaFileManager接口、JavaCompiler.CompilationTask方法
 ```
    public static void main(String[] args) throws Exception {
        JavaCompiler systemJavaCompiler = ToolProvider.getSystemJavaCompiler();
        StandardJavaFileManager standardFileManager = systemJavaCompiler.getStandardFileManager(null, null, null);
        String javaFile="F:\\JavaCodeProject\\demo\\spidemo\\src\\main\\java\\com\\HelloWorld.java";
        Iterable<? extends JavaFileObject> javaFileObjects = standardFileManager.getJavaFileObjects(javaFile);
        JavaCompiler.CompilationTask task = systemJavaCompiler.getTask(null, standardFileManager, null, null, null, javaFileObjects);
        task.call();
        standardFileManager.close();
        //创建实例
        Class<?> aClass = JdkDynCompileDemo.class.getClassLoader().loadClass("com.HelloWorld");
        Object object = aClass.newInstance();
        Method method = aClass.getDeclaredMethod("test", null);
        method.invoke(object, null);
    }
```
