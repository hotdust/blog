# 前言
自己在看《Dynamic-Sized Nonblocking Hash Tables》时，看到 wait-free 和 lock-free 两个词，对具体的含义不太明白。在此记录一下网上比较通俗易懂的一些文章。


# lock-free
## 例子
```java
AtomicInteger atomicVar = new AtomicInteger(0);
public void funcLockFree() {
        int localVar = atomicVar.get();
        while (!atomicVar.compareAndSet(localVar, localVar+1)) {
               localVar = atomicVar.get();
        }
}
```

## 含义
这时lock free意味着，当你有很多线程的时候，尽管可能有一些线程是“饿”着的，但总体的吞吐量是能得到保证的--通俗的说，假如你是个线程，虽然可能一时轮不到你，但是你也不阻碍别人。而且如果执行的时间够长，那些“饿”着的线程早晚也会完成自己的任务。[Concurrency Freaks: Lock-Free and Wait-Free, definition and examples](http://concurrencyfreaks.blogspot.com/2013/05/lock-free-and-wait-free-definition-and.html)
上说说
> “A method is Lock-Free if it guarantees that infinitely often some thread calling this method finishes in a finite number of steps.”  

就是说除了极少极少可能发生的个别的倒霉蛋，非常非常多的线程还是在有限步内完成了任务，从而保证了系统的吞吐量和平均延时，实际的例子就是用一个循环CAS来修改数据。


# wait-free
## 例子
和 lock-free 差不多，只不过它为每一个要发生的操作提供了一个state array，而且为每个操作都赋予了一个number，这样保证了可以在有限步完成入队和出队的操作。

## 含义
wait-free 就更强一点，它保证任何线程都能在“有限”的步骤中取得进展--我们就说是把事情搞定吧。基本的使用CAS做循环重试的算法设计不符合wait-free，因为我们有些倒霉的线程理论上有一定概率一直等在那里，得不到执行。wait free是非常难做的，我们以前一直看不到什么wait free的数据结构和算法，直到这里有个wait - free 队列的实现，是2011年才搞出来的。 虽然这个算法也是利用cas，但它为每一个要发生的操作提供了一个state array，而且为每个操作都赋予了一个number，这样保证了可以在有限步完成入队和出队的操作。这里有这篇论文的pdf全文，写的还挺好懂的。论文：[Wait-Free Queues With Multiple Enqueuers and Dequeuers](http://www.cs.technion.ac.il/~erez/Papers/wfquque-ppopp.pdf)


# 其它
## Wait-Free Bounded
而且wait free这个有限也可能是一个很大的数。其实在实时性要求更高的系统中，我们需要更高保证的wait free，其实就是为有限步数加个上界，这个就叫“Wait-Free Bounded”，简称WFB，具体可以看上面的那个原文链接。

## Wait-Free Population Oblivious
另外现有的wait-free和wfb算法，往往是和你有多少线程相关的。比如说你有10个线程，可能最多只要等100步，但是如果你有1000个线程，可能就要最多等10000步了。这时就有了个新的概念Wait-Free Population Oblivious, 他可以保证你算法的最大完成步数的有限bound是和active的线程数是无关的。显然符合Wait-Free Population Oblivious的算法，必然也是Wait-Free Bounded的。


# 参考
- [wait-free是指什么？ - 知乎](https://www.zhihu.com/question/295904223)：本文内容就是从这个问题的答案。
- [Concurrency Freaks: Lock-Free and Wait-Free, definition and examples](http://concurrencyfreaks.blogspot.com/2013/05/lock-free-and-wait-free-definition-and.html)：lock-free 是这个文章上的例子。还讲了其它 free 很不错。这个文章主要是为每种 free 举例子，因为很多文章只有定义，没有实现。
- [Lock-free vs. wait-free concurrency - RethinkDB](https://rethinkdb.com/blog/lock-free-vs-wait-free-concurrency/)：讲了 lock-free 和 wait-free 的定义，不是很好理解，知乎那个答案比较好理解。


