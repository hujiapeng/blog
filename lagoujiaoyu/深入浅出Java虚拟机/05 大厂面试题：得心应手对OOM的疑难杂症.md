1. 从GC Roots向下追溯、搜索，会产生一个叫作Reference Chain的链条。当一个对象不能和任何一个GC Roots产生关系时，就会被清除。垃圾回收就是围绕着GC Roots去做的，同时，它也是很多内存泄漏的根源，因为保留下的对象都是和GC Roots有关系的，其他引用根本没有这样的权力。
2. GC Roots有哪些：GC Roots是一组必须活跃的引用。即，程序中通过直接引用或间接引用，能够访问到的潜在被使用的对象。GC Roots包括：
 - Java线程中，当前所有正在被调用的方法的引用类型参数、局部变量、临时值等。也就是与栈帧相关的各种引用。
 - 所有当前被加载的Java类
 - Java类的引用类型静态变量
 - 运行时常量池的引用类型常量(String或Class类型)
 - JVM内部数据结构的一些引用，比如sun.jvm.hotspot.memory.Universe类
 - 适用于同步的监控对象，比如调用了对象的wait()方法
 - JNI handles，包括global handles和local handles

 GC Roots大体分为三类：
 - 活动线程相关的各种引用
 - 类的静态变量的引用
 - JNI引用

 两个注意点：
 - 活跃的引用，而不是对象，对象是不能作为GC Roots的
 - GC过程**是**找出所有活对象，并把其余空间认定为“无用”；而**不是**找出所有死掉的对象，并回收它们占用的空间。所以即使JVM堆非常大，基于tracing的GC方式，回收速度也会非常快。

3. 引用级别
 - 强引用 Strong references：当内存不足，JVM抛出OOM错误，程序异常终止，这种对象也不会被回收。这种引用属于最普通最强硬的一种存在，只有在和GC Roots断绝关系时，才会被消灭掉。如我们常用的new Object()；
 - 软引用Soft references：软引用用于维护一些可有可无的对象。内存充足时候，软引用不会被回收，只有在内存不足时，系统则会回收软引用对象，如果回收了软引用对象之后，仍然没有足够内存，则会抛出OOM异常。Java中用SoftReference来表示。这种特性非常适合用在缓存技术上，如网页缓存、图片缓存等。
 软引用可以和一个引用队列(ReferenceQueue)联合使用，如果软引用所引用的对象被垃圾回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。案例如下
 ```
        Person person=new Person();
        person.setName("hjp");
        person.setAge(20);
        ReferenceQueue referenceQueue=new ReferenceQueue();
        SoftReference<Person> personSoftReference=new SoftReference<>(person,referenceQueue);
        person=null;
        System.gc();
        System.runFinalization();
        System.out.println(personSoftReference.get());
```
 软引用的回收可以简单的理解为内存不足时进行回收，但是还和参数-XX:SoftRefLRUPolicyMSPerMB配置有关，默认是1秒(1000)，每MB堆空闲空间中SoftReference的存活时间。这个值千万不要设置成0，否则容易引发故障。可以设置比较大的值，但是不要设置成0。
 通过配置参数“-XX:TraceClassLoading -XX:TraceClassUnloading”，查找出频繁加载卸载类的对象。分析出是-XX:SoftRefLRUPolicyMSPerMB设置为0，导致JVM内部类软引用对象频繁创建和移除，元数据区大小波动频繁。
 - 弱引用Weak reference：比软引用生命周期更短，当JVM进行垃圾回收时，无论内存是否充足，都会回收被软引用关联的对象。 Java中用WeakReference来表示，也是常和引用队列(ReferenceQueue)一起使用。应用场景与软引用类似，可以在一些对内存更加敏感的系统里采用。使用方式如下：
 ```
Person person=new Person();
        person.setName("hjp");
        person.setAge(20);
        ReferenceQueue referenceQueue=new ReferenceQueue();
        WeakReference<Person> personWeakReference=new WeakReference<>(person,referenceQueue);
        person=null;
        System.out.println(personWeakReference.get());
        System.gc();
        System.runFinalization();
        Reference poll = referenceQueue.poll();
        Object o = poll.get();//o为空，说明引用队列里只存放引用，原引用对应的对象已被回收
        System.out.println(personWeakReference==poll);//输出true
```
 - 虚引用Phantom Reference：一种形同虚设的引用，现实场景中用的不多。虚引用和引用队列(ReferenceQueue)联合使用。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，随时会被垃圾回收。虚引用的get始终返回null。
 虚引用主要用来跟踪对象被垃圾回收的活动，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收之前，把这个虚引用加入到与之关联的引用队列中。程序如果发现某虚引用已经被加入到引用队列，那么就可以在所引用的对象内存被回收之前采取必要的行动。例子如下
 ```
public class PReference {
    private static boolean finishFlag=false;
    public static void main(String[] args) {
        Person person=new Person();
        person.setName("hjp");
        person.setAge(20);
        ReferenceQueue queue=new ReferenceQueue();
        PhantomReference phantomReference=new PhantomReference(person,queue);
        startMonitoring(queue,phantomReference);
    }
    private static void startMonitoring(ReferenceQueue queue, PhantomReference phantomReference) {
        ExecutorService executorService= Executors.newSingleThreadExecutor();
        executorService.execute(()->{
            while (queue.poll()!=phantomReference){
                if(!finishFlag){
                    break;
                }
            }
            System.out.println("-- ref gc'ed --");
        });
        executorService.shutdown();
    }
}
```
 基于虚引用，更加优雅的实现方式就是Java9以后新加入的Cleaner，用来替代Object类的finalizer方法
4. 典型OOM场景
 ![](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/OOM%E6%83%85%E5%86%B5%E8%A1%A8.png)
 引起OOM的原因：
 - 内存容量太小了，需要扩容，或者需要调整堆的空间
 - 错误的引用方式，发生了内存泄漏。没有及时的切断与GC Roots的关系。如线程池里的线程，在使用的情况下忘记清理ThreadLocal的内容。
 - 接口没有进行范围校验，外部传参超出范围。如数据库查询的每页条数等。
 - 对堆外内存无限制的使用。这种情况一旦发生，更加验证，导致系统内存耗尽。

 典型的内存泄露场景，原因在于对象没有及时的释放自己的引用。