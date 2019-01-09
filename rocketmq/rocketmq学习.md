todo

rocketmq 为什么快？
1，顺序读
2，内存操作
3，NIO
4，锁内存


下一步要做的：

题：


-1，看一下下面几个文章列表，看看有什么自己看过，不一样的没有。或者自己没看过的源码部分。
http://www.iocoder.cn/categories/RocketMQ/
https://www.jianshu.com/u/26104e701b53
https://blog.csdn.net/prestigeding/article/details/78888290
https://blog.csdn.net/u013160932/article/category/6761662


OK 0，源码要看的部分：consumer,index,recover,HAService
1，把特性总结一下，各组件功能，架构，扩容。
OK [分布式消息队列RocketMQ与Kafka架构上的巨大差异之1 -- 为什么RocketMQ要去除ZK依赖？](https://blog.csdn.net/chunlongyu/article/details/54018010)：因为不需要选举。
[RocketMQ——水平扩展及负载均衡详解](http://jaskey.github.io/blog/2016/12/19/rocketmq-rebalance/)：集群，还有一些组件的内容：
[RocketMQ详解-架构模块解析](https://my.oschina.net/ericquan8/blog/807518)：关于整体和扩容。
[RocketMQ总结整理](https://blog.csdn.net/asdf08442a/article/details/54882769)

OK 2，Rocketmq 是如何做故障恢复的？如果一个数据从 page cache 写到磁盘时，写了一半时停电了，如何会变成什么样？如何恢复？使用没有使用双写技术，有没有`partial page write`问题？
CommitLog，ConsumerQueue，Index 等文件会有什么影响，会不会丢失消息（CommitLog 最先写，ConsumeQueue 和 Index 都是后写，可能存在 CosnumeQueue 写上不的情况，这样就会丢失消息。还有没有其它情况）

OK 2.1 消息重复
OK （1）如果使用 master - slave 模式。slave 死了，要去 master 消费的话，会产生重复消费。会产生多少呢？
回答：应该不会产生重复消息。因为 consumer 持有 consume queue offset，从 Master 取得这个 offset 之后的消息就可以了。


PASS 3，Index 使用的了类似 hash 桶的结构，别的地方使用了什么数据结构吗？
这种数据结构，有几点好处：
1. 相比`新来的消息排在最后面`相比，减少数据的检索。如果新来的消息排在最后面的话，要通过头结点依次向下查找，找到最后一个数据结点，然后修改这个结点的`下一条消息位置`属性。
如果`新来的消息排在最前面`，只要修改新来消息的`下一条消息位置`属性，和 HashSlot 的值即可。而且减少磁盘寻址。
2. 一般来说，比较新的消息被重新消费的可能性大一些，这样也减少了数据的查找。
3. 内容的存储是顺序的，尽量保证了写的速度。


PASS 4，meminfo 里还有 Mlock 内存显示，试试使用 rocketmq 的程序把一块内存 mlock 住。

5，看一下 LSM-Tree 的设计便是合理的利用了存储介质的特性，做到了最大化的性能利用（磁盘换成SSD也依旧能有很好的运行效率）。（LevelDB 就是使用这种方式）
Index 主要是用来查询消息的，因为 RocketMQ 可以重新取得消费过的消息。例如：使用 queryMsgByKey 接口。

PASS 6，[源码分析RocketMQ系列索引](https://blog.csdn.net/prestigeding/article/details/78888290)：看一下这个索引上面的内容，都知道不知道是如何做的

OK 1，如果写完 commitlog 后，写 ConsumeQueue 和 Index 的时候挂掉了，会怎么样？在启动时会恢复吗？回答：会恢复。
OK 2，StoreCheckpoint 这个是用来做什么的？怎么使用？
OK 3，send 方法如何决定发到哪个 queue 上？
回答：根据设置，1，一个好的 Queue（可用的、延迟小的）。2，轮循。
PASS 4，IndexFile 的存储使用是 Java7 的 HashMap 的方式来存储的，即链表Hash方式，有没有其它 Hash 方式提高性能？例如：open address Hash 或 其它数据结构。
```
int keyHash = indexKeyHashMethod(key);
int slotPos = keyHash % this.hashSlotNum;
```
OK 5，死信队列是如何做的？
如果消息超时重试最大次数，就放到死信队列中

OK 6，如果一个消费一直不消费，保存消息的那个 log 文件什么时候归档或销毁呢？
定时删除，不管是不是被消费。

消息丢失：
1，异步发送，Broker 挂了。
2，Master 接收到消息后，传送给 Slave 之前挂了，broker 挂了。
（异步消息时，如果 CommitLog 和 ConsumeQueue 有一个没有写上，消息就会丢失。同步消息时，如果 ConsumeQueue 没有写上，不会丢失消息，因为 CommitLog 恢复机制（使用 CheckPoint 中最小时间戳和恢复 ConsumeQueue），可以保证 ConsumeQueue 会被恢复。）



#基础：

[RocketMQ——角色与术语详解](http://jaskey.github.io/blog/2016/12/15/rocketmq-concept/)：介绍了一些术语，还有一些其它文章可以看看。
[十分钟入门RocketMQ](http://jm.taobao.org/2017/01/12/rocketmq-quick-start-in-10-minutes/)：阿里的介绍文章。
[分布式开放消息系统(RocketMQ)的原理与实践](https://www.jianshu.com/p/453c6e7ff81c)：总结了实践中的一些问题，并说了为什么这么处理这些问题。这个非常好，可以重复看看。
[分布式消息中间件RocketMQ学习教程①](https://blog.csdn.net/u014427391/article/details/78343163)：介绍各个组件，还没看，不知道有没有2，3，4.
[rocketmq 使用学习](https://www.jianshu.com/p/a734aca494f0)：有一些使用例子，和一些介绍，可能介绍是和上面的内容是重复的，但最好看一看。
[rocketmq 官网的架构说明](https://rocketmq.apache.org/docs/rmq-arc/)
[rocketmq 官网的best practice](https://rocketmq.apache.org/docs/core-concept/)
[RocketMQ 开发指南 3.2.4](http://www.sxt.cn/ueditor/php/upload/file/20170901/1504248466272324.pdf)：rocket 的用户指南，具体配置介绍，有一些介绍在别的文章上都没有。
[RocketMQ消息存储流程图及数据结构图](https://blog.csdn.net/qq_27529917/article/details/79595395)：关于 RocketMQ 存储的一个图，画的非常好。




简单使用：
[说说 MQ 之 RocketMQ](http://valleylord.github.io/post/201607-mq-rocketmq/)


关于MQ设计：
[消息队列设计精要](https://tech.meituan.com/mq-design.html)：美团那个设计精要


Rocketmq 和 kafka
[RocketMQ与Kafka对比（18项差异）评价版](http://blog.51cto.com/mengphilip/1619666)
[RocketMQ与Kafka对比（18项差异）](https://yq.aliyun.com/articles/25389)
[分布式消息队列RocketMQ深度解析](https://blog.csdn.net/chunlongyu/article/category/6638499)：把 rocketmq 和 kafka 的设计进行了对比，很好的文章。
[分布式消息队列Kafka源码深度解析](https://blog.csdn.net/chunlongyu/article/category/6417583)：对 kafka 源码的解读，最好看一看。这个 blog 还有其它很好的文章，都建议看看。


好的博客：
[薛定谔的风口猪](http://jaskey.github.io/blog/archives/)：从小白到设置，到各种原理，适合初学者。
[分布式消息队列Kafka源码深度解析](https://blog.csdn.net/chunlongyu/article/category/6417583)：对 kafka 源码的解读，最好看一看。这个 blog 还有其它很好的文章，都建议看看。
[分布式消息队列Kafka & RocketMQ 深度学习资料精选](https://blog.csdn.net/chunlongyu/article/details/54407633)：这个要看看，把里面有什么内容都看一下，把好的选出来
分布式消息队列Kafka & RocketMQ 深度学习资料精选中的博客：
[阿里中间件团队博客](阿里中间件团队博客http://jm.taobao.org/categories/%E6%B6%88%E6%81%AF%E4%B8%AD%E9%97%B4%E4%BB%B6/)：
[隔壁老杨的专栏](https://blog.csdn.net/u014393917/article/category/6332828)：关于 kafka 源码 0.9.0 的分析
[关于Kafka配额的讨论](http://www.cnblogs.com/huxi2b)：很多关于 kafka 的想法，可以看一看
[李志涛的专栏](https://blog.csdn.net/lizhitao/article/list/2)：
[rocketmq源码分析及注意事项](https://blog.csdn.net/a417930422/article/category/6086259)
[如何在MQ中实现支持任意延迟的消息？](https://www.cnblogs.com/hzmark/p/mq-delay-msg.html)：延迟消息的实现方式，里面写了一些算法，挺好的。



源码阅读：
[芋艿RocketMQ源码解析](http://www.iocoder.cn/categories/RocketMQ/)
[芋艿的后端小屋 知乎](https://zhuanlan.zhihu.com/yunaiv)
[薛定谔的风口猪 在知乎的文章](https://www.zhihu.com/people/linjunjie1103/posts)
[消息队列](https://blog.csdn.net/mr253727942/article/category/6445557)：有两篇源码文章，有一个写存储的写的挺多的，建议看看。
[gitbook 的文章](https://wangkang007.gitbooks.io/java/rocketmqyuan_ma_fen_xi_yi_zhi_xiao_fei_zhe_fa_song.html)：文章内容简单，有一些问题，可以看看
[RocketMQ源码学习(二)-Producer](https://segmentfault.com/a/1190000010987806)：文章不多，但有一些基础类的字段的说明。
[RocketMQ源码解读](https://blog.csdn.net/column/details/14689.html)：这个文章老一些，2016 年底的



#总结过的
##RocketMQ 的基础：
关于 roacketmq 的一些基础特性：
[RocketMQ——角色与术语详解](http://jaskey.github.io/blog/2016/12/15/rocketmq-concept/)：介绍了一些术语，还有一些其它文章可以看看。
[十分钟入门RocketMQ](http://jm.taobao.org/2017/01/12/rocketmq-quick-start-in-10-minutes/)：阿里的介绍文章。

关于架构和扩容的特性：
[分布式消息队列RocketMQ与Kafka架构上的巨大差异之1 -- 为什么RocketMQ要去除ZK依赖？](https://blog.csdn.net/chunlongyu/article/details/54018010)：rocketmq 架构
[RocketMQ——水平扩展及负载均衡详解](http://jaskey.github.io/blog/2016/12/19/rocketmq-rebalance/)：集群，还有一些组件的内容：
[RocketMQ详解-架构模块解析](https://my.oschina.net/ericquan8/blog/807518)：关于整体和扩容。
[RocketMQ总结整理](https://blog.csdn.net/asdf08442a/article/details/54882769)

关于 RocketMQ 的一些细节：
[分布式开放消息系统(RocketMQ)的原理与实践](https://www.jianshu.com/p/453c6e7ff81c)：总结了实践中的一些问题，并说了为什么这么处理这些问题。这个非常好，可以重复看看。
[rocketmq 官网的架构说明](https://rocketmq.apache.org/docs/rmq-arc/)
[rocketmq 官网的best practice](https://rocketmq.apache.org/docs/core-concept/)






#问题
1， producer group 是什么概念?
同一生产组的不同生产者实例都会被Broker联络告知提交或者回滚事务，以避免事务后源生产者崩溃。

2， 为什么要有 tag 的概念？

Tag表示消息的第二级类型，比如交易消息又可以分为：交易创建消息，交易完成消息..... 一条消息可以没有Tag。RocketMQ提供2级消息分类，方便大家灵活控制。


2.1， 如果可以把消息分为 2 级消息（例如交易创建、交易完成），为什么不使用多个 topic？
感觉有点像封装的概念，因为同一业务模块的消息可以放到同一个 topic 里，不用做成一个个 topic。
入库的消息（入库开始、入库中转、入库完成）、交易的消息（交易创建、交易完成）都放到一个 topic里，不用分成多个 topic。

2.2， 这样做会不会影响效率？
网上说：Tags有益于保持你的代码干净而条理清晰，同时促进使用RocketMQ提供的查询系统的效率。
但不知道是怎么做的？


3， consumer group 消费是如何消费的？假如有2个 queue， 3 个 consumer， 如何消费？
回答：有两个 consumer 消费两个 queue，有一个 consumer 在空闲。

4， 队列总数发生发化，消费者会触发负载均衡，而默认地负载均衡算法采取哈希取模平均，这样负载均衡分配到定位的队列会发化，使得队列可能分配到别的实例上，则会短暂地出现消息顺序不一致。
回答：记录这个问题，想一想如何解决。可不可以使用一致性hash类似的算法。

另外，顺序消息的一种，无论正常异常情况都能保证顺序，但是牺牲了分布式 Failover 特性，即 Broker集群中只要有一台机器不可用，则整个集群都不可用，服务可用性大大降低。

如果服务器部署为同步双写模式，此缺陷可通过备机自动切换为主避免，不过仍然会存在几分钟的服务不可用。（依赖同步双写，主备自动切换，自动切换功能目前并未实现）

目前已知的应用只有数据库 binlog 同步强依赖严格顺序消息，其他应用绝大部分都可以容忍短暂乱序，推荐使用普通的顺序消息


5， activemq 是如何消费消息的呢？假如有两个 master slave 和 两个 consumer，是每个从一个 ms 从取消息，还是随机取消息。

6，RocketMQ是如何做限流的呢？activemq 的话 windowSize 是用来在 client 端做限流的，Flow Control 是用来在 borker 端做限流的。参考：[ActiveMQ中Producer特性详解](http://shift-alt-ctrl.iteye.com/blog/2034440)

--------------
1， share-nothing design paradigm

2， master slave 是如何工作的， 2 copies 3 copies 是如何做的，是使用多个 slave 吗？

3， 像 kafka 一样做多个 partition 是如何做的？

4， 为什么要回滚所有 group 内的 producer，这样有什么好处，什么地方需要这么做？

5， ASYNC_MASTER, SYNC_MASTER slave 是什么意思？
Broker Role is ASYNC_MASTER, SYNC_MASTER or SLAVE. If you cannot tolerate message missing, we suggest you deploy SYNC_MASTER and attach a SLAVE to it. If you feel ok about missing, but you want the Broker to be always available, you may deploy ASYNC_MASTER with SLAVE. If you just want to make it easy, you may only need a ASYNC_MASTER without SLAVE.


6， 因为异步刷盘，可能造成消息的丢失，所以 rocketmq 推荐下面的使用方法，这个方法是怎么回事？
ASYNC_FLUSH is recommended, for SYNC_FLUSH is expensive and will cause too much performance loss. If you want reliability, we recommend you use SYNC_MASTER with SLAVE.

7， 记录下面的特性
Duplication or Missing
But keep in mind that resending is useless when you get SLAVE_NOT_AVAILABLE. If this happens, you should keep the scene and alert the Cluster Manager.

8， 记录下面消息重新的发生场景
Many circumstances could cause duplication, such as:

Producer resend messages(i.e, in case of FLUSH_SLAVE_TIMEOUT)
Consumer shutdown with some offsets not updated to the Broker in time.
So you may need to do some external work to handle this if your application cannot tolerate duplication. For example, you may check the primary key of your DB.

9， 关于 shard-nothing 结构，看一下下面的资料。
http://www.benstopford.com/2009/11/24/understanding-the-shared-nothing-architecture/
https://en.wikipedia.org/wiki/Shared-nothing_architecture
https://en.wikipedia.org/wiki/Shard_(database_architecture)
https://www.quora.com/Whats-the-difference-between-sharding-DB-tables-and-partitioning-them

https://www.zhihu.com/question/20120283
https://blog.csdn.net/seteor/article/details/10532085

再找一下：Shared Nothing sharding
再看一下：redis sharding

10， rocketmq 的集群使用是 shared-nothing 模式， 它是如何使用的呢？
kafka 是使用的什么模式呢？



# 面试题
## 1，顺序队列如何扩容？成倍增长队列，让之前的队列消费完成。
主要做两件事：
- 扩容队列时，按原来队列个数“倍数”扩容。
- 先禁止新扩容的队列读，保证老队列里的扩容时点数据被消费完了后，再放开新队列的读。

**（1）扩容队列时，按原来队列个数“倍数”扩容。**
按原来队列倍数进行扩容，是为了保证`扩容后，发往扩容前某个队列的消息，不发往扩容前的其它队列。`

例如：扩容前，对于“消息`0~5`”这 6 条消息（0~5 是发向哪条队列时，进行取模的编号，会有多条相同性质的消息的号码都是0 或 1 等。我们要保证有相同属性的消息都发向同一个队列），发往“队列`0、1、2`”三条队列时，分别用消息号对`队列个数`进行取模运算，例如：0%3=0、1%3=1，结果如下：
消息名 | 取模后 | 队列名 
--|--|--
消息0、消息3 | 0 | 队列 0
消息1、消息4 | 1 | 队列 1
消息2、消息5 | 2 | 队列 2

扩容后，由 3 个队列扩容到 6 个队列，则用消息号对 6 进行取模，结果如下：
消息名 | 取模后 | 队列名 
--|--|--
消息0 | 0 | 队列 0
消息1 | 1 | 队列 1
消息2 | 2 | 队列 2
消息3 | 3 | 队列 3
消息4 | 4 | 队列 4
消息5 | 5 | 队列 5

扩容后，`消息 345` 从 `队列012` 改发向`队列 345`。这样的话，可以保证对于`消息 345`，`队列012`里的消息一定会比 `队列345`里的消息旧。而对于`消息 012`，发往的队列没有改变。有了这个保证，就可以进行下一步操作了。

**（2）先禁止新扩容的队列读，保证老队列里的扩容时点数据被消费完了后，再放开新队列的读。**
因为`消息 012` 发往的队列没有改变，所以我们不对它们进行关注，我们主要对`消息 345`进行关注。

对于`消息 345`，因为`队列012`的消息比`队列345`里的消息旧，所以我们保证先消费完`队列012`里的消息，再消费`队列345`里的消息，就可以保证消息的顺序性。所以第 1 步保做的是禁止对`队列345`里的消息进行消费。第 2 步是让消费者继续对`队列012`进行消费，等消费到扩容时间点的消息后，就可以保证`消息345`在`队列012`的消费就已经被全消费完了。这样，再打开`队列345`让消费者进行消费`消息345`的消息，就可以保证消息的顺序性了。

以上就是扩容的过程了。如果我们不按队列倍数进行扩容呢？

<br>

**其它**
（1） 如果我们不按倍数扩容，会怎么样呢？
我们从 3 个队列扩容到 5 个队列，看看消息的去向。下表是扩容后的消息去向，把下面的表和上面的进行对比，可以看出`消息3~5`发送的队列改变了。消息3 改变后影响不太大，消息4和5 影响特别大，因为他们从一个`扩容前的队列`，发往了另一个`扩容前的队列`。这样我们就无法使用上面的 (2) 的做法来，保证让旧消息先被消费完成了。
消息名 | 取模后 | 队列名 
--|--|--
消息0 | 0 | 队列 0
消息1 | 1 | 队列 1
消息2 | 2 | 队列 2
消息3 | 3 | 队列 3 （原来`消息3`是发往队列 0 ）
消息4 | 0 | 队列 0 （原来`消息4`是发往队列 1 ）
消息5 | 1 | 队列 1 （原来`消息5`是发往队列 2 ）

（2）RocketMQ 中禁止新队列消费的代码如何做的呢？


代码：
`UpdateTopicSubCommand#execute` 方法中有设置 readQueueNums 的处理。
```
// readQueueNums
if (commandLine.hasOption('r')) {
    topicConfig.setReadQueueNums(Integer.parseInt(commandLine.getOptionValue('r').trim()));
}
```


`PullMessageProcessor#processRequest` 方法是消费消息的方法，在这个方法中下面的代码中，来判断 queue 是不是可以读。这里的判断不是具体哪一个 queue 可不可以读，是大于 queueNum 的 queue 都不可以读。
```
if (requestHeader.getQueueId() < 0 || requestHeader.getQueueId() >= topicConfig.getReadQueueNums()) {
    String errorInfo = String.format("queueId[%d] is illagal, topic:[%s] topicConfig.readQueueNums:[%d] consumer:[%s]",
            requestHeader.getQueueId(), requestHeader.getTopic(), topicConfig.getReadQueueNums(), channel.remoteAddress());
    LOG.warn(errorInfo);
    response.setCode(ResponseCode.SYSTEM_ERROR);
    response.setRemark(errorInfo);
    return response;
}
```


-----------------
# 交流
mail list :
https://lists.apache.org/api/source.lua/2ee9ae93c57466f6e7c2d9dcfed8482d5ca0ebb3b41a24ec243ecf79@%3Cusers.rocketmq.apache.org%3E
https://lists.apache.org/thread.html/2ee9ae93c57466f6e7c2d9dcfed8482d5ca0ebb3b41a24ec243ecf79@%3Cusers.rocketmq.apache.org%3E

2018 北京 meetup 视频：
https://yq.aliyun.com/webinar/play/506?spm=a2c4e.11154022.0.0.48377b42bovwcd

-------------------
关于分片：
1，数据库如何分片？有几种方式

2， 为什么：哈希取模对于扩容或缩容的目标节点数不是倍数或整除数的情况，需要在旧的节点之间迁移数据。

3，数据库存储时，一致性 hash 可能造成有的结点上已经满了，还向它插入。lish + hash 方式，当一个满了的话，就不向里进行插入了。

https://en.wikipedia.org/wiki/Partition_(database)
https://yq.aliyun.com/articles/57954
https://blog.csdn.net/ydyang1126/article/details/70313981
https://blog.csdn.net/ydyang1126/article/details/70773557
https://blog.csdn.net/icycolawater/article/details/7255109
https://blog.csdn.net/bluishglc/article/details/7970268：这个不错可以看看
https://blog.csdn.net/column/details/sharding.html
https://blog.csdn.net/wangshuang1631/article/details/62898280
https://blog.csdn.net/kingice1014/article/details/76136337