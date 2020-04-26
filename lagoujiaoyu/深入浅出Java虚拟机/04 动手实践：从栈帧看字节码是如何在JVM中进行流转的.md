1. 工具介绍
 - javap是JDK自带的反解析工具，将.class字节码文件解析成可读的文件格式。如下：
 ```
javap -p -v Demo
Classfile /F:/JavaCodeProject/demo/target/classes/Demo.class
  Last modified 2020-3-22; size 516 bytes
  MD5 checksum ded78c5773df25791ab978f345065d23
  Compiled from "Demo.java"
public class Demo
  minor version: 0
······
 public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello World!
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 10: 0
        line 11: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
```
 如果不同IDE生成的同一个class文件不同，可以添加JVM参数，如没有LineNumberTable，则可以使用javac -g:lines进行编译，强制生成；如没有LocalVariableTable，则可以使用javac -g:vars进行编译，强制生成；javac -g生成所有的debug信息。
 - jclasslib：一款查看字节码的图形化工具。可以在IDEA中查找插件进行安装
2. 类加载和对象创建的时机：类的初始化发生在类加载阶段，而对象创建方式有四种
 - 使用CLass的newInstance方法
 - 使用Constructor类的newInstance方法
 - 反序列化
 - 使用Object的clone方法

 其中反序列化和Object.clone方法没有调用到构造函数。当虚拟机遇到一条new指令时，首先检查这个指令的参数能否在常量池中定位一个符号引用。然后检查这个符号引用的类字节码是否加载、解析和初始化。如果没有，将执行对应的类加载过程。
3. 查看字节码
 - 命令行查看字节码
    - 先使用javac -g:lines -g:vars Java源码，编译后的class文件会强制生成LineNumber和LocalVariableTable。如果使用的IDEA，也可以在VM options里添加如下命令
    - 然后使用javap -p -v class字节码，查看字节码，不仅会输出行号、本地变量表信息、反编译汇编代码，还会输出当前类用到的常量池等信息。

 - 可视化查看字节码。如果IDEA中安装了jclasslib插件，可以通过菜单栏View->Show Bytecode With Jclasslib查看java或class文件。LineNumberTable中Start PC和Line Number表示字节码行号(字节码偏移量)和源码行号，有了这些信息，在debug时，就能获取到发生异常的源代码行号。

4. 函数执行过程，案例代码如下，以test函数为例
 ```
public class B {
    private int a = 1234;
    static long C = 1111;
    public long test(long num) {
        long ret = this.a + num + C;
        return ret;
    }
}
```
 对应的字节码如下
 ```
  public long test(long);
    descriptor: (J)J
    flags: ACC_PUBLIC
    Code:
      stack=4, locals=5, args_size=2
         0: aload_0
         1: getfield      #2                  // Field a:I
         4: i2l
         5: lload_1
         6: ladd
         7: getstatic     #3                  // Field C:J
        10: ladd
        11: lstore_3
        12: lload_3
        13: lreturn
      LineNumberTable:
        line 5: 0
        line 6: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      14     0  this   LB;
            0      14     1   num   J
           12       2     3   ret   J
```
 - stack=4表示test方法的最大操作数栈深度为4.JVM运行时会根据这个数值来分配栈帧中操作栈的深度
 - locals=5，locals变量存储了局部变量的存储空间，它的单位是Slot(槽)，可以被重用。其中存放内容如下：
    - this
    - 方法参数
    - 异常处理器的参数
    - 方法体中定义的局部变量

 - args_size指的是方法参数个数，每个方法都有一个隐藏参数this，所以这里的数字是2

5. 字节码执行过程：main线程会拥有两个主要的运行时区域：Java虚拟机栈和程序计数器。其中虚拟机栈中的每一项内容叫作栈帧，包括四项内容：局部变量报表、操作数栈、动态链接和完成出口。
 - 0:aload_0
   把第1个引用型局部变量推到操作数栈，即此处把this装载到了操作数栈中。
   注意，如果是static方法，aload_0表示对方法的第一个参数的操作。
![test方法字节码执行过程1](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/test%E6%96%B9%E6%B3%95%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B1.png)
 - 1:getfield   #2
   将栈顶的指定的对象的第2个实例域(Field)的值压入栈顶。#2就是指成员变量a。
![test方法字节码执行过程2](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/test%E6%96%B9%E6%B3%95%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B2.png)
 - i2l
   将栈顶int类型的数据转化为long类型(隐式转换)
 - lload_1
   将第一个局部变量入栈，即参数num。l表示long
![test方法字节码执行过程3](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/test%E6%96%B9%E6%B3%95%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B3.png)
 - ladd
   把栈顶两个long型数值出栈后相加，并将结果入栈
![test方法字节码执行过程4](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/test%E6%96%B9%E6%B3%95%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B4.png)
 - getstatic    #3
   根据偏移量获取静态属性的值，并把这个值push到操作数栈上
![test方法字节码执行过程5](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/test%E6%96%B9%E6%B3%95%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B5.png)
 - ladd
   再次执行ladd
![test方法字节码执行过程6](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/test%E6%96%B9%E6%B3%95%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B6.png)
 - lstore_3
   把栈顶long型数值存入第4个局部变量，即Slot索引为3的ret
![test方法字节码执行过程7](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/test%E6%96%B9%E6%B3%95%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B7.png)
 - lload_3
   与lstore相反，lstore是将值存入变量；lload_3就是把这个变量ret，压入虚拟机栈中
![test方法字节码执行过程8](https://raw.githubusercontent.com/hujiapeng/imgs/master/lagou/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJava%E8%99%9A%E6%8B%9F%E6%9C%BA/test%E6%96%B9%E6%B3%95%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B8.png)
 - lreturn
   从当前方法返回long
6. 注意上面函数执行过程中，lstore_3和lload_3的作用是将值先存入变量表，然后取出又放入栈，最后出栈返回数据。看似是重复操作，这时因为JVM并不知道变量ret后面是否会用到，但也没必要为了减少JVM的两行命令可以省略掉那个变量，栈的操作复杂的为O(1)。
   