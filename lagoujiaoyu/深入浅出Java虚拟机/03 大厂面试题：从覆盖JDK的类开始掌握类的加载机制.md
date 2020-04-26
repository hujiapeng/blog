1. 类加载过程：加载、链接(验证、准备、解析)、初始化
 - 加载：加载主要作用就是将外部的.class二进制文件，加载到Java的方法区内；
 - 验证：符合JVM规范的.class文件才能加载，否则将抛出java.lang.VerifyError错误。还有就是低版本JVM无法加载一些高版本类库；
 - 准备：为类变量分配内存，并将其初始化为默认值。注意，基本类型类变量会在准备阶段赋初始值，局部变量依赖实例，这些动作是在方法区上进行的，此时实例对象还未创建，所以基本类型局部变量如果没有初始值无法通过编译；
 - 解析：加载过程中将符号引用替换为直接引用的过程。符号引用可以是任何字面上的含义，而直接引用就是直接指向目标的指针、相对偏移量，直接引用的对象都存在于内存中。可以简单理解为解析阶段就是将表示类和方法的字面表示解析为准确指向的内存地址；
 - 初始化：对类变量初始化及执行静态语句块
    - 规则一：静态语句块只能访问到定义在static语句块之前的变量。如下
```
public class A{
    static int a=0;
    static {
        a=1;
        //此处是类的初始化赋值，不会报错
        b=1;
    }
    static int b=0; 
    public static void main(String[] args) {
        System.out.println(a + " " + b);//输出1 0
    }
    //static int b; 
    //public static void main(String[] args) {
        //System.out.println(a + " " + b);//输出1 1
    //}
}
```
如下使用变量就会报错，在静态语句块内访问了定义在static语句块后的变量。此处要注意的是，定义变量及初始赋值是在准备阶段，而操作变量是在初始化阶段，操作必须发生在定义之后。
```
public class A{
    static int a=0;
    static {
        a=1;
        //此处是类的初始化赋值，不会报错
        b=b+1;
    }
    static int b=0; 
   ······
}
```
    - 规则二：JVM会保证在子类的初始化方法执行前，父类的初始化方法已经执行完毕。所以JVM第一个被执行的类初始化方法一定是java.lang.Object，另外也就意味着对于类中的static语句块，父类优先于子类。
    ```<cinit>与<init> ```：```<cinit> ```方法对应的是类初始化，在类加载的初始化阶段就被执行了；```<init> ```方法对应的是对象初始化，在**调用构造方法**来新建对象的时候执行，用来初始化对象的属性。

      ![](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/%E7%B1%BB%E5%8F%8A%E5%AF%B9%E8%B1%A1%E5%88%9D%E5%A7%8B%E5%8C%96.png)
2. 类加载器
 - Bootstrap ClassLoader：启动类加载器由C++编写，随JVM启动。用来加载核心类库，如rt.jar、resource.jar、charsets.jar等，这些jar包的路径也可以通过-Xbootclasspath参数指定。
 - Extension ClassLoader：扩展类加载器，主要用于加载lib/ext目录下的jar包和.class文件。也可以通过系统变量java.ext.dirs指定这个目录。Extension ClassLoader是个Java类，继承自URLClassLoader。
 - App ClassLoader：Java默认加载器，有时也叫System ClassLoader。一般用来加载classpath下的jar包和.class文件。我们写的代码会首先尝试使用这个类加载器进行加载。
 - Custom CLassLoader：自定义加载器，支持一些个性化扩展功能。

3. 双亲委派机制：除了顶层的启动类加载器以外，其他类加载器，在加载类之前，都会委派给他的父加载器进行加载。这样一层层向上传递，直到祖先们都无法加载，它才会真正加载。
4. 打破双亲委派机制的自定义加载器案例
 - 案例一：tomcat：对于加载非基础类，是由一个叫作WebAppClassLoader的类加载器优先加载，等加载不到时候，才会交给上层的ClassLoader进行加载。这个加载器用来隔绝不同应用的.class文件。
 ![](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/tomcat%E7%B1%BB%E5%8A%A0%E8%BD%BD%E8%BF%87%E7%A8%8B.png)
 tomcat中WebAppClassLoader加载自己目录下的.class文件，并不会传递给父类加载器，但是可以使用SharedClassLoader所加载的类，实现了共享和分离的功能。
 注意，即使自己写一个基础类，如ArrayList，放在应用目录里，tomcat依然不会加载，这个是因为自定义加载器顺序不同。
 - 案例二：SPI：全称Service Provider Interface，是Java提供的一套用来被第三方实现或者扩展的API，可以用来启用框架扩展和替换组件。如JDBC驱动，一般加载驱动类方式为Class.forName("com.mysql.jdbc.Driver")。但是如果使用mysql-connector-java-8.0.15.jar，即使删除Class.forName代码，也能加载到正确的驱动类。这是因为使用了SPI机制，在jar包下可以找到META-INF/services目录，然后有一个以接口java.sql.Driver为限定名命名的文件，内容为这个接口实现类com.mysql.cj.jdbc.Driver。
 SPI实际上是“基于接口的编程+策略模式+配置文件”组合实现的动态加载机制，主要使用java.util.ServiceLoader类进行动态装载。
 DriverManager类和ServiceLoader类都属于rt.jar，他们的类加载器是BootStrap ClassLoader，即启动类加载器。而数据库驱动属于业务代码，无法使用启动类加载器加载。通过查看DriverManager类源码，找到如下代码
 ```
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
```
 进入ServiceLoader.load方法，找到如下代码
 ```
public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
```
 可以看出在调用ServiceLoader.load(service, cl)时，使用的类加载器是当前线程的类加载器。通过设置断点，看出当前线程是来自Launcher类，在Launcher类构造函数中，找到如下代码
 ```
 try {
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }
        Thread.currentThread().setContextClassLoader(this.loader);
```
 由此看出ServiceLoader加载类使用的类加载器为应用类加载器。使用应用类加载器加载第三方驱动就没有什么问题了。
 SPI简单案例代码可以参考```https://blog.csdn.net/top_code/article/details/51934459```
 - 案例三：OSGi：是服务平台的规范，旨在用于需要长运行时间、动态更新和对运行环节破坏最小的系统。OSGi类加载器基于OSGi规范和每个绑定包的manifest.mf文件中指定的选项，来限定这些类的交互。OSGi实现了模块化服务，每个模块可以独立安装、启动、停止、卸载。目前使用OSGi的已经不多。

5. 如何替换JDK的类：使用endorsed技术，JVM启动时指定参数-Djava.endorsed.dirs，这个目录下的jar包会比rt.jar中的文件优先加载。注意java.lang包下的类不能替换，这个包是被特殊保护的。
更换HashMap类为例，本地建一个项目，自定义HashMap的包名及类名要和rt.jar下的一样。我的做法就是先完全复制源码文件到自定义的HashMap类中。然后在put方法出做一个打印输出，来区别确实调用自定义的HashMap。如下，开始没有做key的判断，报了空指针，断点后才知，这个HashMap在JVM启动过程中会被使用加载其他。加了判断后，将项目打成jar包，放到某目录下
```
  public V put(K key, V value) {
        if(key.equals("a")){
            System.out.println("Custom HashMap");
        }
        return putVal(hash(key), key, value, false, true);
    }
```
本地main方法如下，启动前配置JVM参数-Djava.endorsed.dirs=自定义HashMap的jar**目录**路径
```
    public static void main(String[] args) {
        HashMap map = new HashMap();
        map.put("a", "a");
        map.put("b", "b");
        map.forEach((key, value) -> System.out.println(key + "-" + value));
    }
```
运行结果如下
```
Custom HashMap
a-a
b-b
```
 