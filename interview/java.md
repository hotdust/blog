# 并发

## 1，ReentrantLock 和 synchronized 区别
ReentrantLock ：
- 有公平和非公平锁之分。
- 时间锁等候：tryLock(timeout, unit)
- 可中断锁等候：lockInterruptibly。
- 无块结构锁：不用大括号

synchronized 不支持上面的特点。


## 2，ThreadPoolExecutor 线程池
### 构造函数参数
参数含义如下：
- corePoolSize：核心线程池大小， 当新的任务到线程池后，线程池会创建新的线程（即使有空闲线程），直到核心线程池已满。
- maximumPoolSize：最大线程池大小，顾名思义，线程池能创建的线程的最大数目
- keepAliveTime：程池的工作线程空闲后，保持存活的时间
- TimeUnit： 时间单位
- BlockingQueue<Runnable>：用来储存等待执行任务的队列
- ThreadFactory：线程工厂
- RejectedExecutionHandler： 当队列和线程池都满了时拒绝任务的策略。具体策略如下：
  - AbortPolicy（默认情况）：直接丢弃，并且抛出RejectedExecutionException异常。 
  - DiscardPolicy：直接丢弃，不做任何处理。 
  - DiscardOldestPolicy：从缓存队列丢弃最老的任务，然后调用execute立刻执行该任务。 
  - CallerRunsPolicy：在调用者的当前线程去执行这个任务。

### poolSize、corePoolSize 和 maximumPoolSize 
poolSize 是线程池中当前线程的数量。关系如下：
（1）如果poolSize<corePoolSize，新增加一个线程处理新的任务。
（2）如果poolSize=corePoolSize，新任务会被放入阻塞队列等待。
（3）如果阻塞队列的容量达到上限，且这时poolSize<maximumPoolSize，新增线程来处理任务。
（4）如果阻塞队列满了，且poolSize=maximumPoolSize，那么线程池已经达到极限，会根据饱和策略RejectedExecutionHandler拒绝新的任务。

### FixedThreadPool、CachedThreadPool、SingleThreadExecutor 和 ScheduledThreadPool 的 corePoolSize 和 maximumPoolSize 设置。
- FixedThreadPool：corePoolSize、maximumPoolSize 相同
- CachedThreadPool：corePoolSize 为 0，maximumPoolSize 则是 int 最大值
- SingleThreadExecutor：corePoolSize 和 maximumPoolSize 都是 1
- ScheduledThreadPool：corePoolSize 是用户指定的，而maximumPoolSize 则是 int 最大值

### 大概原理
虽然每个`任务`都要实现 Runnable 接口，但实际运行时不是 new 一个 Thread，把`任务`当做构造函数参数传进去。实际运行时，是用 ThreadFactory 生成的一个 Worker 线程，Worker 线程不断从`任务队列`里取得`任务`。如果没有`任务`的话，就看看`实际线程数`和 corePoolSize 大小，决定当前 Worker 是否退出。


### 参考：
- [Java8 线程池源码学习 - 简书](https://www.jianshu.com/p/a60d40b0e4e9)
- [java8线程池 - wy11933的博客 - CSDN博客](https://blog.csdn.net/wy11933/article/details/80399562)
- [JDK1.8 ThreadPoolExecutor浅析 - 简书](https://www.jianshu.com/p/847095d9aeab)


## 3，Condition.await, signal 与 Object.wait, notify 的区别
最大的区别是在实现`生产者/消费者`模式时的不同。

下面是一个说明 Condition 和 Object 方法区别的文章：
> [wait(),notify() 与 await(), signal(), signalAll() 的区别 - 萧萧的专栏 - CSDN博客](https://blog.csdn.net/lengxiao1993/article/details/81482410)

文章指出，当使用 Object.nofity 方法时，无法确定下一个执行的线程是`生产者`还是`消费者`，因为 Synchronized 进行处理是`非公平锁`。而 Condition 可以`大概`确定是`生产者`还是`消费者`，原因是每个 Condition 有一个队列，而使用 Synchronized 方式的话，只有一个队列。
> 当有 2 个生产和 2 个消费者时，虽然调用了 consumer.signal，但也无法一定保证下一个启动的线程是 consumer。因为在 AQS 队列中，`第一个 producer 线程`后面可能`第二个 producer 线程`。但可以保证`所有的 producer 线程` await 后，会有 consumer 线程启动。使用 Synchronized 方法的 Object 是无法保证这个的。

对于`每个 Condition 有一个队列，而使用 Synchronized 方式的话，只有一个队列`的内容，可以参看这个文章：[java Condition源码分析 - Code-lover's Learning Notes - CSDN博客](https://blog.csdn.net/coslay/article/details/45217069)

## 4，Lock 和 Synchronized 区别
- Lock 可中断（lockInterruptibly 方法），而 Synchronized 不可中断。
- Lock 可以设置`获取锁`的超时时间（tryLock(long time, TimeUnit unit)）
- 可以立刻返回（不管是否获取到锁）


## 5，ReentrantLock 的公平锁和非公平锁实现上有什么区别
公平锁在 lock 时，直接插入到 AQS 队列的队尾。在执行线程时，从 AQS 的队列队首开始执行。
非公平锁在 lock 时，会先看一下锁是否有人占用。如果没人占用，就直接占用锁；如果有人占用，就和公平锁操作一样了，放到 AQS 队列的队尾，按顺序执行。

参考：[Java中的公平锁和非公平锁实现详解 - 平菇虾饺 - CSDN博客](https://blog.csdn.net/qyp199312/article/details/70598480#%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%81nonfairsync)
 