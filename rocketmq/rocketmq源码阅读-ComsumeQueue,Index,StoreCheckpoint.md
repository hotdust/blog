#一、存储结构
![这里写图片描述](https://img-blog.csdn.net/20180710095115309?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

#二、ConsumeQueue
Consumer 在消费消息时，要指定消费哪个 Topic，再定位从 Topic 下的哪个 Queue 来取得消息。从 Queue 取得消息相关数据后，再用相关数据从 CommitLog 中取得具体的消息内容。而 ConsumeQueue 类就是对应每个 Queue 文件。


##ConsumeQueue 内容

- CommitLog offset(8)：消息存储的物理位移。保存的值为CommitLog的 PHYSICALOFFSET。
- Size(4)：消息的长度。保存的值为CommitLog的TOTALSIZE。
- Message tag HashCode(8)：producer 端指定消息的tags属性的 hashcode（tags.hashCode()）。

![这里写图片描述](https://img-blog.csdn.net/20180705182348520?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



##ConsumeQueue 写过程
###1，写消息到 buffer
ConsumeQueue 的写入是由 ReputMessageService 类来处理的。这个类是在 DefaultMessageStore 类里面被启动的，和它一起启动的还有 CommitLog、HAService 等服务类。这个类也是一个异步定时类，每隔一段时间启动一次，每次启动后都要比较是否`当前落盘的消息的 offset > 自己存储的“前一回落盘的消息的 offset”`，如果为 true 就读取那些最新的已经落盘的消息，然后保存到 ConsumeQueue。默认是保存在`store/consumequeue`目录下，再以 topic 和 queueid 再细分子目录，例如：`store/consumequeue/TopicTest/1`、`store/consumequeue/TopicTest/2`。

ReputMessageService#doReput 方法里做的，这个方法最终调用 DefaultMessageStore#doDispatch 方法， 进行调用具体的写 ConsumeQueue 方法：DefaultMessageStore#putMessagePositionInfo

###2，把 buffer 内容写到磁盘
上面的 putMessagePositionInfo 方法只是写数据 ByteBuffer，而把 ByteBuffer 内容写到磁盘则是由 FlushConsumeQueueService 类执行的。这个类也是一个定时执行类，定时调用 ConsumeQueue.flush 方法，把内存中的数据写到磁盘上。





#三、IndexFile
IndexFile 是用来为每条消息创建索引（一条索引或多条索引）。在想要重新读取某条消息时，指定 topic 和 key 在索引文件中，取得指定消息的`所在 CommitLog 上的位置信息`。之后，再通过 CommitLog 信息，取得信息的消息内容（这部分就不是 IndexFile 所负表的部分了）。保存的文件在`store/index`文件夹下，例如：store/index/20180709090040132。

##IndexFile 结构
IndexFile 的结构是由 3 个部分组成：

- IndexHeader：保存 Index 文件的一些 Meta 信息。例如：IndexFile 中第一条消息的时间和最后一条消息的时间等，下面会详细介绍。
- HashSlot：保存消息 `Index 内容`的具体位置。`消息 topic + Key` 进行 Hash 后的值为 HashSlot。
- Index：保存`消息 topic + Key`进行 Hash 后，Hash 值相同的`消息的 Index`内容。`Index 内容`包括：keyHash、phyOffset、timeOffset、slotValue

其中 Index 部分的`Index 内容`部分是要存储的具体内容，大部分的空间都用来保存它了。具体结构如下：

- keyHash：`消息 topic + Key`的 HashCode
- phyOffset：消息的在 CommitLog 中的物理位置
- timeOffset：消息和 IndexFile 的第一条消息的时间位移（当前消息在 IndexFile 中的第一条消息之后多长时间。为什么使用时间位移，可能是因为要减少存储的大小，时间差是 int，而绝对时间是 long）
- slotValue：前一条消息的 Index 部分的位置（用这个字段做的链表）

上面结果的图如下：
![这里写图片描述](https://img-blog.csdn.net/20180705182405987?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
上面图中，Slot Table 部分是 HashSlot 部分，而 IndexLinkedList 部分则是 Index 部分。右上角部分的图是`Index 内容`。

#IndexFile 实现
其实这个文件的结构就是 Java7 中的 HashMap 的结构。不同点有两点：

- HashMap 因为是在内存中使用，所以使用是`对象引用`的方式创建链表的。而 IndexFile 是文件，所以使用的`类似数组下标`的方式创建链表的。
- HashMap 中链表的顺序是，先进来的排在第 1 个，最后进来的排在最后 1 个。而 IndexFile 中的 Index 部分的链表存储方式是，先进来的排在最后 1 个，最后进来的排在第 1 个。

**下面详细说一下 IndexFile 的实现方式：**

每个消息的具体 `Index 内容`是保存在 Index 部分的，而且是按顺序存储的，也就是说第 1 个消息就保存在 Index 部分的第 1 个位置的，第 2 个消息就保存在 Index 的第 2 个位置，不管他们的 Hash 值是否相同。如何知道某个消息的 `Index 内容`保存在 Index 的哪个位置上呢？就要通过 HashSlot 了，HashSlot 是通过`消息 topic + Key` 的 Hash 值算出来的，所以每个消息的 HashSlot 位置是固定的。把 Index 位置保存到 HashSlot 中后，就可以通过 HashSlot 取得 Index 中的位置了。

在保存`Index 内容`时，先算出 `Index 内容`应该保存在 Index 部分哪个位置，先把`Index 内容`保存了。在保存之前，还要取出 HashSlot 中的值（也就是 Hash 值相同的消息所在 Index 部分的位置），放到当前消息的`前一条消息的 Index 部分的位置`字段上，这样当前消息就连接到前一条 Hash 值相同的 `Index 内容`上了。之后，再用当前消息所在的 Index 部分的位置，更新 HashSlot 的值，这样当前消息也可以从 HashSlot 中找到了。

关于 IndexFile 的落盘。IndexFile 是通过 MMAP 写数据的，那多长时间进行一次强制刷盘呢？当 IndexFile 满了时才进行强制刷盘。虽然强制刷盘的时间间隔很长，但 Linux 系统的 pdflush 还是会根据一些指标不定期刷盘的，例如：page cache 大小超过定额刷盘、一定时间之后刷盘一次等。



## 具体代码部分
IndexFile 的写过程和 ConsumeQueue 有一部分相似，都是通过 ReputMessageService 类来取得已经落盘的消息，然后在 ReputMessageService#doReput 方法调用 DefaultMessageStore#doDispatch 方法， 进行调用具体的写 IndexFile 方法：IndexService#buildIndex。

IndexService 是一个用来处理 IndexFile 的类。这个类在 DefaultMessageStore 中被实例化，并调用了 start() 方法，但实际看一下 start() 方法，发现里面并没有任何实现。这个类主要是用来处理 Index 的保存和查询的。

其它：

- DefaultMQProducer#queryMessage 查询消息的接口。QueryMessageProcessor#queryMessage 方法是用来接收`查询消息`的接口。
- 在保存 Index 时，会使用 UniqKey 字段保存一个 Index，还会使用 Key 字段保存一份或多份 Index（在使用 Key 保存 Index 时，会用空格进行切割，每个 Key 都保存一份 Index）




#四、StoreCheckpoint
StoreCheckPoint 类是用来记录 CommitLog、ConsumeQueue 和 Index 的最后一次落盘时间的。在恢复 CommitLog 数据时，根据 CheckPoint 中记录的时间来恢复，以 CommitLog、ConsumeQueue 和 Index 三个时间戳中最小的为恢复起点。

在记录数据时，把数据写到磁盘后，再记录`最后一次写`的时间，也就是记录 CheckPoint。CheckPoint 是用来保证记录时间点之前的数据是安全的，但 CheckPoint 时间点之后的数据不能算是丢失。

以下是各文件写和刷新 CheckPoint 的时间

- CommitLog：在刷盘的 GroupCommitService 和 FlushRealTimeService 中都有写时间的处理，但没有对 CheckPoint 刷盘的处理。
- ConsumeQueue：在 ConsumeQueue 类中进行写 ConsumeQueue 数据时，会有写时间的处理。对 CheckPoint 进行刷盘的处理，是在 FlushConsumeQueueService 类中进行的。FlushConsumeQueueService 这个类是定期对 ConsumeQueue 和 CheckPoint 中的数据进行刷盘的类。
- Index：在 IndexService 中对 CheckPoint 进行写，并进行刷盘。也是在写完数据后，再进行 CheckPoint 的操作。

> 因为所有的 CheckPoint 中的记录的时间，都是在相应的数据刷盘成功后再写入的，并且两个操作是顺序操作，所以可以保证 CheckPoint 的记录的时间所对应的数据，都正常写到磁盘。

这个类保存文件名为`checkpoint`，默认在`store`根目录下。这个3个long的值为：

- physicMsgTimestamp为commitLog最后刷盘的时间
- logicMsgTimestamp为consumeQueue最终刷盘的时间
- indexMsgTimestamp为索引最终刷盘时间


# 特点
## 1，关于 Index 的数据结构
这种数据结构，有几点好处：
1. 相比`新来的消息排在最后面`相比，减少数据的检索。如果新来的消息排在最后面的话，要通过头结点依次向下查找，找到最后一个数据结点，然后修改这个结点的`下一条消息位置`属性。
如果`新来的消息排在最前面`，只要修改新来消息的`下一条消息位置`属性，和 HashSlot 的值即可。
2. 一般来说，比较新的消息被重新消费的可能性大一些，这样也减少了数据的查找。
3. 内容的存储是顺序的，尽量保证了写的速度。




#参考：
[RocketMQ原理解析-broker 3.load&recover](https://blog.csdn.net/quhongwei_zhanqiu/article/details/39144325)：参考了 StoreCheckpoint 的作用。
[RocketMQ消息存储流程图及数据结构图](https://blog.csdn.net/qq_27529917/article/details/79595395)：首页的图非常棒，把主要存储关系都画出来。
[rocket mq 底层存储源码分析(4)-索引构建](https://www.jianshu.com/p/97beacc85439)：讲了 Index 创建的代码
[源码分析RocketMQ之消费队列、Index索引文件存储结构与存储机制-上篇](https://blog.csdn.net/prestigeding/article/details/79156276)：讲了 Index 和ConsumeQueue 的代码。
[《RocketMq》二、存储篇](https://blog.csdn.net/xxxxxx91116/article/details/50333161)：大概讲了存储的各种结构。
[rocket mq 底层存储源码分析(1)-存储总概](https://www.jianshu.com/p/f01534f21b71)：讲了各种存储结构