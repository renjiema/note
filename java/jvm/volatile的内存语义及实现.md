# volatile的内存语义及实现

## volatile的特性

理解volatile特性的一个好方法是把对volatile变量的单个读/写，看成是使用同一个锁对这些单个读/写操作做了同步，示例代码如下：

```java
// 使用volatile声明64位的long型变量
volatile long vl = 0L;                  
public void set(long l) {
    // 单个volatile变量的写
    vl = l;                            
}
public void getAndIncrement () {
    // 复合（多个）volatile变量的读/写
    vl++;                               
}
public long get() {
    // 单个volatile变量的读
    return vl;
}
```

上面的代码在语义上等价于下面的代码：

```java
// 使用volatile声明64位的long型变量
volatile long vl = 0L;                  
public synchronized void set(long l) {  // 对单个的普通变量的写用同一个锁同步
    vl = l;
}
public void getAndIncrement() {        // 普通方法调用
    long temp = get();                  // 调用已同步的读方法
    temp += 1L;                         // 普通写操作
    set(temp);                          // 调用已同步的写方法
}
public synchronized long get() {        // 对单个的普通变量的读用同一个锁同步
    return vl;
}
```

一个volatile变量的单个读/写操作，与一个普通变量的读/写操作都是使用同一个锁来同步的执行效果相同。

因为锁的happens-before规则保证释放锁和获取锁的两个线程之间的内存可见性，因此对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。

volatile变量的特性：

* **可见性**。对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。
* **原子性**：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不具有原子性。

volatile对线程的内存可见性的影响比volatile自身的特性更为重要，从内存语义的角度来说，volatile的写-读与锁的释放-获取有相同的内存效果：volatile写和锁的释放有相同的内存语义；volatile读与锁的获取有相同的内存语义。

分析以下使用volatile变量的代码：

```java
int a = 0;
volatile boolean flag = false;
public void writer() {
    a = 1;          // 1
    flag = true;    // 2
}
public void reader() {
    if (flag) {     // 3
        int i = a;  // 4
    }
}
```

假设线程A执行writer()方法之后，线程B执行reader()方法。根据happens-before规则，这个过程建立的happens-before关系如下：

1. 根据程序次序规则，1 happens-before 2、3 happens-before 4。
2. 根据volatile规则，2 happens-before 3。
3. 根据happens-before的传递性规则，1 happens-before 4。

执行流程如下图

<img src="https://raw.githubusercontent.com/renjiema/images/main/blogs/20210714175326.png" alt="image-20210714160000083"  />

每一个箭头链接的两个节点，代表了一个happens-before关系。黑色箭头表示程序顺序规则；橙色箭头表示volatile规则；蓝色箭头表示组合这些规则后提供的happens-before保证。因此：**A线程在写volatile变量之前所有可见的共享变量，在B线程读同一个volatile变量后，将立即变得对B线程可见**。

## volatile读写的内存语义

volatile写的内存语义：当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。

volatile读的内存语义：当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

## volatile内存语义的实现

重排序分为编译器重排序和处理器重排序。为了实现volatile内存语义，JMM会分别限制这两种类型的重排序类型。下表是JMM针对编译器制定的volatile重排序规则表。

![image-20210714180504670](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210714180508.png)

表中NO表示禁止重排序。从上表可以看出

* 当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序。这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后。
* 当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。
* 当第一个操作是volatile写，第二个操作是volatile读时，不能重排序。

为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能。为此，JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略。

* 在每个volatile写操作的前插入一个StoreStore屏障：禁止上面的普通写和volatile写重排序。
* 在每个volatile写操作的后插入一个StoreLoad屏障：防止volatile写与下面可能的volatile读/写重排序。
* 在每个volatile读操作的后插入一个LoadLoad屏障：禁止下面所有的普通读和volatile读重排序。
* 在每个volatile读操作的后插入一个LoadStore屏障：禁止下面所有的普通写和volatile读重排序。

上述内存屏障插入策略非常保守，但它可以保证在任意处理器平台，任意的程序中都能得到正确的volatile内存语义。

比较有意思的是，**volatile写后面的StoreLoad屏障**。此屏障的作用是避免volatile写与后面可能有的volatile读/写操作重排序。因为编译器常常无法准确判断在一个volatile写的后面是否需要插入一个StoreLoad屏障（比如，一个volatile写之后方法立即return）。为了保证能正确实现volatile的内存语义，JMM在采取了保守策略：在每个volatile写的后面，或者在每个volatile读的前面插入一个StoreLoad屏障。因为volatile写-读内存语义的常见使用模式是：一个写线程写volatile变量，多个读线程读同一个volatile变量。当读线程的数量大大超过写线程时，选择在volatile写之后插入StoreLoad屏障将带来可观的执行效率的提升。从这里可以看到JMM在实现上的一个特点：**首先确保正确性，然后再去追求执行效率**。

上述volatile写和volatile读的内存屏障插入策略非常保守。在实际执行时，只要不改变volatile写-读的内存语义，编译器可以根据具体情况省略不必要的屏障。

```java
int a;
volatile int v1 = 1;
volatile int v2 = 2;
void readAndWrite() {
    int i = v1;         // 第一个volatile读
    int j = v2;         // 第二个volatile读
    a = i + j;          // 普通写
    v1 = i + 1;         // 第一个volatile写
    v2 = j * 2;         // 第二个 volatile写
}
```

针对readAndWrite()方法，编译器在生成字节码时可以做如下的优化。

![image-20210715103740948](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210715103753.png)

最后的StoreLoad屏障不能省略。因为第二个volatile写之后，方法立即返回。此时编译器可能无法准确断定后面是否会有volatile读或写，为了安全起见，编译器通常会在这里插入一个StoreLoad屏障。

由于不同的处理器有不同“松紧度”的处理器内存模型，内存屏障的插入还可以根据具体的处理器内存模型继续优化。以X86处理器为例，保守策略下的volatile读和写，在X86处理器平台可以优化成如下图所示。

![image-20210715104225919](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210715104230.png)

X86处理器仅会对写-读操作做重排序不会对读-读、读-写和写-写操作做重排序，因此在X86处理器中会省略掉这3种操作类型对应的内存屏障。JMM仅需在volatile写后面插入一个StoreLoad屏障即可正确实现volatile写-读的内存语义。因此在X86处理器中，volatile写的开销比volatile读的开销会大很多。

由于volatile仅仅保证对单个volatile变量的读/写具有原子性，而锁的互斥执行的特性可以确保对整个临界区代码的执行具有原子性。在功能上，锁比volatile更强大；在可伸缩性和执行性能上，volatile更有优势。

