# 低延迟垃圾收集器

垃圾收集器的三项最重要的指标：内存占用（Footprint）、吞吐量（Throughput）和延迟（Latency），三者构成了一个“不可能三角”。三者总体的表现会随着技术进步而越好，但要在三个方面同时具有卓越表现的收集器是极其困难的。

相比于内存占用和吞吐量而言，延迟的重要性日益凸显。因为随着计算机硬件的发展，性能的提升，内存的成本降低，吞吐量提高，但对延迟反而带来负面的效果，因为虚拟机要回收完整的1TB的堆内存，毫无疑问要比回收1GB的堆内存耗费更多时间。下图是一些常用的垃圾收集器的停顿状态：

![image-20210517141520014](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210517141527.png)

上图中浅色阶段表示必须挂起用户线程，深色表示收集器线程与用户线程是并发工作的。在CMS和G1之前的全部收集器，其工作的所有步骤都会产生“Stop The World”式的停顿；CMS和G1分别使用增量更新和原始快照技术，实现了标记阶段的并发。CMS使用标记-清除算法，避免了整理阶段收集器带来的停顿，但是避免不了空间碎片的产生，随着空间碎片不断增加最终依旧会**Stop The World**。G1虽然可以按更小的粒度进行回收，从而抑制整理阶段出现时间过长的停顿，但还是要暂停的。

最后的两款收集器，Shenandoah和ZGC，几乎整个工作过程全部都是并发的，只有初始标记[^1]、最终标记这些阶段有短暂的停顿，这部分停顿的时间基本上是固定的，与堆的容量、堆中对象的数量没有正比例关系。它们都可以在任意可管理的（譬如现在ZGC只能管理4TB以内的堆）堆容量下，实现垃圾收集的停顿都不超过十毫秒这种听起来是天方夜谭的目标。这两款目前仍处于实验状态[^2]的收集器，被官方命名为“低延迟垃圾收集器”（Low-Latency Garbage Collector或者Low-Pause-Time Garbage Collector）。

## Shenandoah收集器

Shenandoah是第一款不由Oracle（包括以前的Sun）公司的虚拟机团队所领导开发的HotSpot垃圾收集器。最初Shenandoah是由RedHat公司独立发展的新型收集器项目，在2014年RedHat把Shenandoah贡献给了OpenJDK，并推动它成为OpenJDK 12的正式特性之一，也就是后来的JEP 189。

比起Oracle的ZGC，Shenandoah反而更像是G1的下一代继承者，它们有着相似的堆内存布局，在初始标记、并发标记等许多阶段的处理思路上都高度一致，甚至还直接共享了一部分实现代码，这使得部分对G1的打磨改进和Bug修改会同时反映在Shenandoah之上，而由于Shenandoah加入所带来的一些新特性，也有部分会出现在G1收集器中，比如在并发失败后作为**逃生门**的Full GC，G1就是由于合并了Shenandoah的代码才获得多线程Full GC的支持。

虽然Shenandoah也是使用基于Region的堆内存布局，同样有着用于存放大对象的Humongous Region，默认的回收策略也同样是优先处理回收价值最大的Region……但在管理堆内存方面，它与G1至少有三个明显的不同之处：

* 最重要的当然是支持并发的整理算法，G1的回收阶段是可以多线程并行的，但却不能与用户线程并发。
* 其次，Shenandoah（目前）是默认不使用分代收集的，换言之，不会有专门的新生代Region或者老年代Region的存在，没有实现分代，并不是说分代对Shenandoah没有价值，这更多是出于性价比的权衡，基于工作量上的考虑而将其放到优先级较低的位置上。
* 最后，Shenandoah摒弃了在G1中耗费大量内存和计算资源去维护的记忆集，改用**连接矩阵**（Connection Matrix）的全局数据结构来记录跨Region的引用关系，降低了处理跨代指针时的记忆集维护消耗，也降低了伪共享问题的发生概率。

连接矩阵可以简单理解为一张二维表格，如果Region N有对象指向Region M，就在表格的N行M列中打上一个标记，如下图所示，如果Region 5中的对象Baz引用了Region 3的Foo，Foo又引用了Region 1的Bar，那连接矩阵中的5行3列、3行1列就应该被打上标记。在回收时通过连接矩阵就可得出哪些Region之间产生了跨代引用。

![image-20210517151427040](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210517151427.png)

Shenandoah收集器的工作过程大致可以划分为以下九个阶段（此处以Shenandoah在2016年发表的原始论文进行介绍。在最新版本的Shenandoah 2.0中，进一步强化了“部分收集”的特性，初始标记之前还有Initial Partial、Concurrent Partial和Final Partial阶段，它们可以不太严谨地理解为对应于以前分代收集中的Minor GC的工作）：

* 初始标记（Initial Marking）：与G1一样，首先标记与GC Roots直接关联的对象，**需要STW**，但停顿时间与堆大小无关，与GC Roots数量相关。
* 并发标记（Concurrent Marking）：与G1一样，遍历对象图，标记出全部可达的对象。
* 最终标记阶段（Final Marking）：与G1一样，处理剩余SATB（Snapshot At The Beginning）扫描，并在这个阶段统计出回收价值最高的Region，将这些Region构成一组回收集（Collection Set）。最终标记阶段也会有一小段**STW**。
* 并发清理（Concurrent Cleanup）：清理整个区域内没有一个存活对象的Region（Immediate Garbage Region）。
* 并发回收（Concurrent Evacuation）：并发回收阶段是Shenandoah与之前HotSpot中其他收集器的核心差异。在此阶段，Shenandoah要将回收集中的存活对象复制到其他未被使用的Region中。并发复制对象的困难的在于移动对象的同时，用户线程仍可能不停对被移动对象进行读写访问，移动对象是一次性的行为，但移动后整个内存中所有指向该对象的引用都还是旧对象的地址。对于并发回收阶段遇到的这些困难，Shenandoah将会通过读屏障和被称为[Brooks Pointers](#Brooks Pointer)的转发指针来解决。并发回收阶段运行的时间长短取决于回收集的大小。
* 初始引用更新（Initial Update Reference）：并发回收阶段复制对象结束后，需要把堆中所有旧对象的引用修正为复制后的新地址，这个操作为引用更新。 初始化阶段只是为了建立一个线程集合点，确保所有并发回收阶段中进行的收集器线程都已完成分配给它们的对象移动任务而已。初始引用更新时间很短，会产生一个非常短暂的**STW**。
* 并发引用更新（Concurrent Update Reference）：真正开始进行引用更新操作，时间长短取决于内存中涉及的引用数量的多少。只需要按照内存物理地址的顺序，线性地搜索出引用类型，把旧值改为新值即可。
* 最终引用更新（Final Update Reference）：解决了堆中的引用更新后，还要修正存在于GC Roots中的引用。这个阶段是Shenandoah的最后一次**STW**，停顿时间只与GC Roots的数量相关。
* 并发清理（Concurrent Cleanup）：经过并发回收和引用更新之后，整个回收集中所有的Region已再无存活对象（Immediate Garbage Regions），再调用一次并发清理过程来回收这些Region的内存空间。

以上Shenandoah收集器的九个阶段中，最重要的三个并发阶段（并发标记、并发回收、并发引用更新），下图为Shenandoah收集器的运作过程：

![shenandoah-gc-cycle](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210517152026.png)

黄色的区域代表的是被选入回收集的Region，绿色部分就代表还存活的对象，蓝色就是用户线程可以用来分配对象的内存Region。形象的展示了Shenandoah三个并发阶段的工作过程，还形象地表示出并发标记阶段如何找出回收对象确定回收集，并发回收阶段如何移动回收集中的存活对象，并发引用更新阶段如何将指向回收集中存活对象的所有引用全部修正。

### Brooks Pointer

"Brooks”是一个人的名字。1984年，Rodney A.Brooks在论文《Trading Data Space for Reduced Time and Code Space in Real-Time Garbage Collection on Stock Hardware》中提出了使用转发指针（Forwarding Pointer，也常被称为Indirection Pointer）来实现对象移动与用户程序并发的一种解决方案。此前，要做类似的并发操作，通常是在被移动对象原有的内存上设置保护陷阱（Memory Protection Trap），一旦用户程序访问到旧对象的内存空间就会产生自陷中断，进入预设好的异常处理器中，再由其中的代码逻辑把访问转发到新对象上。虽然确实能够实现对象移动与用户线程并发，但是如果没有操作系统层面的直接支持，这种方案将导致用户态频繁切换到核心态，代价是非常大的，不能频繁使用。

Brooks提出的方案不需要内存保护陷阱，而是在原对象布局结构上增加一个新的引用字段，在正常不处于并发移动的情况下，该引用指向对象自己。如下如所示：

<img src="https://raw.githubusercontent.com/renjiema/images/main/blogs/20210517152928.png" alt="3e8ec24160fdc7a26e646b229d0d1c7c" style="zoom:50%;" />

转发指针的缺点很明显：每次对象访问会带来一次额外的转向开销，尽管这个开销被优化到只有一行汇编指令的程度——`mov r13,QWORD PTR [r12+r14*8-0x8]`。不过比起内存包含陷阱已经好了很多。

加入转发指针后，只需要修改转发指针的引用位置，使其指向新对象，便可将所有对该对象的访问转发到新的副本上。这样只要旧对象的内存仍然存在，虚拟机内存中所有通过旧引用地址访问的代码便可被自动转发到新对象上继续工作，如下图所示：



<img src="https://raw.githubusercontent.com/renjiema/images/main/blogs/20210517153747.png" alt="image-20210517153747839" style="zoom:50%;" />

Brooks转发指针在设计上决定了它是必然会出现多线程竞争问题的，如果只是并发读取，那无论读到旧对象还是新对象上的字段，返回的结果都应该是一样的，这个场景还可以有一些“偷懒”的处理余地；但如果发生了并发写入，就一定必须保证写操作只能发生在新复制的对象上，而不是写入旧对象的内存中。不妨设想以下三件事情并发进行时的场景：

1. 收集器线程复制了新的对象副本；
2. 用户线程更新对象的某个字段；
3. 收集器线程更新转发指针的引用值为新副本地址。

事件2在事件1、事件3之间发生的话，将导致用户线程对对象的变更发生在旧对象上，所以必须针对转发指针的访问操作采取同步措施。实际上Shenandoah收集器是通过**CAS**（Compare And Swap）操作来保证并发时对象的访问正确性的。

转发指针还必须注意执行频率的问题，尽管通过对象头上的Brooks Pointer来保证并发时原对象与复制对象的访问一致性，这件事情只从原理上看是不复杂的，但是“对象访问”这四个字的分量是非常重的，对于一门面向对象的编程语言来说，对象的读取、写入，对象的比较，为对象哈希值计算，用对象加锁等，这些操作都属于对象访问的范畴，要覆盖全部对象访问操作，Shenandoah不得不同时设置读、写屏障去拦截。

在经典收集器中，写屏障被用来维护卡表或实现并发标记，在Shenandoah中除了之前的功能，为了实现Brooks Pointer，Shenandoah在读、写屏障中都加入了额外的转发处理，尤其是使用读屏障的代价比写屏障更大（因为读写的读取频率比写入的频率高的多）。所以计划在JDK 13中将Shenandoah的内存屏障模型改进为基于**引用访问屏障**（Load Reference Barrier）的实现，所谓“引用访问屏障”是指内存屏障只拦截对象中数据类型为引用类型的读写操作，这能省去到了对原生类型、对象比较、对象加锁场景中设置内存屏障的消耗。

Shenandoah收集器作为第一款由非Oracle开发的垃圾收集器，一开始就预计到了缺乏Oracle公司那样富有经验的研发团队可能会遇到很多困难。所以Shenandoah采取了“小步快跑”的策略，将最终目标进行拆分，分别形成Shenandoah 1.0、2.0、3.0……的小版本计划，现在已经可以看到Shenandoah的性能在日益改善，逐步接近“Low-Pause”的目标。此外，RedHat也积极拓展Shenandoah的使用范围，将其Backport到JDK 11甚至是JDK 8之上，让更多不方便升级JDK版本的应用也能够享受到垃圾收集器技术发展的最前沿成果。



[^1]: JDK16后ZGC在扫描根时不用stop-the-world。
[^2]: JDK15中ZGC称为正式特性。

> 注：本文为《深入理解Java虚拟机：JVM高级特性与最佳实战》读书笔记

