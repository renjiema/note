# Lock锁

## Lock接口

在Lock接口出现之前，Java程序是靠synchronized关键字实现锁功能的，Lock提供了与synchronized关键字类似的同步功能，只是在使用时需要显式地获取和释放锁，却拥有了锁获取与释放的可操作性、可中断的获取锁以及超时获取锁等多种synchronized关键字所不具备的同步特性。

Lock接口提供的synchronized关键字所不具备的主要特性如下：

* 尝试获取锁：如果这一时刻锁没有被其他线程获取到，则成功获取锁
* 能获取可中断锁：当获取到可中断锁的线程被中断时，将会抛出中断异常并释放锁
* 超时获取锁：超时为获取到锁则返回

Lock是一个接口，它定义了锁获取和释放的基本操作，Lock的API如下所示

|                           方法名称                           | 描述                                                    |
| :----------------------------------------------------------: | ------------------------------------------------------- |
|                         void lock()                          | 阻塞获取锁，未获取到锁会一直阻塞                        |
|     void lockInterruptibly() throws InterruptedException     | 获取可中断锁，获取锁后会被中断并释放锁                  |
|                      boolean tryLock()                       | 尝试获取锁，立即返回，成功获取锁放回true，否则返回false |
| boolean tryLock(long time, TimeUnit unit) throws InterruptedException | 超时获取锁                                              |
|                        void unlock()                         | 释放锁                                                  |
|                   Condition newCondition()                   | 获取等待通知组件                                        |

## AQS

AbstractQueuedSynchronizer（简称AQS）是用来创建锁或其他同步组件的基础。AQS使用一个volatile变量修饰的int成员 变量`state`表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。

AQS的主要使用方式是继承，子类通过继承AQS并实现它的抽象方法来管理同步状态，AQS提供的3个方法（getState()、setState(int newState)和compareAndSetState(int expect,int update)）来操作同步状态，它们能够保证状态的改变是安全的。子类推荐被定义为自定义同步组件的静态内部类，AQS自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，AQS既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件。

锁是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；AQS面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和AQS很好地隔离了使用者和实现者所需关注的领域。

AQS的设计是基于模板方法模式，因此使用者需要继承AQS并重写指定的方法，随后将AQS组合在自定义同步组件的实现中，调用AQS提供的模板方法，而这些模板方法将会调用使用者重写的方法。

AQS可重写的方法如下：

| 方法名称                                    |           描述           |
| ------------------------------------------- | :----------------------: |
| protected boolean tryAcquire(int arg)       |        获取独占锁        |
| protected boolean tryRelease(int arg)       |        释放独占锁        |
| protected int tryAcquireShared(int arg)     |        获取共享锁        |
| protected boolean tryReleaseShared(int arg) |        释放共享锁        |
| protected boolean isHeldExclusively()       | 表示锁是否被当前线程独占 |

同步器提供的模板方法基本上分为3类：独占式获取与释放同步状态、共享式获取与释放同步状态和查询同步队列中的等待线程情况。