# 判断Java对象是否存活

## 引用计数法

引用计数法是最被广为猿知的判断对象存活的方法：在对象中添加一个引用计数器，当一个引用时计数器加一，引用失效时计数器减一，当计数器为零时对象不被使用。特点：占用一些额外空间进行计数，当原理简单、效率高。但Java的主流虚拟机都没有选用引用计数法来管理内存，主要是引用计数法很难解决对象之间循环依赖等问题。

## 可达性分析算法

当前主流的商用程序语言都基于可达性分析（Reachability Analysis）算法来判定对象是否存活。可达性分析算法是通过一些列称为`GC Roots`的根对象作为起始节点。从这些根节点开始根据引用关系向下搜索，搜索走过的路径称为引用链（Reference Chain），如果没有一个GC Roots能够到达某个对象，那个这个对象就不再被使用。如下图所示：

![image-20210512112936787](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210512162552.png)

### GC Roots对象

在Java中以下几种情况可作为GC Roots：

* 在虚拟机栈（栈帧中的本地变量表）中引用的对象
* 在方法区中类静态属性引用的对象。
* 在方法区中常量引用的对象，比如字符串常量池里的引用。
* 子本地发方法栈中JNI（Native方法）引用的对象。
* Java虚拟机内部的引用，如基本类型的Class对象，常驻的异常对象（NullPointException、OOM），还有系统类加载器。
* 所有被同步锁（synchronized）持有的对象。
* 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等。

除上述的这些GC Roots以外，根据用户所选用的垃圾回收器集当前回收的内存区域不同，还会有其他对象“临时性”的加入。比如分代收集和局部回收（Partial GC），如果只针对Java堆中某一块区域发起垃圾收集时（如最典型的只针对新生代的垃圾收集），必须考虑到内存区域是虚拟机自己的实现细节（在用户视角里任何内存区域都是不可见的），更不是孤立封闭的，所以某个区域里的对象完全有可能被位于堆中其他区域的对象所引用，这时候就需要将这些关联区域的对象也一并加入GC Roots集合中去，才能保证可达性分析的正确性。

## 引用分类

在JDK2之后，Java中的引用分为强引用（Strongly Reference）、软引用（Soft Reference）、弱引用（Weak Reference）和虚引用（Phantom Reference）4种，这4中引用强度一次减弱。

* 强引用指在代码中使用“=”赋值引用，无论在任何情况下，之哟啊强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象。
* 软引用是用来描述有用但非必须的对象，软引用的对象在系统发生内存溢出前，才会被列进回收范围中进行第二次回收，如果内存空间还是不足才会抛出内存溢出异常。JDK中提供了SoftReference类来实现软引用。
* 弱引用也是来描述非必须的对象，但是强度比软引用更弱，并只能生存到洗一次垃圾收集发生为止。不管内存是否足够，垃圾收集器都会回收掉弱引用的对象。JDK中提供了WeakReference类来实现弱引用。
* 虚引用也被称为“幽灵引用”，为对象设置虚引用的唯一目的只是能在这个对象被收集器回收时收到一个系统通知，JDK提供了PhantomReference类来实现虚引用。没见过在哪用😂。

## finalize方法

一个对象被标记为不可达对象后，并不是直接别回收，要宣告一个对象死亡，至少要经历两次标记：如果被可达性分析判断为不可达，将会被第一次标记，随后筛选随心是否有必要执行finalize()方法。如果对象没有Override finalize()方法，或者finalize()方法已经被虚拟机调用过，都会被判定为：没有必要执行。

当对象被判定为有必要执行finalize()方法后，将会被放置到名为`F-Queue`的队列中等待Finalizer线程执行finalize()方法，该线程由虚拟机创建且优先级很低。finalize()方法的执行是指会触发该方法开始运行，但不能保证等待方法结束，这样做是为了防止某个finalize()方法执行缓慢或者死循环导致F-Queue队列的奇特对象永久处于等待状态，进而导致垃圾收集系统崩溃。

finalize()方法可以让对象实现一次自我救赎——只需要月引用链上的任何一个对象建立关联即可。下面的代码演示了对象一次自我救赎的过程。

```java
public class FinalizeEscapeGC {
    public static FinalizeEscapeGC SAVE_HOOK = null;

    private void isAlive() {
        System.out.println("i am still alive :)");
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("finalize method is being executed");
        //自我拯救
        SAVE_HOOK = this;
    }

    public static void main(String[] args) throws Exception {
        SAVE_HOOK = new FinalizeEscapeGC();
        while (true) {
            SAVE_HOOK = null;
            System.gc();
            //等待Finalizer线程执行
            Thread.sleep(1000);
            if (SAVE_HOOK != null) {
                SAVE_HOOK.isAlive();
            } else {
                System.out.println("i am dead :(");
                break;
            }
        }
    }
}
```

执行结果

```
finalize method is being executed
i am still alive :)
i am dead :(
```

从执行结果中可以看到，SAVE_HOOK对象的finalize()方法确实被垃圾收集器触发过，并且在被收集前成功逃脱了，但是第二次却失败了。因为对象的finalize()方法指挥被系统执行一次。

## 回收方法区

《Java虚拟机规范》中提到过可以不要求虚拟机在方法区中实现垃圾收集，事实上也确实有未实现或未能完整实现方法区类型卸载的收集器存在（如JDK 11时期的ZGC收集器就不支持类卸载），方法区垃圾收集的“性价比”通常也是比较低的。

方法区的垃圾收集主要回收两部分内容：**废弃的常量**和**不再使用的类型**。废弃常量的回收与回收Java堆对象非常类似。虚拟机判断常量在其他地方没有引用则会在内存回收时进行清理。

常量的废弃判断相对简单，而一个类型“不再使用”的判断就比较苛刻了，需要同事满足以下三个条件：

* 该类的所有实例都已被回收。
* 加载该类的类加载器已经被回收，这个条件除非是经过精心设计的可替换类加载器的场景，如OSGi、JSP的重加载等，否则通常是很难达成的。
* 该类对象的Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

Java虚拟机被允许对满足上述三个条件的无用类进行回收，这里说的仅仅是“被允许”，而并不是和对象一样，没有引用了就必然会回收。关于是否要对类型进行回收，HotSpot虚拟机提供了-Xnoclassgc参数进行控制，还可以使用-verbose:class以及-XX:+TraceClass-Loading、-XX:+TraceClassUnLoading查看类加载和卸载信息，其中-verbose:class和-XX:+TraceClassLoading可以在Product版的虚拟机中使用，-XX:+TraceClassUnLoading参数需要FastDebug版的虚拟机支持。

> 注：本文为《深入理解Java虚拟机：JVM高级特性与最佳实战》读书笔记