# OOM异常

本文会通过演示《Java虚拟机规范》中描述的各个运行时区域存储的内容，记录各个区域内存溢出的异常提示信息，了解怎样的代码可能会导致内存移除及出现OOM异常后该如何处理。

## Java堆溢出

Java堆用于存储对象实例，因此只要不断创建对象，并保证GC Roots和对象之间有可达路径来避免垃圾回收机制回收这些对象，总会达到最大堆的容量限制产生OOM异常。代码清单如下：

```java
/**
 * VM Args:-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {
    static class OOMObject {
    }

    public static void main(String[] args) {
        final ArrayList<OOMObject> list = new ArrayList<>();
        while (true){
            list.add(new OOMObject());
        }
    }
}
```

参数`-XX:+HeapDumpOnOutOfMemoryError`可以在OOM异常时Dump当前的内存堆快照以便进行问题分析。

运行结果：

```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid24388.hprof ...
Heap dump file created [28108263 bytes in 0.086 secs]
```

Java堆内存的OOM异常时实际应用中最常见的内存溢出异常。一般会在异常堆栈信息`java.lang.OutOfMemoryError`后会进一步提示`Java heap space`。

解决Java堆的OOM异常，常规的处理方法时通过内存快照缝隙工具对Dump的文件进行分析。首先分清楚是内存泄漏（Memory Leak）还是内存溢出（Memory Overflow），并确认导致OOM的对象是否必要。

如果是内存泄漏，可进一步通过工具查看泄漏对象到GC Roots的引用链，找到泄漏对象是通过怎样的引用路径、与哪些GC Roots相关联，才导致垃圾收集器无法回收它们，根据泄漏对象的类型信息以及它到GC Roots引用链的信息，一般可以比较准确地定位到这些对象创建的位置，进而找出产生内存泄漏的代码的具体位置。

如果是内存溢出那就应当检查Java虚拟机的堆参数（-Xmx与-Xms）设置，与机器的内存对比，看看是否还有向上调整的空间。再从代码上检查是否存在某些对象生命周期过长、持有状态时间过长、存储结构设计不合理等情况，尽量减少程序运行期的内存消耗。

## 虚拟机栈和本地方法栈溢出

HotSpot虚拟机中不区分虚拟机栈和本地方法栈，因此栈容量只能通过-Xss参数类设定。并且HotSpot虚拟机是不支持扩展，所以只有在创建线程申请内存时就因内存不足而出现OutOfMemoryError异常，在线程运行时只会因为栈容量无法容纳新的栈帧而导致StackOverflowError异常。

首先减少栈内存容量来使栈溢出

```Java
/**
 * VM Args:-Xss128k
 */
public class JavaVMStackSOF {
    private int stackDepth = 1;

    public void stackLeak() {
        stackDepth++;
        stackLeak();
    }

    public static void main(String[] args) {
        final JavaVMStackSOF sof = new JavaVMStackSOF();
        try {
            sof.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack depth:" + sof.stackDepth);
            throw e;
        }
    }
}
```

运行结果：

```
stack depth:1002
Exception in thread "main" java.lang.StackOverflowError
	at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:12)
	at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:13)
	...
```

多次运行stack depth值不一样，后面再看看啥问题。

如果测试不限与单线程，通过不断建立线程的方式，在HotSpot上也可以产生内存溢出异常，这种内存溢出异常和栈空间不存在任何直接的关系，主要取决与操作系统本身的内存使用状态。而且给每个线程的栈分配的内存越大，越容易产线内存溢出异常。因为操作系统分配给每个进程的内存是有限制的，每个线程分配的内存越大，可建立的线程数越少，新建线程是越容易内存耗尽。创建线程时内存溢出的报错信息：

```java
Exception in thread "main" java.lang.OutOfMemoryError: unable to create native thread
```

多线程导致的内存溢出，在不能减少线程数量或者更换64位虚拟机的情况下，就只能通过减少最大堆和减少栈容量来换取更多的线程

## 方法区和运行时常量池溢出

运行时常量池时方法区的一部分，不过HotSpot从jdk7开始逐步去除`永久代`，并在jdk8中完全使用元空间代替永久代，接下来通过代码测试以下不同实现的方法区在实际运行时的区别。

分别以JDK6和JDK7+执行以下代码，将会得到不同的结果，因为运行时常量池在JDK7以后已经被移至Java堆之中。其中String.intern()方法是一个本地方法，作用时将不再字符串常量池终端字符串添加到常量池中，已存在则返回引用。

```java
/**
 * JDK6 VM Args:-XX:PermSize=6M -XX:MaxPermSize=6M
 * JDK7+ VM Args:-Xmx20m
 */
public class RuntimeConstantPoolOOM {
    public static void main(String[] args) {
        final HashSet<String> set = new HashSet<>();
        int i = 0;
        while (true){
            set.add(String.valueOf(i++).intern());
        }
    }
}
```

JDK6执行结果

```java
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
    at java.lang.String.intern(Native Method)
```

JDK7+执行结果

```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.HashMap.resize(HashMap.java:580)
	at java.util.HashMap.addEntry(HashMap.java:879)
    
//或者
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.lang.Integer.toString(Integer.java:333)
	at java.lang.String.valueOf(String.java:2954)
    
//或者
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.HashMap.createEntry(HashMap.java:897)
	at java.util.HashMap.addEntry(HashMap.java:884)
```

具体取决与在哪里分配内存时产生溢出。

方法区的主要存储信息为：类名、访问修饰符、常量池、字段描述、方法描述等。对于这些信息的测试，可以通过尝试大量的类使方法区溢出。代码如下：

```java
/**
 * VM Args:-XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class JavaMethodAreaOOM {
    static class OOMObject {

    }

    public static void main(String[] args) {
        try {
            while (true) {
                final Enhancer enhancer = new Enhancer();
                enhancer.setSuperclass(OOMObject.class);
                enhancer.setUseCache(false);
                enhancer.setCallback(new MethodInterceptor() {
                    @Override
                    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                        return methodProxy.invokeSuper(o, objects);
                    }
                });
                enhancer.create();
            }
        } catch (Throwable e) {
            e.printStackTrace();
        }
    }
}
```

JDK7执行结果

```java
Caused by: java.lang.OutOfMemoryError: PermGen space
    at java.lang.ClassLoader.defineClass1(Native Method)
```

以上代码通过CGLib动态创建大量动态类使方法区内存溢出，该异常可能会出现在实际的应用中，当前的主流框架：spring、mybatis等都是通过CGLib这类字节码技术对类进行代理，代理的类越多则越可能导致方法区的内存溢出。除此之外大量的jsp或基于OSGi的应用都可能导致方法区OOM。

在JDK8以后使用元空间替代永久代，方法区被移至堆空间中因此上面所述方法区溢出异常的情况比较难发生了。

## 本地直接内存溢出

直接内存的容量大小可以通过`-XX:MaxDirectMemorySize`参数指定，默认和Java堆最大值一致，但是这个参数并不能限制Unsafe::allocateMemory方法申请本地直接内存，因此下面的代码不会抛出OOM异常，直到本地主机内存不足。

```java
/**
 *VM Args：-Xmx20M -XX:MaxDirectMemorySize=10M
 */
private static final int _1MB = 1024 * 1024;
public static void main(String[] args) throws Exception {
    final Field field = Unsafe.class.getDeclaredFields()[0];
    field.setAccessible(true);
    final Unsafe unsafe = (Unsafe) field.get(null);
    while (true){
        unsafe.allocateMemory(_1MB);
    }
}
```

下面的代码会抛出OOM异常，主要原因是在调用Unsafe::allocateMemory方法前会判`MaxDirectMemorySize`大小主动抛出异常。

```java
/**
 *VM Args：-Xmx20M -XX:MaxDirectMemorySize=10M
 */
public static void main(String[] args) {
    ByteBuffer.allocateDirect(30 * 1024 * 1024);
}
```

执行结果

```java
Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
	at java.nio.Bits.reserveMemory(Bits.java:658)
```

如果直接内存导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看尽啊有什么明显的异常情况。

> 注：本文为《深入理解Java虚拟机：JVM高级特性与最佳实战》读书笔记