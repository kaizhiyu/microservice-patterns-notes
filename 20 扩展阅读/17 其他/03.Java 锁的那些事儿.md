# Java 锁的那些事儿

- 骆向南

**2020 年 6 月 22 日

**[语言 & 开发](https://www.infoq.cn/topic/development)[编程语言](https://www.infoq.cn/topic/programing-languages)[Java](https://www.infoq.cn/topic/java)[方法论](https://www.infoq.cn/topic/methodologies)

Java 多线程开发中，如果涉及到共享资源操作场景，那就必不可少要和 Java 锁打交道。

Java 中的锁机制主要分为 `Lock`和 `Synchronized`，本文主要分析 Java 锁机制的使用和实现原理，按照 Java 锁使用、JDK 中锁实现、系统层锁实现的顺序来进行分析，话不多说，let’s go~



## 一、Java 锁使用

在 Lock 接口出现之前，Java 程序是靠 synchronized 关键字实现锁功能的，而 JavaSE 5 之后，并发包中新增了 Lock 接口（以及相关实现类）用来实现锁功能，它提供了与 synchronized 关键字类似的同步功能，只是在使用时需要显式地获取和释放锁。虽然它缺少了（通过 synchronized 块或者方法）隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、可中断的获取锁以及超时获取锁等多种 synchronized 关键字所不具备的同步特性。



Java 锁使用示例：

```
Lock lock = new ReentrantLock();lock.lock();try {    // ..} finally {    lock.unlock();}
```

注意：在 finally 块中释放锁，目的是保证在获取到锁之后，最终能够被释放。不要将获取锁的过程写在 try 块中，因为如果在获取锁（自定义锁的实现）时发生了异常，异常抛出的同时，会提前进行 unlock 导致 `IllegalMonitorStateException`异常。



Lock 相较于 Synchronized 优势如下：

- **可中断获取锁** ：使用synchronized关键字获取锁的时候，如果线程没有获取到被阻塞了，那么这个时候该线程是不响应中断(interrupt)的，而使用Lock.lockInterruptibly()获取锁时被中断，线程将抛出中断异常。
- **可非阻塞获取锁** ：使用synchronized关键字获取锁时，如果没有成功获取，只有被阻塞，而使用Lock.tryLock()获取锁时，如果没有获取成功也不会阻塞而是直接返回false。
- **可限定获取锁的超时时间** ：使用Lock.tryLock(long time, TimeUnit unit)。
- 同一个所对象上可以有多个等待队列（Conditin，类似于Object.wait()，支持公平锁模式）。

Lock 除了更多的功能之外，有一个很大的优势：synchronized 的同步是 jvm 底层实现的，对一般程序员来说程序遇到出乎意料的行为的时候，除了查官方文档几乎没有别的办法；而显示锁除了个别操作用了底层的 Unsafe 类(LockSupport 封装了 Unsafe 类)之外，几乎都是用 java 语言实现的，我们可以通过学习显示锁的源码，来更加得心应手的使用显示锁。



当然，Lock 也不是完美的，否则 java 就不会保留着 synchronized 关键字了，显示锁的缺点主要有两个：

- 使用比较复杂，这点之前提到了，需要手动加锁，解锁，而且还必须保证在异常状态下也要能够解锁。而synchronized的使用就简单多了。
- 效率较低，synchronized关键字毕竟是jvm底层实现的，因此用了很多优化措施来优化速度(偏向锁、轻量锁等)，而显示锁的效率相对低一些。



### 1.1 Synchronized

Synchronized 在 JVM 里的实现是基于进入和退出 Monitor 对象来实现方法同步和代码块同步的。monitorenter 指令是在编译后插入到同步代码块的开始位置，而 monitorexit 是插入到方法结束处和异常处，JVM 要保证每个 monitorenter 必须有对应的 monitorexit 与之配对。任何对象都有一个 monitor 与之关联，当且一个 monitor 被持有后，它将处于锁定状态。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 monitor 的所有权，即尝试获得对象的锁。

synchronized 用的锁是存在 Java 对象头里的。如果对象是数组类型，则虚拟机用 3 个字宽（Word）存储对象头，如果对象是非数组类型，则用 2 字宽存储对象头。



![img](https://static001.infoq.cn/resource/image/8a/3e/8a9e5d913e0421b0a16908c4bcd53a3e.png)



> 关于对Java象头，可以使用JOL工具（jol-core）类直接打印对象头，如下所示：



![img](https://static001.infoq.cn/resource/image/dd/7b/dd1753b0d0add53545842a40adaa417b.png)



### 1.2 锁升级

Java 1.6 为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，在 Java SE 1.6 中，锁一共有 4 种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。

> 锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率。



![img](https://static001.infoq.cn/resource/image/a3/c3/a3eaf36d571515b38aaaeafb90870bc3.png)



## 二、JDK 中锁实现

JDK 中 Lock 是一个接口，其定义了锁获取和释放的基本操作：

![img](https://static001.infoq.cn/resource/image/71/5b/717a8853daf71e571a575b3e53ea755b.png)



**Lock 底层是基于 AQS 同步器（AbstractQueuedSynchronizer）的，AQS 是用来构建锁或者其他同步组件的基础框架，它使用了一个 int 成员变量表示同步状态，通过内置的 FIFO 队列来完成资源获取线程的排队工作，并发包的作者（Doug Lea）期望它能够成为实现大部分同步需求的基础，事实上目前的 JDK 并发包都是基于 AQS 来完成同步需求的。**

关于锁和 AQS，可以这样理解二者之间的关系：锁是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和同步器很好地隔离了使用者和实现者所需关注的领域。

这里稍微分析下 AQS，其是由一个同步状态+FIFO 的同步队列组成，提供了同步队列、独占式同步状态获取与释放、共享式同步状态获取与释放以及超时获取同步状态等同步器的核心数据结构与模板方法。简单来说就是，当线程需要阻塞时就将其放到同步队列中，等到该唤醒时就将其移除队列并唤醒，使其继续工作。关于 AQS 具体的实现原理，可以参考阿里大神写的《Java 并发编程的艺术》。

AQS 当需要阻塞或唤醒一个线程的时候，都会使用 LockSupport 工具类来完成相应工作。LockSupport 定义了一组的公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而 LockSupport 也成为构建同步组件的基础工具。LockSupport 定义了一组以 park 开头的方法用来阻塞当前线程，以及 unpark(Thread thread)方法来唤醒一个被阻塞的线程。



LockSupport 提供的阻塞和唤醒方法：

![img](https://static001.infoq.cn/resource/image/26/fb/26d2ad2e3e56295d8ff824f71d2feafb.png)



LockSupport 常用方法源码如下：

```
// LockSupportpublic static void park(Object blocker) {    Thread t = Thread.currentThread();    // blocker在什么对象上进行的阻塞操作    setBlocker(t, blocker);    UNSAFE.park(false, 0L);    setBlocker(t, null);} public static void parkNanos(Object blocker, long nanos) {    if (nanos > 0) {        Thread t = Thread.currentThread();        setBlocker(t, blocker);        // 超时阻塞        UNSAFE.park(false, nanos);        setBlocker(t, null);    }} public static void unpark(Thread thread) {    if (thread != null)        UNSAFE.unpark(thread);}
```



三、系统层锁实现

UNSAFE 使用 park 和 unpark 进行线程的阻塞和唤醒操作，park 和 unpark 底层是借助系统层（Linux 下）方法 `pthread_mutex`和 `pthread_cond`来实现的，通过 `pthread_cond_wait`函数可以对一个线程进行阻塞操作，在这之前，必须先获取 `pthread_mutex`，通过 `pthread_cond_signal`函数对一个线程进行唤醒操作。

`pthread_mutex`和 `pthread_cond`使用示例如下：

```
void *r1(void *arg){    pthread_mutex_t* mutex = (pthread_mutex_t *)arg;    static int cnt = 10;     while(cnt--)    {        printf("r1: I am wait.\n");        pthread_mutex_lock(mutex);        pthread_cond_wait(&cond, mutex); /* mutex参数用来保护条件变量的互斥锁，调用pthread_cond_wait前mutex必须加锁 */        pthread_mutex_unlock(mutex);    }    return "r1 over";} void *r2(void *arg){    pthread_mutex_t* mutex = (pthread_mutex_t *)arg;    static int cnt = 10;     while(cnt--)    {        pthread_mutex_lock(mutex);        printf("r2: I am send the cond signal.\n");        pthread_cond_signal(&cond);        pthread_mutex_unlock(mutex);        sleep(1);    }    return "r2 over";}
```

**注意** ，Linux 下使用 `pthread_cond_signal`的时候，会产生“惊群”问题的，但是 Java 中是不会存在这个“惊群”问题的，那么 Java 是如何处理的呢？



> 实际上，Java只会对一个线程调用 `pthread_cond_signal`操作，这样肯定只会唤醒一个线程，也就不存在所谓的惊群问题。Java在语言层面实现了自己的线程管理机制（阻塞、唤醒、排队等），每个Thread实例都有一个独立的 `pthread_mutex`和 `pthread_cond`（系统层面的/C语言层面），在Java语言层面上对单个线程进行独立唤醒操作。（怎么感觉Java中线程有点小可怜呢，只能在Java线程库的指挥下作战，竟然无法直接获取同一个 `pthread_mutex`或者 `pthread_cond`。但是Java这种实现线程机制的实现实在太巧妙了，虽然底层都是使用 `pthread_mutex`和 `pthread_cond`这些方法，但是貌似C/C++还没这么强大易用的线程库）



具体 LockSuuport.park 和 LockSuuport.unpark 的底层实现可以参考对应 JDK 源码，下面看一下 gdb 打印处于 LockSuuport.park 时的线程状态信息：

![img](https://static001.infoq.cn/resource/image/9f/1a/9f78461df1d949fff333af1a878aad1a.png)



由上图可知底层确实是基于 pthread_cond 函数来实现的。



## 小结

了解 Java 锁机制之后，在后续的业务开发过程中，当需要进行同步时，优先考虑使用 synchronized 关键字，只有 synchronized 关键字不能满足需求时，才考虑使用显示锁（Lock）。



**本文转载自公众号有赞 coder（ID：youzan_coder）。**

**原文链接**：

https://mp.weixin.qq.com/s?__biz=MzAxOTY5MDMxNA==&mid=2455760880&idx=1&sn=9de07d01621ff5eb01a35bf739648481&chksm=8c6869d5bb1fe0c351a91179912c8c1791a5f3d86dfe265873d4a9fe85200d023fbbb77e45f4&scene=27#wechat_redirect



2020 年 6 月 22 日 14:051769



原文：https://www.infoq.cn/article/DOvfyp8kFP5YPdaTAJFF