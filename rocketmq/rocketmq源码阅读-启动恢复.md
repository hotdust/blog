RocketMQ 在启动时，对数据和一些数据结构进行恢复处理。主要恢复的文件：CommitLog、ConsumeQueue、Index。

#一、处理流程
程序入口在 DefaultMessageStore#load，主要逻辑如下：

- 判断系统是否是异常关闭。
- 如果开启延迟消息，加载延迟消息服务。
- 加载 CommitLog 文件。
- 加载 ConsumeQueue 文件。
- 如果加载 CommitLog、ConsumeQueue 文件失败，就关闭 broker。
- 提取最近的 CheckPoint
- 继续加载 Index 文件
- 恢复 CommitLog、ConsumeQueue、TopicQueueTable


##1，关系系统是否异常关闭
通过 abort 文件是否存在，来判断文件是还异常关闭。如果正常关闭的情况下，是会删除掉这个文件的；异常关闭的时候，不会删除掉这个文件。

**什么是正常关闭呢？**
普通的`kill pid`方式就属于正常关闭。`BrokerStartup#createBrokerController`方法中注册了一个 shutDownHook 方法，在普通的 kill 时会调用这个方法。这个方法里调用了 controller.shutDown 方法，走正常关闭流程。

##2，如果开启延迟消息，加载延迟消息服务
从代码上来看，一定会开始延迟消息服务的。延迟消息服务的加载处理，主要是初始化一些运行时使用的常量。
```
if (null != scheduleMessageService) {
    result = result && this.scheduleMessageService.load();
}
```

##3，加载 CommitLog 文件
加载 CommitLog 文件是指，生成 MappedQueue，生成已经存在的 MappedFile 对象，并打开 MappedFile 对应的文件。至于文件的内容合法性处理，在后面的恢复中做。

##4，加载 ConsumeQueue 文件。
和上面的`加载 CommitLog 文件`类似，主要是把结构生成出来。

##5，如果加载 CommitLog、ConsumeQueue 文件失败，就关闭 broker。
CommitLog、ConsumeQueue 文件是必须的两个文件，如果这两个文件加载有文件，broker 无法工作，就关闭掉。

##6，提取最近的 CheckPoint
StoreCheckPoint 类是用来记录 CommitLog、ConsumeQueue 和 Index 的最后一次落盘时间的。在恢复数据时，根据 CheckPoint 中记录的时间来恢复，以 CommitLog、ConsumeQueue 和 Index 三个时间戳中最小的为恢复起点。

在记录数据时，把数据写到磁盘后，再记录`最后一次写`的时间，也就是记录 CheckPoint。CheckPoint 是用来保证记录时间点之前的数据是安全的，但 CheckPoint 时间点之后的数据不能算是丢失。

以下是各文件写和刷新 CheckPoint 的时间

- CommitLog：在刷盘的 GroupCommitService 和 FlushRealTimeService 中都有写时间的处理，但没有对 CheckPoint 刷盘的处理。
- ConsumeQueue：在 ConsumeQueue 类中进行写 ConsumeQueue 数据时，会有写时间的处理。对 CheckPoint 进行刷盘的处理，是在 FlushConsumeQueueService 类中进行的。FlushConsumeQueueService 这个类是定期对 ConsumeQueue 和 CheckPoint 中的数据进行刷盘的类。
- Index：在 IndexService 中对 CheckPoint 进行写，并进行刷盘。也是在写完数据后，再进行 CheckPoint 的操作。

> 因为所有的 CheckPoint 中的记录的时间，都是在相应的数据刷盘成功后再写入的，并且两个操作是顺序操作，所以可以保证 CheckPoint 的记录的时间所对应的数据，都正常写到磁盘。


##7，继续加载 Index 文件
加载 Index 文件过程，就是对每个 Index 文件进行以下处理：

- 读取文件的 Header 等信息
- 判断是否`是异常关闭` 并且 `文件的 EndTimestamp > CheckPoint 中 Index 的时间戳`，符合条件的话就把文件删除掉；否则就继续下一个文件。

> 为什么要把整个文件删除掉呢？


##8，恢复 CommitLog、ConsumeQueue、TopicQueueTable
这个处理中内容相对多一些，复杂一些。我们每个都看看。

###（1）恢复 ConsumeQueue
恢复 ConsumeQueue 的过程就是依次对每个 consumeQueueTable 里的 ConsumeQueue 做具体的加载处理。在第 4 步`加载 ConsumeQueue`时，只是做了创建 ConsumeQueue（和相应的 MappedFile） 并且放到，已经放到 consumeQueueTable 里了。在这里都做了什么事呢？恢复内容如下：

- 把数据加载到内存中，加载到内存里的方式。就是把读取一遍数据，这样数据就自放到 page cache 中了。恢复的话，不是所有文件都恢复，只恢复最近 3  个文件，可能是最近 3 个文件利用率比较高吧。
- 设置 ConsumeQueue 的相关属性：
 - mappedFileQueue：flushedWhere、committedWhere
 - maxPhysicOffset（这个属性是记录 ConsumeQueue 里消息的最大`物理 offset`。主要作用是，防止消息被多次加入到 ConsumeQueue 里。因为在恢复消息时，会调用创建 ConsumeQueue 数据的方法，如果消息的 phyOffset 小于 这个属性的话，就不会为消息创建 ConsumeQueue 的数据了）

在加载最后，通过设置一下`ConsumeQueue 最大 offset`、FlushWhere、CommittedWhere 等属性，再删除掉不合法文件。什么样的文件是不合法的文件呢？不合法文件是：

- `文件初始 offset（也就是文件名中的数字）`< `ConsumeQueue 最大 offset`  并且 `文件最大 offset（初始 offset + fileSize）` > `ConsumeQueue 最大 offset`


 为什么这样的文件是不是合法文件呢？因为在恢复过程中，`offset < 0` 或 `size <= 0` 的数据都被认为是不合法的消息，不在继续恢复这个文件后面的消息了。如果第一条消息就是不合法数据的话，那么这个文件就是不合法文件，这个文件就被跳过去了，看看倒数第二个文件是不是合法文件。如果倒数第 2 个文件是合法的，那么`ConsumeQueue 最大 offset`就是倒数第 2 个文件里的内容。这样倒数第 1 个文件就符合上面的判断条件了，就应该被删除了。因为 MappedQueue 很多操作都是从最后一个文件取得数据，所以最后一个文件不合法的话，必须删除掉。


###（2）恢复 CommitLog
恢复 CommitLog 分两种方式：正常退出恢复 和 异常退出恢复。这两种的区别是：

- 正常退出恢复：恢复最后 3 个 MappedFile 文件，恢复顺序为从`倒数第 3 个文件`开始恢复，恢复到最后一个文件。
- 异常退出恢复：
 - 从最后一个文件开始，向前找，看哪个应该从哪个文件恢复。找到后，从这个文件开始恢复，一直恢复最后一个文件。也就是说，从后向前找适合恢复的文件，找到后再从当前文件向后进行恢复。
 - 恢复消息时，同时再进行创建一次 ConsumeQueue 和 Index 数据，防止这两个数据因为保存失败，造成消息无法被消费。

下面分别说一下每种恢复的具体处理内容：

####**正常退出恢复**

- 找到倒数第 3 个文件，并取得文件开始 offset（也就是文件名）
- 取得每一条消息，进行 check，然后进行下面的流程
 - 如果 check 成功，就记录`处理成功的数据的 offset`
 - 如果 check 成功 并且 size = 0，说明到了文件尾了，就读取下一个文件进行此流程。
 - 如果 check 失败，说明此条消息有问题，就不进行后面数据的 check了。
（出现这样的情况后，就认为前一条 check 成功的数据为最后一条保存成功的数据。当前这个 MappedFile 后面的 MappedFile 都是有问题的文件，应该删除掉）
- 当前文件 check 完后，用`处理成功的数据的 offset`来修改 CommitLog 的 FlushWhere、CommitedWhere 等数据，来记录最后一次刷盘的位置。
- 删除掉不合法文件。不合法文件定义和 ConsumeQueue 中的不合法文件定义一样，调用的方法都是同一个方法。但这个情况可能只会在一种情况下发生，比如，消息是在写在新的 MappedFile 的第一条消息，写这个消息时出错了，这样这个新文件就可以不要了。但像下面这个场景感觉应该不会发生：在恢复倒数第二个文件一半时，有一条消息出错了，但最后一个文件的消息都是合法的消息。
- 接着取得后一个文件，按此流程进行恢复。


####**异常退出恢复**
异常退出恢复的处理流程，和正常情况的很像。只是恢复顺序和个数不同。

- 先要找到第一个合法的文件。合法的意思是：
 - 文件第一条消息的 `magic code == MagicCode 常量` 并且 `storeTimestamp != 0`。
 - 并且第一条消息的时间戳 < CheckPoint 中 3 个时间戳中最小的时间戳
- 从找到的文件开始，向后进行恢复每个文件。恢复消息时，同时再进行创建一次 ConsumeQueue 和 Index 数据，防止这两个数据因为保存失败，造成消息无法被消费。

**恢复消息时，为什么要同时再创建一次消息的 ConsumeQueue 和 Index 数据呢？**
因为是异常退出，并且 ConsumeQueue 和 Index 数据的创建都是异步的，所以有可能这两个数据没有创建成功，这样 Consumer 就无法消费到消息。

**会不会创建重复的 ConsumeQueue 和 Index 数据呢？**
对于 ConsumeQueue，在`ConsumeQueue#putMessagePositionInfo`方法中，有一个判断来防止重复创建数据：`if (offset <= this.maxPhysicOffset)`，maxPhysicOffset 是保存过的最大的 CommitLog offset。

对于 Index，在`IndexService#buildIndex`方法中，也有一个判断来防止重复创建数据：`if (msg.getCommitLogOffset() < endPhyOffset)`。



###（3）恢复 TopicQueueTable
TopicQueueTable 是用来保存每个`ConsumeQueue`的。Topic 下的每一个 Queue 都有一个对应的 ConsumeQueue。

恢复 TopicQueueTable 主要是恢复每个 ConsumeQueue 的 minLogicOffset。minLogicOffset 的作用是，记录 ConsumeQueue 最小可被消费的 offset。minLogicOffset 在用户在想从头取得 topic 所有消息时使用。

恢复的过程也很简单：

- 从 CommitLog 中取得`最小物理 offset`
- 从前往后，取得每个 ConsumeQueue 对应的 MappedFile，从中取得每条消息保存的`物理 offset`，和`最小物理 offset`进行比较。当 ConsumeQueue 中的数据保存的`物理 offset` >= `最小物理 offset`，就把这个 ConsumeQueue 数据的位置记录到 minLogicOffset。


#二、细节
##1，在恢复异常 CommitLog 时，isMappedFileMatchedRecover 方法中，为什么要拿 CheckPoint 中“最小的时间戳”和第一条记录的时间戳比较呢？
因为在异常退出时，很有可能数据没有写完整，造成部分数据损坏。CheckPoint 的记录的时间所对应的数据都可以保证写成功，所以如果我们要解决数据损坏的问题，最简单办法就是读取 CheckPoint 中的 physicMsgTimestamp，从包含这个时间戳的 CommitLog 的 MappedFile 开始恢复就可以。

但如果 `physicMsgTimestamp < logicsMsgTimestamp or indexMsgTimestamp` 的话还好，在恢复 CommitLog 时同时创建了 ConsumeQueue 和 Index。但如果 `physicMsgTimestamp > logicsMsgTimestamp or indexMsgTimestamp`的话，说明有一部分消息的 ConsumeQueue 数据和 Index 数据可能没有创建成功（因为 ConsumeQueue 和 Index 数据创建是异步的，所以可能在创建这些数据时机器挂掉了，造成创建失败）。如果在这种情况下，还是从 physicMsgTimestamp 的时间戳开始恢复，可能就会有一部分消息的 ConsumeQueue 和 Index 数据不被创建，结果造成 Consumer 无法消费到这些消息。但如果找比`min(physicMsgTimestamp, logicsMsgTimestamp, indexMsgTimestamp)`小的文件开始恢复的话，就会不产生消费不到的情况。
> 例如：最后写了 3 条数据，前 2 条数据是第 N - 1 个文件的最后两条数据，第 3 条数据是第 N 个文件的第一条数据，这样 physicMsgTimestamp 的时间就是第 N 个文件的第一条数据的时间戳。但在写这 3 条数据的 ConsumeQueue 和 Index 时机器挂了，这样 logicsMsgTimestamp 和 indexMsgTimestamp 的时间就小于 physicMsgTimestamp。这如果从第 N 个文件开始恢复的话，前两条写入的数据的 ConsumeQueue 和 Index 数据就无法被创建。但如果用`min(physicMsgTimestamp, logicsMsgTimestamp, indexMsgTimestamp)`的时间找文件的话，就会找到第 N-1 个文件，从这个文件头开始恢复数据，就会创建那没有被创建的 ConsumeQueue 和 Index 补上。

最后，不怕 CheckPoint 时间戳写入不完整吗？感觉有这个可能性，因为 CheckPoint 的写方式，不是使用安全文件写入方式`MixAll#string2File`。是不是因为数据内容比较小，只会一次全写到磁盘上，或全不写到磁盘上，所以不使用安全文件写入的方式。



##2，对于一条消息，如果 CommitLog、ConsumeQueue、Index 其中有一个没有保存上，会产生什么影响？

| CommitLog | ConsumeQueue | Index | 结果 |
| :-: | :-: | :-: | :-: |
| 失败 | 成功 | 成功 | 会丢失消息，因为消息没有被保存成功。在恢复阶段，不会通过 CommitLog 数据的最后 offset，来修改 ConsumeQueue 和 Index 数据。当 Consumer 消费这个丢失的数据时，会接收到 Message Removed 的响应 |
| 成功 | 失败 | 成功 | 可能正常消费消息。在恢复 CommitLog 里的消息时，会对消息重新创建一次 ConsumeQueue 和 Index 数据 |
| 成功 | 成功 | 失败 | 同 ConsumeQueue 失败结果 |


#三、问题
##1，关于数据写入不完整问题。
（1）在 `ConsumeQueue#recover` 方法中，使用 `if (offset >= 0 && size > 0)` 来判断应该恢复到哪。

- 在异常退出时，会不会发生数据保存不完整的问题呢？例如，tagCode 没有保存上。
- 使用 CheckPoint 中的 ConsumeQueue 相关的时间戳恢复，应该就能保证安全。但可能会造成一部分消息的丢失。

（2）在写 StoreCheckPoint 时，没人使用安全文件写入方式`MixAll#string2File`，不怕产生数据写入不完整的问题吗？是不是因为数据内容比较小，只会一次全写到磁盘上，或全不写到磁盘上？

##2，关于 CommitLog 和 ConsumeQueue 恢复的问题。
CommitLog 恢复时，如果 size=-1（消息不合法） 就停止恢复了。如果 ConsumeQueue 已经把这个消息信息保存到了 CosnumeQueue 里，那么 Consumer 在消费时可能会消费到不合法的消息。

感觉这里有两种作法：

- 把 ConsumeQueue 里的数据删除掉呢（使用标志位进行表示已经删除），然后把这个信息同步给 Slave。这么做的话，处理内容有些多。
- 在 Cosnumer 端加入处理，如果消息不合法就跳过，或者不提供给消费 listener。

#参考

- [rocket mq 底层存储源码分析(6)-存储恢复](https://www.jianshu.com/p/f2201edfaca2)



<br>
<br>
=====================过程===================
看代码过程中的问题：
OK 1，如果启动时，判断是正常关闭还是异常关闭，是通过判断 abort 文件是不是存在决定的。正常关闭的情况下，会删除掉 abort 文件；异常关闭不会删除掉这个文件。

OK 2，说一下 isMappedFileMatchedRecover 方法中，为什么要拿 CheckPoint 中最小的时间戳和“第一条记录的时间戳”比较。

OK 3，恢复时，有调用。。。方法里有判断，如果消息的 commit log offset 比 Index 文件的 endPhyOffset 大的话，就不为些 commit log 创建 Index。endPhyOffset 是每个 Index 文件的 Meta 数据。
`if (msg.getCommitLogOffset() < endPhyOffset)`

阅读中的疑问：
OK 1，save point 在哪保存的？什么时间点？
PASS 2，getMinTimestampIndex 方法中，为什么要`min -= 1000 * 3;`？


关于恢复：
OK 1，ConsumeQueue#recover 中的判断`if (offset >= 0 && size > 0)` 这段代码作用是什么？ 想读取合法的数据吗？如果是的话，如何判断最后的 tagCode 字段是被完整保存了呢？
为什么不使用 check point 中的时间戳，来恢复 ConsumeQueue ？

OK 2，关于 MappedFileQueue#truncateDirtyFiles 中`if (fileTailOffset > offset) `条件的 else 的场合。对于 consume queue 的恢复来说，什么情况下会遇到这种场合？因为 offset 是从 mappedFiles 中计算出来的，不太可能会产生这种情况。（CommitLog 也有相同的问题。CommitLog 是有通过“MagicCode”跳过文件的处理，所以这个“删除掉后面的文件”这个处理可能也是有用的）
在写消息到两个文件时，可能会发生上面的情况。

OK 3，在 CommitLog 恢复时，会调用`this.defaultMessageStore.doDispatch`方法。这个方法会创建 comsume queue，如何防止重复创建？

OK 4，CommitLog 恢复时，如果 size=-1 就停止恢复了。但这样可以吗？当前 offset 之前的数据能保证没有问题吗？= -1 的数据的相关的 consume queue 和 Index 不做什么调整吗？

OK 5，在 Index.load 时候，为什么在`是异常关闭` 并且 `文件的 EndTimestamp > CheckPoint 中 Index 的时间戳`时候，要把整个文件删除掉呢？这个文件前面的数据可能是好的。有没有在其它地方通过什么恢复 Index 文件的处理呢？

OK 6，在写 StoreCheckPoint 时，没人使用安全文件写入方式`MixAll#string2File`，不怕产生数据写入不完整的问题吗？是不是因为数据内容比较小，只会一次全写到磁盘上，或全不写到磁盘上？

