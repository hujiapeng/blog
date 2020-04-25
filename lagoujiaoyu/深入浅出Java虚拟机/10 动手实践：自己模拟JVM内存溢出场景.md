1. JVM堆中，年轻代如果满了，可以晋升到老年代，老年代如果满了，那就只能OOM了，JVM会在这种情况下直接停止工作。OOM一般是内存泄漏引起的，在GC日志里的表现一般就是GC的时间变长了。
2. 堆溢出模拟
 代码中开放一个Http接口，通过访问http://localhost:8888/，将每秒产生1MB数据。由于是和GC Roots强关联，每次都不能被回收。程序通过JMX(Java Management Extensions，即Java管理扩展)在每一秒创建数据之后，输出一些内存区域的占用情况。
 Java环境为JDK8，JVM参数为：
```
-Xmx20m
-Xms20m
-Xmn4m
-verbose:gc
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintTenuringDistribution
-Xloggc:G:/jvmlogs/gc_%p.log
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=G:/jvmlogs
-XX:ErrorFile=G:/jvmlogs/hs_error_pid%p.log
```
 代码如下：
```
public class OOMTest {
    public static final int _1MB = 1024 * 1024;
    static List<byte[]> byteList = new ArrayList<>();
    private static void oom(HttpExchange exchange) {
        try {
            String response = "oom begine!";
            exchange.sendResponseHeaders(200, response.getBytes().length);
            OutputStream os = exchange.getResponseBody();
            os.write(response.getBytes());
            os.close();
        } catch (Exception ex) {
            ex.printStackTrace();
        }
        for (int i = 0; ; i++) {
            byte[] bytes = new byte[_1MB];
            byteList.add(bytes);
            System.out.println(i + "MB");
            memPrint();
            try {
                Thread.sleep(1000);
            } catch (Exception ex) {

            }
        }
    }
    private static void memPrint() {
        for (MemoryPoolMXBean memoryPoolMXBean : ManagementFactory.getMemoryPoolMXBeans()) {
            System.out.println(memoryPoolMXBean.getName() + " committed:" +
                    memoryPoolMXBean.getUsage().getCommitted() + " used:" + memoryPoolMXBean.getUsage().getUsed());
        }
    }
    private static void srv() throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress(8888), 0);
        HttpContext context = server.createContext("/");
        context.setHandler(OOMTest::oom);
        server.start();
    }
    public static void main(String[] args) throws Exception {
        srv();
    }
}
```
 通过VisualVM的Visual GC插件观察的话，开始Eden区运行平稳，内存泄漏后疯狂回收，也就是提升到老年代，老年代内存一直增长，直到OOM。
 JVM中常用参数说明：
 - NewRatio：默认值为2，表示年轻代是老年代的1/2，设置-XX:NewRatio=1的话，就是把年轻代和老年代的空间大小调成一样。实践中，一般使用-Xmn来指定年轻代大小。注意这两个参数不要用在G1垃圾回收器中，因为G1是自动分配年轻代和老年代空间大小的，如果设置了参数，则自动分配失效。
 - SurvivorRatio默认值为8，表示伊甸区和幸存区的比例
 - MaxTenuringThreshold：年轻代对象提升到老年代的年龄限制，CMS中默认为6，G1下默认为15。其实最大值为15，因为存储这个值的内存长度为4bit。
 - PretenureSizeThreshold：这个参数默认为0，表示所有的对象年轻代优先分配。如果设置一定大小的值，则表示超过这个大小的对象分配到老年代。
 - TargetSurvivorRatio：默认值50。在动态计算对象提升阈值的时候使用。计算时，会从年龄最小的对象开始累加，如果累加的对象大小大于幸存区的一半，则将当前对象的age作为新的阈值，年龄大于此阈值的对象直接进入老年代。不建议修改此值，如果要改，也要调成比50大的值。
 - UseAdaptiveSizePolicy：和CMS不兼容，所以在CMS下默认为false，G1下默认为true。注意CMS中不能使用。这个参数是用来自适应调整空间大小的，每次GC后，重新计算Eden、From、To的大小。
 - G1HeapRegionSize：G1垃圾收集器中的参数，用来调整小堆区大小的
3. 元空间溢出
 元空间溢出主要是由于加载的类太多，或者动态生成的类太多。如果不指定元空间大小，那么元空间溢出会造成操作系统内存溢出，所以使用的时候要指定元空间大小。
 Java环境为JDK8，JVM参数为：
```
-XX:MetaspaceSize=16m
-XX:MaxMetaspaceSize=16m
-verbose:gc
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintTenuringDistribution
-Xloggc:G:/jvmlogs/gc_%p.log
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=G:/jvmlogs
-XX:ErrorFile=G:/jvmlogs/hs_error_pid%p.log
```
 代码如下：
```
public class MetaspaceOOMTest {
    public interface Facade {
        void m(String input);
    }
    public static class FacadeImpl implements Facade {
        @Override
        public void m(String input) {

        }
    }
    public static class MetaspaceFacadeInvocationHandler implements InvocationHandler {
        private Object impl;
        public MetaspaceFacadeInvocationHandler(Object impl) {
            this.impl = impl;
        }
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            return method.invoke(impl, args);
        }
    }
    private static Map<String, Facade> classLeakingMap = new HashMap<>();
    private static void oom(HttpExchange exchange) {
        try {
            String response = "oom begin!";
            exchange.sendResponseHeaders(200, response.getBytes().length);
            OutputStream os = exchange.getResponseBody();
            os.write(response.getBytes());
            os.close();
        } catch (Exception ex) {
        }
        try {
            for (int i = 0; ; i++) {
                String jar = "file:" + i + ".jar";
                URL[] urls = new URL[]{new URL(jar)};
                URLClassLoader newClassLoader = new URLClassLoader(urls);
                Facade t = (Facade) Proxy.newProxyInstance(newClassLoader, new Class<?>[]{Facade.class},
                        new MetaspaceFacadeInvocationHandler(new FacadeImpl()));
                classLeakingMap.put(jar, t);
            }
        } catch (Exception ex) {
        }
    }
    private static void srv() throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress(8888), 0);
        HttpContext context = server.createContext("/");
        context.setHandler(MetaspaceOOMTest::oom);
        server.start();
    }
    public static void main(String[] args) throws Exception {
        srv();
    }
}
```
4. 堆外内存溢出
 严格来说元空间内存溢出也属于堆外内存溢出。此处是指Java应用程序通过直接方式从操作系统申请内存。程序通过ByteBuffer的allocateDirect方法每秒申请1MB的直接内存。这个过程通过VisualVM和JMX是无法看到内存使用情况的。
```
public class OffHeapOOMTest {
    public static final int _1M = 1024 * 1024;
    static List<ByteBuffer> byteList = new ArrayList<>();
    private static void oom(HttpExchange exchange) {
        try {
            String response = "oom begin!";
            exchange.sendResponseHeaders(200, response.getBytes().length);
            OutputStream os = exchange.getResponseBody();
            os.write(response.getBytes());
            os.close();
        } catch (Exception ex) {
        }
        for (int i = 0; ; i++) {
            ByteBuffer buffer = ByteBuffer.allocateDirect(_1M);
            byteList.add(buffer);
            System.out.println(i + "MB");
            memPrint();
            try {
                Thread.sleep(1000);
            } catch (Exception ex) {
            }
        }
    }
    private static void memPrint() {
        for (MemoryPoolMXBean memoryPoolMXBean : ManagementFactory.getMemoryPoolMXBeans()) {
            System.out.println(memoryPoolMXBean.getName() + " committed:" + memoryPoolMXBean.getUsage().getCommitted() +
                    " used:" + memoryPoolMXBean.getUsage().getUsed());
        }
    }
    private static void srv() throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress(8888), 0);
        HttpContext context = server.createContext("/");
        context.setHandler(OffHeapOOMTest::oom);
        server.start();
    }
    public static void main(String[] args) throws Exception {
        srv();
    }
}
```
 通过top或其他操作系统工具可以看到内存占用明显增长。为了限制这些危险的内存申请，要显示设置MaxDirectMemorySize=<N>参数.
5. 栈溢出
 可通过参数-Xss指定。递归调用层次过深就容易导致栈溢出
6. 进程异常退出
 操作系统随着使用内存越用越多，第一层防护墙就是SWAP；当SWAP用的差不多了，会尝试释放Cache；当这两者资源都耗尽，oom-killer就会在系统内存耗尽的时候选择性的干掉一些进程以求释放一点内存。这种情况下终结的Java进程，JVM里是没有日志记录的，只能到操作系统日志里查找。
 有的程序里通过调用System.exit()终止系统，这种情况下也不会留下日志信息。所以不推荐使用。
 有时候通过XShell或其他连接Linux的客户端通过&符号在后台启动一个应用程序，如```java -jar com.cn.AA.jar &```，但是很多情况下，关闭这个客户端后，程序就退出了，这是因为启动的程序是在当前客户端连接下创建的线程，当前线程退出，子线程退出。正确的方式是使用nohup，方式如```nohup java -jar com.cn.AA.jar &```。
 Linux中终止程序使用kill命令，不建议使用kill -9 pid，因为这种命令是强制终止进程，不会清理善后；建议使用kill pid(即，默认为kill -15 pid)。