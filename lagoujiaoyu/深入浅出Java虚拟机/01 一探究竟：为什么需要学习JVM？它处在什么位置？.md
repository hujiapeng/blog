1. 有如下Demo.java
```
public class Demo {
    public static void main(String[] args) {
        System.out.println("Hello world!");
    }
}
```
2. 对编译后的Demo.class文件使用```javap -c```命令，得到反编译后代码
```
Compiled from "Demo.java"
public class Demo {
  public Demo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #3                  // String Hello world!
       5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```
3. Java虚拟机采用基于栈的架构，其指令由操作码和操作数组成。这些字节码指令就叫作opcode，如getstatic、ldc、invokevirtual、return等
4. notepad++安装HEX-Editor查看二进制文件Demo.class，可搜索到如下一串数字
```
b2 20 02 12 03 c2 b6 20 04 c2 b1  
```
其中与反编译后的字节码对应关系如下
```
0Xb2    getstatic
0X12    ldc
0Xb6    invokevirtual
0Xb1    return
```
opcode的长度为一个字节(0~255)，即指令集的操作码个数不能超过256条。紧跟在opcode后面的是被操作数。如```b2 20 02```代表```getstatic #2 <java/lang/System.out>```。JVM就是靠解析这些opcode和操作数来完成程序的执行的。
5. JVM首先会加载class文件，将其加载到"元数据"区，运行class文件，就会翻译这些字节码，有两种执行方式。解释执行，将opcode和操作数翻译成机器码；即时编译，即JIT，会在一定条件下将字节码直接编译成机器码后再执行。JVM中的执行引擎会通过"混合模式"执行这些字节码。
6. JVM程序运行都是在栈上完成的，当运行到方法时，就会给它分配一个栈帧，当退出方法时，就会弹出相应的栈帧。