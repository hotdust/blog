#一、基础类介绍
##1，MappedFile
存储用的基础类，每一个文件都对应一个 MappedFile，例如、commitlog 文件、Index 文件等等。

##2，MappedFileQueue
MappedFileQueue 是对 MappedFile 的一个简单的包装，取得和创建 MappedFile 都是在这里做的，刷盘处理也是调用它的 flush 方法，然后 MappedFileQueue 再对 MappedFile 进行刷盘操作。但在实际使用时，有时候是通过 MappedFileQueue 取得 MappedFile 后，使用 MappedFile 进行操作，比如写消息到 buffer 操作。

##3，CommitLog
CommitLog 持有 MappedFileQueue 的引用，使用 MappedFileQueue 和 MappedFile 进行写消息和落盘操作。并且它还持存储的配置（是通过 DefaultMessageStore 引用得到的）

##4，DefaultMessageStore
储存这块的一个大的管理类，它持有存储的配置，并且负责各种服务的启动和停止，例如：IndexService、HAService等待。它在接收到要存储的消息后，调用 CommitLog 去进行具体的存储。

##5，AllocateMappedFileService
这个类主要是用来创建 MappedFile 的。内部使用生产者/消费者模式，putRequestAndReturnMappedFile 方法算是一个生产者，它发送生成 MappedFile 的请求给内部的消费者，并同步等待消费者完成，并把结果返回。mmapOperation 方法是消费方法，用来生成 MappedFile。

putRequestAndReturnMappedFile 每次都发送两个请求，也就是生成两个 MappedFile。第一个 MappedFile 生成成功后就返回，不等待第二个 MappedFile 生成完毕。第二个 MappedFile 目的主要是为了下次再取新 MappedFile 时，能立刻返回给调用者一个 MappedFile，不用等待。




这几个类的包含关系如下：
```
DefaultMessageStore
  -> CommitLog
    -> MappedFileQueue
      -> MappedFile
      -> AllocateMappedFileService
```



#二、启动
broker 的启动是按下面的流程进行的。
```
BrokerStartup#main
  -> start()
    -> createBrokerController()
    -> controller.start()
```
在上面的方法中，重要的是 createBrokerController，下面说一下这个方法。
##1.1 BrokerStartup#createBrokerController
这个方法主要做了以下几件事：

- 启动进的基础 check
- 取得基础服务所需要的参数，从命令行上。例如：name server 地址，broker 端口号等。
- 初始化基础服务，例如：RemotingServer(网络传输服务)，MessageStore（存储服务）等

```
public static BrokerController createBrokerController(String[] args) {
    // 基础 check
    if (null == System.getProperty(NettySystemConfig.COM_ROCKETMQ_REMOTING_SOCKET_SNDBUF_SIZE)) {
        NettySystemConfig.socketSndbufSize = 131072;
    }
    ......


    try {
        // 取得参数
        //PackageConflictDetect.detectFastjson();
        Options options = ServerUtil.buildCommandlineOptions(new Options());
        commandLine = ServerUtil.parseCmdLine("mqbroker", args, buildCommandlineOptions(options),
            new PosixParser());
        if (null == commandLine) {
            System.exit(-1);
        }

        final BrokerConfig brokerConfig = new BrokerConfig();
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        final NettyClientConfig nettyClientConfig = new NettyClientConfig();
        nettyServerConfig.setListenPort(10911);
        final MessageStoreConfig messageStoreConfig = new MessageStoreConfig();

        if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
            int ratio = messageStoreConfig.getAccessMessageInMemoryMaxRatio() - 10;
            messageStoreConfig.setAccessMessageInMemoryMaxRatio(ratio);
        }
        
        // 从命令行上取得参数
        if (commandLine.hasOption('p')) {
            ......
        }

        if (commandLine.hasOption('c')) {
            String file = commandLine.getOptionValue('c');
            if (file != null) {
            ......
            }
        }
        ......

        // 初始化 controller
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }
        ......

        return controller;
    } catch (Throwable e) {
    ......
    }

    return null;
}
```
在初始化 controller 时，也会初始化在 controller 中的基础服务。其中在初始化 RemotingServer 时，会在 RemotingServer 里面生成一些`处理器（processor）`。RemotingServer 在接收到请求后，会根据请求类型调用不同的 processor，来处理这些消息。这其中就有接收消息的 processor `SendMessageProcessor`。
>  大概的技术流程就是，Netty 接收到一个消息的请求后，把消息的处理交给一个线程池，由线程池来处理这些消息。处理消息时，就使用这些 processor 来进行具体的处理。



##1.2 BrokerStartup#start()
准备工作都是在 createBrokerController 里做的，而 start() 方法的作用就是启动那些基础服务，例如：RemotingServer(网络传输服务)，MessageStore（存储服务）等


#三、处理
上面说过，每个请求都有对应的 processor 来处理。下面我们讲一下接收消息的 `SendMessageProcessor`。

接收消息的整个流程如下：
```
SendMessageProcessor.sendMessage()
  -> DefaultMessageStore.putMessage()
    -> CommitLog.putMessage()
      -> MappedFile.appendMessage() // 把消息存储到内存中
      -> 进行刷盘处理，把内存中的消息写到磁盘上。
      -> 把消息同步到 slave server 上。

```

##2.1 SendMessageProcessor#sendMessage
这个方法主要是做一些保存消息前的前期准备工作，内容大概如下：

- 这个地方应该是判断 broker 是否是暂停使用阶段 (此功能好像没有使用)
- 对消息进行 check ，看能否保存消息
- 对消息发送次数进行 check，看是否要放到死信队列里。
- 准备消息内容（主要是 broker 要使用的一些信息）
- 保存消息
- 根据保存结果设置返回值

```
private RemotingCommand sendMessage(final ChannelHandlerContext ctx, //
    final RemotingCommand request, //
    final SendMessageContext sendMessageContext, //
    final SendMessageRequestHeader requestHeader) throws RemotingCommandException {

    final RemotingCommand response = RemotingCommand.createResponseCommand(SendMessageResponseHeader.class);
    final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader) response.readCustomHeader();

    response.setOpaque(request.getOpaque());

    response.addExtField(MessageConst.PROPERTY_MSG_REGION, this.brokerController.getBrokerConfig().getRegionId());
    response.addExtField(MessageConst.PROPERTY_TRACE_SWITCH, String.valueOf(this.brokerController.getBrokerConfig().isTraceOn()));

    if (log.isDebugEnabled()) {
        log.debug("receive SendMessage request command, {}", request);
    }

    final long startTimstamp = this.brokerController.getBrokerConfig().getStartAcceptSendRequestTimeStamp();
    // 这个地方应该是判断 broker 是否是暂停使用阶段，应该是可以 n 秒后恢复的那种。
    // 从代码上看，这个功能好像没有使用。
    if (this.brokerController.getMessageStore().now() < startTimstamp) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark(String.format("broker unable to service, until %s", UtilAll.timeMillisToHumanString2(startTimstamp)));
        return response;
    }

    response.setCode(-1);
    // 对消息能否被接收进行 check
    super.msgCheck(ctx, requestHeader, response);
    if (response.getCode() != -1) {
        return response;
    }

    final byte[] body = request.getBody();
    ......

    // 判断消息发送次数，是否应该保存到死信队列
    String newTopic = requestHeader.getTopic();
    if (null != newTopic && newTopic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
        ......
    }

    // 准备消息
    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(newTopic);
    msgInner.setBody(body);
    msgInner.setFlag(requestHeader.getFlag());
    MessageAccessor.setProperties(msgInner, MessageDecoder.string2messageProperties(requestHeader.getProperties()));
    msgInner.setPropertiesString(requestHeader.getProperties());
    msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(topicConfig.getTopicFilterType(), msgInner.getTags()));

    msgInner.setQueueId(queueIdInt);
    msgInner.setSysFlag(sysFlag);
    msgInner.setBornTimestamp(requestHeader.getBornTimestamp());
    msgInner.setBornHost(ctx.channel().remoteAddress());
    msgInner.setStoreHost(this.getStoreHost());
    msgInner.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());

    // 是否使用了事务
    if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {
        String traFlag = msgInner.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
        if (traFlag != null) {
            response.setCode(ResponseCode.NO_PERMISSION);
            response.setRemark(
                "the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1() + "] sending transaction message is forbidden");
            return response;
        }
    }

    // 保存消息
    PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);

    // 根据保存结果，设置返回值。
    if (putMessageResult != null) {
        boolean sendOK = false;
        ......
    } else {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("store putMessage return null");
    }

    return response;
}
```

下面我们主要看一下保存消息的代码：DefaultMessageStore.putMessage()


###2.1.1 DefaultMessageStore.putMessage()
这个方法的主要内容如下：

- 进一步判断消息能否保存（主要是判断是：否是准备停机状态，是否是 slave等）
- 判断是否是 page cache 繁忙状态
- 写消息


```
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
    if (this.shutdown) {
        log.warn("message store has shutdown, so putMessage is forbidden");
        return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
    }

    if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
        long value = this.printTimes.getAndIncrement();
        if ((value % 50000) == 0) {
            log.warn("message store is slave mode, so putMessage is forbidden ");
        }

        return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
    }

    if (!this.runningFlags.isWriteable()) {
        long value = this.printTimes.getAndIncrement();
        if ((value % 50000) == 0) {
            log.warn("message store is not writeable, so putMessage is forbidden " + this.runningFlags.getFlagBits());
        }

        return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
    } else {
        this.printTimes.set(0);
    }

    if (msg.getTopic().length() > Byte.MAX_VALUE) {
        log.warn("putMessage message topic length too long " + msg.getTopic().length());
        return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);
    }

    if (msg.getPropertiesString() != null && msg.getPropertiesString().length() > Short.MAX_VALUE) {
        log.warn("putMessage message properties length too long " + msg.getPropertiesString().length());
        return new PutMessageResult(PutMessageStatus.PROPERTIES_SIZE_EXCEEDED, null);
    }

    // 判断 OSPageCache 是否是繁忙状态
    if (this.isOSPageCacheBusy()) {
        return new PutMessageResult(PutMessageStatus.OS_PAGECACHE_BUSY, null);
    }

    long beginTime = this.getSystemClock().now();
    PutMessageResult result = this.commitLog.putMessage(msg);

    long eclipseTime = this.getSystemClock().now() - beginTime;
    if (eclipseTime > 500) {
        log.warn("putMessage not in lock eclipse time(ms)={}, bodyLength={}", eclipseTime, msg.getBody().length);
    }
    this.storeStatsService.setPutMessageEntireTimeMax(eclipseTime);

    if (null == result || !result.isOk()) {
        this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
    }

    return result;
}
```

其中，判断 OSPageCache 是否是繁忙状态这里，进行细说一下。
在每次写消息到 buffer 之前，会记录一个开始时间，在写完消息到 buffer 之后，就会把这个开始时间清 0。在本次写消息到 buffer 之前，会取得`上次写消息时设置的开始时间`，如果`当前系统时间 - 上次写消息设置的开始时间`在一个`指定区间内`的话，就说明写 page cache 繁忙。还有两点要注意一下：

- 因为`写消息到 buffer`处理是一个串行处理（使用阻塞或非阻塞锁保证的串行），所以当取得的`上次写 buffer 前的开始时间`不为 0 的话，则表示上次写 buffer 处理还没有完成（或者也可能是`上 n 次写 buffer 处理`没有完成）。如果这个时间很大的话，就表示 写 page cache 繁忙。如果上次写 buffer 处理已经完成，`上次写消息设置的开始时间`为 0 的话，那么`当前系统时间 - 上次写消息设置的开始时间`就不在`指定区间内`，就算是不繁忙。
- 为什么写 buffer 很长时间没有完成，就表示写 page cache 繁忙呢？因为写处理使用的是 mmap 方式，这种方式减少了数据复制到`用户空间（也就是 jvm）`这个步骤，但是减少不了数据写内核内存的步骤（也就是到 page cache 的步骤，因为 page cache 就是 mmap 使用的那些内核内存）。page cache 满了的话，存在交换和清理的操作；如果没有满，可能存在 page fault 的操作，这些都可能影响 page cache 写的性能。另外，写 buffer 只是向内存里写，还没有向磁盘上写，所以这里不涉及磁盘性能影响的问题。

```
/**
 * 消息在写入 buffer 的时间（不算刷盘时间），大于指定值的话，就算 busy
 * @return
 */
@Override
public boolean isOSPageCacheBusy() {
    long begin = this.getCommitLog().getBeginTimeInLock();
    long diff = this.systemClock.now() - begin;

    if (diff < 10000000 //
        && diff > this.messageStoreConfig.getOsPageCacheBusyTimeOutMills()) {
        return true;
    }

    return false;
}
```


###2.1.1.1 CommitLog.putMessage()
这个方法是个很重要的方法，写 buffer、刷盘、slave 同步处理，这几个重要处理都是在这个方法里面做的。每个处理都可以讲出一些内容，下面让我们看看都具体有什么样的处理。

- 使用锁机制，保证只有一个线程可以进行写 buffer 操作（使用重入锁 或 自旋锁（非阻塞锁））。
- 进行写 buffer
- 解除上面写的锁
- 进行刷盘处理
- 将数据同步到 slave

```
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {

    StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

    String topic = msg.getTopic();
    int queueId = msg.getQueueId();

    long eclipseTimeInLock = 0;
    MappedFile unlockMappedFile = null;
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

    // 使用重入锁 或 自旋锁（非阻塞锁）控制并发处理，保证只有一个线程可以进行写操作。
    lockForPutMessage(); //spin...
    try {
        long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
        this.beginTimeInLock = beginLockTimestamp;

        // Here settings are stored timestamp, in order to ensure an orderly
        // global
        msg.setStoreTimestamp(beginLockTimestamp);

        // 如果这里需要判断，那上面在 try 外面取得 MappedFile 处理，拿到这里可读性是不是更好。
        if (null == mappedFile || mappedFile.isFull()) {
            mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
        }
        if (null == mappedFile) {
            log.error("create maped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
        }

        // 写消息到 buffer 里
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        switch (result.getStatus()) {
            case PUT_OK:
                break;
            case END_OF_FILE:
                // 如果 mappedFile 剩余空间不够保存 message，就生成一个新文件，来保存
                unlockMappedFile = mappedFile;
                // Create a new file, re-write the message
                mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                if (null == mappedFile) {
                    // XXX: warn and notify me
                    log.error("create maped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
                }
                result = mappedFile.appendMessage(msg, this.appendMessageCallback);
                break;
            case MESSAGE_SIZE_EXCEEDED:
            case PROPERTIES_SIZE_EXCEEDED:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
            case UNKNOWN_ERROR:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
            default:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
        }

        eclipseTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
        beginTimeInLock = 0;
    } finally {
        releasePutMessageLock();
    }

    if (eclipseTimeInLock > 500) {
        log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", eclipseTimeInLock, msg.getBody().length, result);
    }

    if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
        // TODO Q: 18/5/10 这个是干什么用的？
        this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
    }

    PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

    GroupCommitRequest request = null;

    // 进行刷盘
    // Synchronization flush
    if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
        final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
        // 如果消息类型是"批量同步刷盘"消息
        if (msg.isWaitStoreMsgOK()) {
            request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
            // 向同步刷盘线程发送要落盘的消息
            service.putRequest(request);
            // 等待消息落盘完成
            boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
            if (!flushOK) {
                log.error("do groupcommit, wait for flush failed, topic: " + msg.getTopic() + " tags: " + msg.getTags()
                    + " client address: " + msg.getBornHostString());
                putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
            }
        } else { // 消息类型是"单条刷盘"
            service.wakeup();
        }
    }
    // Asynchronous flush
    else {
        // TODO Q: 2018/5/11 为什么异步刷盘时，要判断是不是 isTransientStorePoolEnable，根据这个来判断选用启动哪个 service 呢？
        if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
            flushCommitLogService.wakeup();
        } else {
            commitLogService.wakeup();
        }
    }

    // 数据同步到 slave 
    // Synchronous write double
    if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
        HAService service = this.defaultMessageStore.getHaService();
        if (msg.isWaitStoreMsgOK()) {
            // Determine whether to wait
            if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
                if (null == request) {
                    request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                }
                service.putRequest(request);

                service.getWaitNotifyObject().wakeupAll();

                boolean flushOK =
                    // TODO
                    request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                if (!flushOK) {
                    log.error("do sync transfer other node, wait return, but failed, topic: " + msg.getTopic() + " tags: "
                        + msg.getTags() + " client address: " + msg.getBornHostString());
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
                }
            }
            // Slave problem
            else {
                // Tell the producer, slave not available
                putMessageResult.setPutMessageStatus(PutMessageStatus.SLAVE_NOT_AVAILABLE);
            }
        }
    }

    return putMessageResult;
}
```

#### 2.1.1.1.1 CommitLog.lockForPutMessage()
这个方法是保证线程同步的方法，允许只有一个线程能进行后面的操作。我们先看看代码：
```
private void lockForPutMessage() {
    if (this.defaultMessageStore.getMessageStoreConfig().isUseReentrantLockWhenPutMessage()) {
        putMessageNormalLock.lock();
    } else {
        boolean flag;
        do {
            flag = this.putMessageSpinLock.compareAndSet(true, false);
        }
        while (!flag);
    }
}
```
同步锁的机制使用两种方式实现：

- Reentrantlock(也就是代码中的 putMessageNormalLock)
- 自旋锁

Reentrantlock 资料很多，看一下就可以明白。下面说一下自旋锁的实现。
```
boolean flag;
do {
    flag = this.putMessageSpinLock.compareAndSet(true, false);
}
while (!flag);
```
主要就是使用一个 AtomicBoolean 变量和一个 do while 循环，锁不上就一直循环。这种自旋锁是非公平模式的，无法控制先到的等待线程，在锁释放后就一般会先得到锁。（上面的 Reentrantlock 也是使用的非公平模式）

为什么要使用自旋锁呢？
这里的自旋锁其实是一种“非阻塞锁”。独占锁(Reentrantlock)在竞争不是很激烈时候，效率不如非阻塞锁。而在竞争非常激烈的时候，还是独占锁效率高。因为在竞争激烈的时候，非阻塞锁可能会经常失败。


####2.1.1.1.2 MappedFile#appendMessage

```
public AppendMessageResult appendMessage(final MessageExtBrokerInner msg, final AppendMessageCallback cb) {
    assert msg != null;
    assert cb != null;

    int currentPos = this.wrotePosition.get();

    if (currentPos < this.fileSize) {
        // 取得 file 中，剩余部分的 buffer
        ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
        // 为什么要设置 currentPos？ slice()后的 buffer 是从剩余部分开始的，position 前的部分都不包含呀。
        // 因为没有单独使用过 上面两个 buffer 单独进行过 put 操作，都是 slice 后的引用进行 put，
        // 所以，position 一起都是 0。对于 position，是使用手动进行记录 position 位置的：wrotePosition。
        byteBuffer.position(currentPos);
        // 在回调方法里，把消息真正的写入 buffer 中
        AppendMessageResult result =
            cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, msg);
        // 使用 message 的字符数，更新 wrotePosition，记录下次开始写的位置。
        this.wrotePosition.addAndGet(result.getWroteBytes());
        this.storeTimestamp = result.getStoreTimestamp();
        return result;
    }

    log.error("MappedFile.appendMessage return null, wrotePosition: " + currentPos + " fileSize: "
        + this.fileSize);
    return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}
```
在写消息时，首先是向 ByteBuffer 里写消息。但向哪个 ByteBuffer 里写消息，需要判断一下。这里有两个 ByteBuffer：一个是通过 MMAP 取得的 ByteBuffer(mappedByteBuffer) ，还有一个是 new 出来的 ByteBuffer(writeBuffer)。

那是使用 writeBuffer 还是 mappedByteBuffer 是由什么决定的呢？是由一个方法决定的：isTransientStorePoolEnable。这个方法在 AllocateMappedFileService#mmapOperation 中被使用，如果为 true ，在生成 MappedFile 时，就会初始化 writeBuffer；如果为 false，就会初始化 writeBuffer 和 mappedByteBuffer（感觉不应该初始化 mappedByteBuffer）。虽然为 false 时两个都初始化了，但在使用时判断为`writeBuffer != null ? writeBuffer : mappedByteBuffer`，所以如果 writeBuffer 被初始化了的话，就优先使用它。
```
public boolean isTransientStorePoolEnable() {
        return transientStorePoolEnable && FlushDiskType.ASYNC_FLUSH == getFlushDiskType()
            && BrokerRole.SLAVE != getBrokerRole();
}
```

下面总结一下 writeBuffer 和 mappedByteBuffer 使用的场景：

- mappedByteBuffer：设置为同步处理时（FlushDiskType.SYNC_FLUSH)，一定会使用 mappedByteBuffer。也一定会使用同步刷盘(GroupCommitService) ，对 mappedByteBuffer 进行刷盘。设置异步处理（FlushDiskType.ASYNC_FLUSH)时，如果 transientStorePoolEnable 为 false，那么也会使用 mappedByteBuffer，异步刷盘(FlushRealTimeService)也会对 mappedByteBuffer 进行刷盘。
- writeBuffer：设置异步处理（FlushDiskType.ASYNC_FLUSH)时，如果 transientStorePoolEnable 为 true 时，会使用 writeBuffer。异步刷盘类(CommitRealTimeService)会对这个 writeBuffer 进行刷盘。



**为什么要写到 ByteBuffer 里？ByteBuffer 是什么呢？**
这里要就先讲讲写处理的基本实现。在 RocketMQ 写消息时，有两种方式：
(注意：下面说到 buffer 时，是指由 writeBuffer 或是 mappedByteBuffer 总产生的 ByteBuffer)

- mmap
- DirectBuffer + FileChannel

1，mmap
当不使用 pool（transientStorePoolEnable == false）时，使用的是 mmap 方式。这种方式生成的 buffer 的引用，就是`文件所在的内核内存地址`。也就是说，当进行刷盘处理时，会直接把这个 buffer 所指向的内存刷到磁盘上，不需要其它内存拷贝处理。代码里写的 buffer 名为`mappedByteBuffer`。
```
MappedByteBuffer mbb = fileChannel.getMappedByteBuffer();
mbb.put(message);
```

2，DirectBuffer + FileChannel
当使用 pool（transientStorePoolEnable == true）时，使用的是这种方式。它是先申请一个文件大小相同的`DirectBuffer`(堆外内存)，然后在写消息时，向这个 DirectBuffer 里面写。然后隔一段时间，通过 channel 把 DirectBuffer 的数据写到磁盘上。

**位置变量**
这两种方式简单看来，都是对一大堆内存进行写。在写的时候，使用位置变量来记录“已经写到哪个位置”了，使用的变量就是`wrotePosition`。每次写完消息后，都对把消息大小加到这个位置变量上。



####2.1.1.1.2.1 CommitLog#doAppend()
这个方法是传给`MappedFile#appendMessage`的一个回调方法，在这个方法里对消息进行进一步的组装，然后使用方法的参数中的 buffer，把消息写入 buffer 中。

这个方法中有一点需要注意，有可能 buffer 的剩余空间不够写入一个消息。这种情况的话，会使用写一条和剩余空间大小相同的空白消息（通过 Magic Code 标识消息类型），把空间填满。然后在返回结果中，标识出这次操作的结果是`END_OF_FILE`。在最外层的方法中，如果识别到结果是`END_OF_FILE`，会再进行一次保存处理，然后就会把消息保存到新的文件中。

```
class DefaultAppendMessageCallback implements AppendMessageCallback {



    DefaultAppendMessageCallback(final int size) {
        this.msgIdMemory = ByteBuffer.allocate(MessageDecoder.MSG_ID_LENGTH);
        this.msgStoreItemMemory = ByteBuffer.allocate(size + END_FILE_MIN_BLANK_LENGTH);
        this.maxMessageSize = size;
    }


    public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank, final MessageExtBrokerInner msgInner) {
        // STORETIMESTAMP + STOREHOSTADDRESS + OFFSET <br>

        // PHY OFFSET
        long wroteOffset = fileFromOffset + byteBuffer.position();

        this.resetByteBuffer(hostHolder, 8);
        String msgId = MessageDecoder.createMessageId(this.msgIdMemory, msgInner.getStoreHostBytes(hostHolder), wroteOffset);

        /**
         * Serialize message
         */
        final byte[] propertiesData =
            msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);

        final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;

        // 如果“消息长度+空白消息长度 > 剩余空间”，说明保存当前消息后，
        // 剩余空间不够保存一个空白消息体，所以把当前剩余空间都保存成空白。
        // 然后把当前消息保存到下一个文件中。
        // Determines whether there is sufficient free space
        if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
            this.resetByteBuffer(this.msgStoreItemMemory, maxBlank);
            // 1 TOTALSIZE
            this.msgStoreItemMemory.putInt(maxBlank);
            // 2 MAGICCODE
            // TODO Q: 18/5/10 这个 BLANK_MAGIC_CODE 是干什么用的？
            this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
            // 3 The remaining space may be any value
            //

            // Here the length of the specially set maxBlank
            final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
            byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);
            return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset, maxBlank, msgId, msgInner.getStoreTimestamp(),
                queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
        }

        // Initialization of storage space
        this.resetByteBuffer(msgStoreItemMemory, msgLen);
        // 1 TOTALSIZE
        this.msgStoreItemMemory.putInt(msgLen);
        ......
        final long beginTimeMills = CommitLog.this.defaultMessageStore.now();

        // 把消息写进 buffer
        // Write messages to the queue buffer
        byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);

        AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgId,
            msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);


        return result;
    }
}
```

####刷盘处理
**刷盘处理有两种方式：**

- 同步刷盘
- 异步刷盘

**刷盘类有 3 个：**

- GroupCommitService：同步刷盘类（FlushDiskType 为 true），对 mappedByteBuffer 里的内容进行刷盘。
- FlushRealTimeService：异步刷盘类（FlushDiskType 为 false），对 mappedByteBuffer 或 writeBuffer 进行刷盘。当 transientStorePoolEnable 为 false 时候，对 mappedByteBuffer 里的内容进行刷盘；为 true 时，对 writeBuffer 进行刷盘。
- CommitRealTimeService：异步刷盘类（FlushDiskType 为 false），把 writeBuffer 里的数据写到 fileChannel 中，然后启动 FlushRealTimeService，使用 flush 方法把 fileChannel 中的数据 force 到磁盘上。

**调用方式如下：**

同步刷盘（FlushDiskType == true）：

- 使用 mappedByteBuffer 来保存消息
- 使用 GroupCommitService 进行把 mappedByteBuffer 里的数据“同步”刷到磁盘上。

异步刷盘（FlushDiskType == false）：（异步刷盘分成两种，根据 transientStorePoolEnable 进行区分。）

- transientStorePoolEnable 为 false：
 - 使用 mappedByteBuffer 来保存消息
 - 使用 FlushRealTimeService 进行把 mappedByteBuffer 里的数据“异步”刷到磁盘上。
- transientStorePoolEnable 为 true：
 - 使用 writeBuffer 来保存消息
 - 使用 CommitRealTimeService 把 writeBuffer 里的数据“异步”写进 fileChannel 中，再把 fileChannel 里的数据刷到磁盘上。

| FlushDiskType(条件1) | transientStore(条件2) | 使用的刷盘类 | 刷盘Buffer |
|----------|--:|:--:|---|
| true | false | GroupCommitService | mappedByteBuffer |
| false | false | FlushRealTimeService | mappedByteBuffer |
| false | true | FlushRealTimeService | writeBuffer |
| false | true | CommitRealTimeService | writeBuffer |

也就是说：

- 当为同步刷盘时（FlushDiskType == true），GroupCommitService 自己就能进行整个刷盘操作。
- 当为异步刷盘（FlushDiskType == false），并且不使用 Pool（transientStorePoolEnable == false）时，FlushRealTimeService 自己也能搞定刷盘操作。
- 当为异步刷盘（FlushDiskType == false），并且使用 Pool（transientStorePoolEnable == true）时，需要 CommitRealTimeService 和 FlushRealTimeService 一起来搞定刷盘操作。




下面具体来说一下“同步刷盘”和“异步刷盘”。

<br>
**1，同步刷盘**
GroupCommitService 这个类是同步刷盘类，这个类是对 也是对 mappedByteBuffer 里的内容进行刷盘操作。同步刷盘整体结构是一个“生成者/消费者”结构。生产者和消费者使用一个`同步队列`传递内容。

- 生成者：CommitLog 把接收到的消息放到同步队列中。
- 消费者：同步刷盘线程，这个线程在 broker 启动时就启动了。这个线程接收队列中的消息，进行刷盘。

那是不是每条消息都进行刷盘呢？
不一定，根据消息类型来决定。如果 producer 在发消息时，设置了 WAIT 属性为 true 的话，就会每条都进行同步刷盘；如果没有设置的话，默认每 10ms 进行一次同步刷盘（这种也属于批量形式）。


下面我们看看同步刷盘生产者的代码：

```
if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
    final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
    // 如果消息类型是"批量同步刷盘"消息
    if (msg.isWaitStoreMsgOK()) {
        request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
        // 向同步刷盘线程发送要落盘的消息
        service.putRequest(request);
        // 等待消息落盘完成
        boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
        if (!flushOK) {
            log.error("do groupcommit, wait for flush failed, topic: " + msg.getTopic() + " tags: " + msg.getTags()
                + " client address: " + msg.getBornHostString());
            putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
        }
    } else { // 消息类型是"单条刷盘"
        service.wakeup();
    }
}
```
- 首先把消息包装成一个同步刷盘用的 request
- 然后把这个 request 入到同步队列中
- 最后 request 进入等待状态，等待刷盘服务完成后通知它

接下来我们看看消费者的处理：
```
public void run() {
    CommitLog.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            this.waitForRunning(10);
            this.doCommit();
        } catch (Exception e) {
            CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
        }
    }

    // Under normal circumstances shutdown, wait for the arrival of the
    // request, and then flush
    try {
        Thread.sleep(10);
    } catch (InterruptedException e) {
        CommitLog.log.warn("GroupCommitService Exception, ", e);
    }

    synchronized (this) {
        this.swapRequests();
    }

    this.doCommit();

    CommitLog.log.info(this.getServiceName() + " service end");
}
```
　
同步刷盘的消费进程方面还是有一些学习的地方：
> 同步刷盘使用两个队列来处理要刷盘的消息：requestWrite (接收要落盘的消息) 和 requestRead (把消息落盘时，从从这个队列里读取消息)。有一个队列在刷盘前一起接收消息，在要进行刷盘前，把两个队列的引用交换，这时 requestRead 指向的就是那些要落盘的消息，而 requestWrite 指向一个空的队列，这个空队列负责在刷盘期间继续接收要落盘的消息。
（这里的消息，不是指具体的消息内容，因为消息已经写到 buffer 里了。这里的消息，是指`消息的字节长度`和`消息写到的位置`）

下面让我们分析一下代码，首先是`waitForRunning(10)`，这个方法是做什么的呢？

- 如果没有消息进来时，防止 GroupCommitService 不停地空转浪费 CPU，所以让此线程暂停等待。当有消息进来时，就会立刻唤醒 GroupCommitService。
- 如果没有消息准备落盘，就等待 10ms，然后再交换两个队列。（即使没有消息进来，也要交换两个队列。只是在刷盘时候，看队列里没有消息就直接返回了）

```
protected void waitForRunning(long interval) {
    // 如果有消息进来
    if (hasNotified.compareAndSet(true, false)) {
        // 交换处理队列
        this.onWaitEnd();
        // 直接返回，进行后续操作，就不进行等待一段时间了。
        return;
    }

    // reset方法是为了恢复初始化状态。
    // 因为原始的 CountDownLatch 在使用一次后就不能使用了，为了能重复使用，新加了一个 reset 方法。
    // 把状态设置成初始状态，这样 await 又可以使用了。
    //entry to wait
    waitPoint.reset();

    try {
        waitPoint.await(interval, TimeUnit.MILLISECONDS);
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        // 超过等待时间后，恢复 hasNotified 状态为没有消息进入状态。
        hasNotified.set(false);
        // 交换处理队列
        this.onWaitEnd();
    }
}

// ----------- 下面的两个方法，是在 GroupCommitService 类中
@Override
protected void onWaitEnd() {
    this.swapRequests();
}

// 使用两个队列进行处理消息。
// 在等待时，使用 requestsWrite 接收消息。当可以处理消息刷盘后，就把 requestsWrite 和 requestsRead 引用互换，刷盘处理从 requestsRead 里读取要刷盘的消息，进行刷盘。刷盘时，如果有消息进来，就向 requestsWrite 里写消息。等再到可以刷盘时，再对新进来的消息进行刷盘。
private void swapRequests() {
    List<GroupCommitRequest> tmp = this.requestsWrite;
    this.requestsWrite = this.requestsRead;
    this.requestsRead = tmp;
}
```

`if (hasNotified.compareAndSet(true, false))`是判断是否有消息进来的，hasNotified 是何进被设置的呢？就是消息被放进队列里时设置的：
```
public void putRequest(final GroupCommitRequest request) {
    synchronized (this) {
        this.requestsWrite.add(request);
        if (hasNotified.compareAndSet(false, true)) {
            waitPoint.countDown(); // notify
        }
    }
}
```

`waitPoint.reset()` 是什么意思呢？
waitPoint 是自定义的一个类`CountDownLatch2`。这个类和`CountDownLatch`区别在哪呢？区别就在于`CountDownLatch2`是一个可重复使用的类，像 CyclicBarrier 一样，可重复使用。如何重复使用呢？就在于它新方法`reset`上面，调用`reset`方法后，计数器又回到了初始化时的状态。

<br>
`waitForRunning`方法调用完后，调用的就是`doCommit`方法了，这个方法就是把 buffer 里的内容写到磁盘上的地方了。

```
private void doCommit() {
    if (!this.requestsRead.isEmpty()) {
        for (GroupCommitRequest req : this.requestsRead) {
            // There may be a message in the next file, so a maximum of
            // two times the flush
            boolean flushOK = false;
            // 这里为什么最多执行两次呢？
            // 因为如果消息大小超过 mappedFile 剩余空间时，会对两个 mappedFile 进行写入。执行一次 mappedFileQueue.flush 方法的话，只能对老的 mappedFile 写入，新的写不了。再执行一次的话，就会对新的 mappedFile 进行写入了，这个是在 MappedFileQueue#findMappedFileByOffset 方法中进行选择的。
            for (int i = 0; i < 2 && !flushOK; i++) {
                // 判断 mappedFileQueue 中文件刷完盘的位置(FlushedWhere) >= 消息的最后位置的话，说明这个消息已经落盘了（因为消息是多次写到 buffer 中的，但一次刷盘就可以把消息都落盘）
                flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();

                // 如果有消息还没有落盘，就进行刷盘
                if (!flushOK) {
                    CommitLog.this.mappedFileQueue.flush(0);
                }
            }

            // 刷新完了，告诉等待线程可以继续。（同步刷盘在向 list 里放入要落盘消息后，就进入等待）
            req.wakeupCustomer(flushOK);
        }

        long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
        if (storeTimestamp > 0) {
            CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
        }

        this.requestsRead.clear();
    } else {
        // Because of individual messages is set to not sync flush, it
        // will come to this process
        CommitLog.this.mappedFileQueue.flush(0);
    }
}
```
这里有一点要注意的地方：刷盘时，最多进行两次刷盘，为什么要这么做呢？
因为如果消息大小超过 mappedFile 剩余空间时，会对两个 mappedFile 进行写入。执行一次 mappedFileQueue.flush 方法的话，只能对老的 mappedFile 写入，新的写不了。再执行一次的话，就会对新的 mappedFile 进行写入了，这个是在 MappedFileQueue#findMappedFileByOffset 方法中进行选择的。

其它没有什么特别的地方了，让我们再进一步，进入具体的刷盘方法里面：`CommitLog.this.mappedFileQueue.flush(0)`
```
public boolean flush(final int flushLeastPages) {
    boolean result = true;
    // 取得应该写入的 mappedFile
    MappedFile mappedFile = this.findMappedFileByOffset(this.flushedWhere, false);
    if (mappedFile != null) {
        long tmpTimeStamp = mappedFile.getStoreTimestamp();
        // mappedFile 进行刷盘
        int offset = mappedFile.flush(flushLeastPages);
        long where = mappedFile.getFileFromOffset() + offset;
        result = where == this.flushedWhere;
        this.flushedWhere = where;
        if (0 == flushLeastPages) {
            this.storeTimestamp = tmpTimeStamp;
        }
    }

    return result;
}
```
在这个刷盘方法里，首先要找到要刷新哪个 MappedFile，所以要先看一下`findMappedFileByOffset`方法。
```
public MappedFile findMappedFileByOffset(final long offset, final boolean returnFirstOnNotFound) {
    try {
        // 取得 mappedFiles 里第一个文件，也就是文件名最小的文件。
        MappedFile mappedFile = this.getFirstMappedFile();
        if (mappedFile != null) {
            int index = (int) ((offset / this.mappedFileSize) - (mappedFile.getFileFromOffset() / this.mappedFileSize));
            if (index < 0 || index >= this.mappedFiles.size()) {
                LOG_ERROR.warn("Offset for {} not matched. Request offset: {}, index: {}, " +
                        "mappedFileSize: {}, mappedFiles count: {}",
                    mappedFile,
                    offset,
                    index,
                    this.mappedFileSize,
                    this.mappedFiles.size());
            }

            try {
                return this.mappedFiles.get(index);
            } catch (Exception e) {
                if (returnFirstOnNotFound) {
                    return mappedFile;
                }
                LOG_ERROR.warn("findMappedFileByOffset failure. ", e);
            }
        }
    } catch (Exception e) {
        log.error("findMappedFileByOffset Exception", e);
    }

    return null;
}
```
这个方法的计算过程如下：
假设：
现在有两个文件：00000000000000000000、00000000000000524288。
offset: 734297 （这个 offset 是：第二个文件名(524288) + 已经写入的消息位置(210009)）
mappedFileSize: 524288（每个文件的大小）
mappedFile.getFileFromOffset(): 0 （因为第一个文件名就是 0）

那么：
(offset / this.mappedFileSize) = 1
(mappedFile.getFileFromOffset() / this.mappedFileSize) = 0
index = 1 - 0 = 1


但是，如果写消息到 buffer 时，buffer 剩余空间不够写，那么就会把当前 buffer 用空白填满，然后，新创建一个 buffer 来保存消息。在这种情况下，此方法取得的 mappedFile 是第用空白填满的那个。那个 mappedFile 落盘后，getFileFromOffset就变成新的 buffer 的了，这个值一定比 mappedFileSize 小，所以取得的 mappedFile 是最新的那个。


取得 MappedFile 后，就要进行对 MappedFile 进行刷盘处理了，即`MappedFile.flush`方法。
这个方法主要是调用 force 方法，强制把 buffer 里的内容写到磁盘上。但要注意一点，这里更新了`flushedPosition`（用 wrotePosition 或 committedPosition 更新 flushedPosition，让 flushedPosition 和  wrotePosition 或 committedPosition 保持一致）。这个变量是用来记录落盘的位置的，这个变量需要作为方法的返回值，用来设置 doCommit 方法的 flushedWhere 变量的。

感觉有一点奇怪的地方是，如果`hold()`返回 false，就无法刷盘磁盘，但也把 flushedPosition 更新到和 buffer 写位置相同的状态，意思是就是刷新成功，同步消息也会被通知成功。但其实没有刷盘成功，这样同步消息是不是也存在丢失消息的问题呢？
```
/**
 * @param flushLeastPages 刷盘时的页数限制。大于这个页数的话，就可以进行刷盘。
 * @return The current flushed position
 */
public int flush(final int flushLeastPages) {
    // 首先判断是否可以刷盘
    if (this.isAbleToFlush(flushLeastPages)) {
        // 判断此 MappedFile 是否还可以使用。通过引用计数的方式来确认是否可以使用。
        if (this.hold()) {
            int value = getReadPosition();

            try {
                //We only append data to fileChannel or mappedByteBuffer, never both.
                // 异步刷盘
                if (writeBuffer != null || this.fileChannel.position() != 0) {
                    this.fileChannel.force(false);
                } else {
                    // 同步刷盘
                    this.mappedByteBuffer.force();
                }
            } catch (Throwable e) {
                log.error("Error occurred when force data to disk.", e);
            }

            // 设置 flushedPosition。（用 wrotePosition 或 committedPosition 更新 flushedPosition。）
            this.flushedPosition.set(value);
            // 释放引用
            this.release();
        } else {
            log.warn("in flush, hold failed, flush offset = " + this.flushedPosition.get());
            this.flushedPosition.set(getReadPosition());
        }
    }
    return this.getFlushedPosition();
}
```
这个方法里还有一些设计，不知道大家注意没注意到`this.hold()`和`this.release()`方法。这两个方法是`ReferenceResource`类里的方法 ，这个类是做什么用的呢？
ReferenceResource 是一个引用计数类，表示当前类“有几个地方正在使用”、或者当前类“能否使用”。“有几个地方正在使用”和“能否使用”是什么意思呢？

- 有几个地方正在使用：使用一个计数器，有一个地方使用，计数器就加 1。使用完后，计数器减 1。加 1 减 1 就是使用的 hold() 和 release() 方法。
- 能否使用：使用一个布尔型变量，来表示能否使用。

从使用上来看，这个类的作用应该是保证在 broker 进行 shutdown 时候，停止刷盘工作。还有就是在关闭和删除 MappedFile 时候使用。

下面是文章 [rocket mq 底层存储源码分析(1)-存储总概](https://www.jianshu.com/p/f01534f21b71) 对 ReferenceResource 的解释：
>
这里在说一下MappedByteBuffer，可能会导致JVM crash ,因为MappedByteBuffer可以通过特殊的方法释放，实际上调用了unmap的方法。此时，之前映射到jvm的地址空间就是非法地址，如果此后仍然对MappedByteBuffer进行读写，系统就会向jvm发送sigbus信号来通知进程非法操作，这个问题一般是由于程序没有处理好并发问题导致的。
>
因此rmq通过引用计数法，即只要引用计数不为0,MappedByteBuffer对象就不会释放来解决这个问题。具体的抽象实现为ReferenceResource，使用AtomicLong原子变量来保证并发，性能上会比较好。
>
当然，这种方式也会存在弊端，就是程序不能正确操作引用计数，可能会导致文件无法删除，因此，rmq增加了一个补救措施，就是一旦文件被关闭了状态位available会设置为false，并且开始计时，如果超过2分钟，引用计数还没有变为0，就强行释放。上文提到，MappedByteBuffer会在jvm发生gc时，可能被回收，但不是一定，rmq通过反射的方式调用Cleaner.clean，手动清除。
>
DirectByteBuffer本身是一个java heap内的对象，自身所占用的内存并不会很大，只是其实例所映射的堆外内存可能会比较大，当jvm发起young gc时，如果DirectByteBuffer实例是非可达性对象，那么，jvm就会将DirectByteBuffer实例回收，在回收前，会通过Cleaner.clean方法，委托Deallocator释放堆外内存；但DirectByteBuffer经过多次ygc后，会晋升到老年代，此时，如果不通过full gc 或old gc,就无法释放堆外内存；因此我们可以通过程序手动释放。
>
Cleaner是 PhantomReference虚引用的子类，并通过自身的next和prev字段维护的一个双向链表。PhantomReference的作用在于跟踪垃圾回收过程，并不会对对象的垃圾回收过程造成任何的影响。
所以cleaner = Cleaner.create(this, new Deallocator(base, size, cap)); 用于对当前构造的DirectByteBuffer对象的垃圾回收过程进行跟踪。
当DirectByteBuffer对象从pending状态 ——> enqueue状态时，会触发Cleaner的clean()，而Cleaner的clean()的方法会实现通过unsafe对堆外内存的释放。


<br>
**2，异步刷盘**
**FlushRealTimeService**
FlushRealTimeService 是一个异步刷盘类，也是对 mappedByteBuffer 或 writeBuffer 里的内容进行刷盘操作。当 transientStorePoolEnable 为 false 时候，对 mappedByteBuffer 里的内容进行刷盘；为 true 时，对 writeBuffer 进行刷盘。这个异步刷盘类主要逻辑就是：

- 如果是“定时”异步刷盘，就等待一定时间后，进行刷盘。等待时，不可中断（这个中断是指 RocketMQ 实现的一种自定义的中断，不是指线程中断）
- 如果不是“定时”异步刷盘，也等待一定时间后刷盘，但这个等待是可以中断的。而中断的调用，就是在消息写完 buffer 后，调用异步刷盘服务的 wakeup 方法后。

这个异步刷盘类，也是对 mappedByteBuffer 里的内容进行刷盘操作。


**CommitRealTimeService**
CommitRealTimeService 这个异步刷盘类，这个类是对 writeBuffer 里的内容进行刷盘操作。这个刷盘类处理有点不一样，在这个类中把 writerBuffer 的数据写到 fileChannel 中，但刷盘处理，是调用的 FlushRealTimeService 进行的刷盘。

- 首先把 writeBuffer 里的内容写到 fileChannel 里
- 然后调用 FlushRealTimeService 进行异步刷盘
- 最后等待一定时间（这个等待也是可中断的）

<br>
CommitRealTimeService 把消息写到 fileChannel 上，也有一定条件（就像 GroupCommitService 每 10ms 写一次）。具体条件如下：
- 如果写的消息大于 4 个系统页大小。（每个系统页大小，默认为 4096 Byte）。
- 如果距离上次写操作，已经过去 200ms。

每个消息来到时，都会 wakeup CommitRealTimeService。如果符合上面的条件，就进行写消息；如果不符合，CommitRealTimeService 就再次进行等待状态。



####将数据同步到 slave
暂时省略

<br>
##其它细节
最后再来说说其中没有介绍到方法中的一些细节。

###1，MappedFileQueue#putRequestAndReturnMappedFile
这个方法是用来生成 MappedFile 的，每次生成两个 mappedFile。第一个 MappedFile 创建成功后就返回，第二个在异步线程里创建。这样不太影响主要的调用，还能为下一次使用 MappedFile 节省一些创建时间。

###2，在 MappedFile 类中，有一个warmMappedFile 方法，这个类的作用是对生成的文件进行“预热”。“预热”都有什么动作呢？为什么要这么做呢？

**“预热”动作(1)：把 MappedFile 写满**
不管是 DirectBuffer 方式还是 MMAP 方式，在`第一次`写数据时都要产生 page fault 来建立`虚拟内存`和`物理内存`映射关系。第二次写时，因为映射已经创建好了，所以就不会产生 page fault 了（前提是内存够大，不需要因为内存紧张把 page cache 移除）。而第二次写，也就是真正储存消息时，进行的写操作。
> 说一下 page fault。CPU 所作的一切运算，都是通过 CPU 缓存间接与内存进行操作的。若是 CPU 请求的内存数据在物理内存中不存在，那么 CPU 就会报告「缺页错误（Page Fault）」。发生 page fault 后，就会把数据读到内存中，然后建立映射，让 CPU 能正常访问数据。所以不管是 DirectBuffer 方式还是 MMAP 方式，在第一次进行写数据时，都会发生 page fault。

~~**还有一段处理，在 MMAP 时候如果写满后，进行刷盘。为什么 MMAP 时候进行刷盘，而 DirectBuffer+Channel 时候不进行刷盘呢？**
这里的猜测是，因为 MMAP 在写完数据后不进行刷盘的话，在一定条件下 linux 的 pdflush 进程会对数据进行刷盘操作。如果在真正写消息时，pdflush 才进行刷盘的话会影响性能，所以提前进行刷盘。DirectBuffer+Channel 的话，因为 DirectBuffer 没有和 Channel 或 文件有绑定关系，所以 DirectBuffer 只是一块大的内存，不会被 pdflush 刷盘，所以不用在写满后刷盘。~~

**“预热”动作(2)：防止 GC**
warmMappedFile 方法中有一段代码：
```
// prevent gc
if (j % 1000 == 0) {
    log.info("j={}, costTime={}", j, System.currentTimeMillis() - time);
    time = System.currentTimeMillis();
    try {
        Thread.sleep(0);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```
从注释上看，使用 Thread.sleep(0) 来防止 GC。下面一段网上的关于 sleep 和 gc 的说明：
> 这时就需要使用安全区域进行解决。安全区域就是代码片段中引用关系不会发生变化的地方。当线程执行到安全区域的时候，会对线程进行标记，发生GC时候，jvm不管这些线程，在GC的时候，如果这些线程要离开安全区域，此时，判断jvm是否已经完成GC，如果完成，则线程执行;如果没有完成，则线程停顿等待GC完成的信号。（当线程发生sleep时正处于安全区域）
https://blog.csdn.net/b_x_p/article/details/60349291

**有个问题，为什么要防止 GC？**
以下只是猜测：有可能是防止 gc 影响写的时间。因为 gc 的话应该是影响不到 page cache 和内存映射的，所以感觉可能是怕影响写的时间过长。这个防止 GC 的代码是在循环内部的，就是每写一部分数据就要防止一下。


**“预热”动作(3)：mlock**
mlock 方法中，做了两件事：

- mlock：把 MappedFile 的使用的 page cache 内存锁住，让操作在清理内存时，不要清理这些内存。（当内存不够时，操作系统会清理掉一些不使用的内存，page cache 有可能会被直接清理掉 或是 放到 swap cache（根据 page cache 是不是 dirty page）。因为这部分内存是和文件有关的，如果是从文件上读入的话，需要时候再从文件中读出来就好了。）
- madvise：这个系统调用的功能是：建议内核如何处理“给定地址开始部分的`内存页的写入/写出操作`”。例如：如果建议是`MADV_WILLNEED`的话，就表示可能会在很短的时间内要访问给定的地址。这样的话，系统有可能就会预先读取一部分数据到内存中。
这个系统调用允许应用程序告诉内核“它期望如何使用一些共享内存”，然后内核可能会选择适当的“预读”和缓存操作。注意，这个系统调用只是建议内核如何去做，并不是强制，所以内核有可能不做。

mlock 作用是想把 MappedFile 所占用的内存锁住，不让系统清理。而 madvise 选择的是 `MADV_WILLNEED`，作用是让内核预计一些数据到内存中。这里对 madvise 的作用有点疑惑，因为刚刚写完的数据是无效数据，是为了提前产生 page fault 而提高真正写时候的性能才写的数据。把这些无效数据加载到内存中，没有什么作用。而且这些数据已经在内存中了，还使用 mlock 锁住了内存，也不会被释放，这个方法的作用到底是什么呢？

最后说一下，上面两方法，都是系统级别的调用，是和具体的操作系统有关的（这里使用的是 Linux 的系统调用）。在别的操作系统中，要使用其它的系统调用，例如：windows 的话，可能和 mlock 相对应的是 VirtualMlock。

**其它**
mlock 方法中，使用的代码块语法，不知道为什么要使用这种语法，mlock 和 madvise 为什么需要这么急的释放呢？等方法执行完后释放和这种释放，差距有多大呢？

###3，关于 xxxPosition 的作用和使用方法
在进行写数据和刷盘时，使用了几个 position 属性来记录写数据和刷盘的位置。下面对几个 position 进行总结一下：

|    Position    | MMAP | Direct+<br>Chan | 所属类 | 作用 |
|----------|--:|:--:|---|---|
| wrotePosition | O | O | MappedFile | 这个变量是在把消息写到 buffer 时候使用。就是指写消息时，写到 buffer 哪个位置了。这个位置不代表被写的消息已经被刷新到磁盘了，只是告诉你，写下一个消息时从哪开始。<br>MMAP 和 DirectBuffer+Channel 在写消息时，都使用这个 position。|
| committedPosition | X | O | MappedFile | 这个变量是把 buffer 里的消息，写到 FileChannel 时候使用。是指 buffer 里哪些消息已经被写到 FileChannel 里了。CommitRealTimeService 刷盘类使用，其它刷盘类不使用。|
| flushedPosition | O | O | MappedFile | 这个变量是指，buffer 中哪些消息已经被刷新到磁盘。异步刷盘时，flushedPosition 配合 committedPosition 使用。同步刷盘，不使用 committedPosition。|
| committedWhere | X | O | MappedFileQueue | 这个是文件名（fileFromOffset）+ 当前文件最近一个写完的消息的位置（committedPosition），算是一个绝对位置。CommitRealTimeService 刷盘类使用，其它刷盘类不使用。|
| flushedWhere | X | O | MappedFileQueue | 这个是文件名（fileFromOffset）+ 当前文件最近一个刷盘的位置（也就是刷盘时的 wrotePosition 或 committedPosition，刷盘后 wrotePosition 或 committedPosition会随着消息进入而变化，但 flushedWhere 就不变了。当再进行刷盘时，再进行变化）。算是一个绝对位置|
| PHYSICALOFFSET | - | - | - | fileFromOffset+bytebuffer当前的位置。消息结构体中的一个属性，用来记录消息所在的绝对位置。|

**为什么要使用这么多 position 呢？**
从 wrotePosition 来看，不使用这个变量单独记录，而使用 ByteBuffer#position 方法感觉也行，也能达到效果。

为什么要使用 wrotePosition 记录写的位置呢，而不使用`Buffer.position`呢？这个还不太清楚，可能是因为减少变量的使用判断？如果使用 `Buffer.position` 的话，就需要每次都判断`是使用同步写还是异步写(writeBuffer != null)`；如果使用 wrotePosition 的话，就直接取得这个变量就可以了。不知道是不是因为这个原因。

再有，从代码上看，每次进行写时，都是使用`writeBuffer.slice()`或`this.mappedByteBuffer.slice()`取得一个新的`ByteBuffer`，这个新的`ByteBuffer`虽然指向原来的内存位置，但使用新的`ByteBuffer`进行写时，不会改变原来的`writeBuffer`或`mappedByteBuffer`的 Position。这个做法应该是先选择使用 wrotePosition 这种方式后，才这么做的吧，因为写数据的时候，把`writeBuffer`或`mappedByteBuffer`的引用传进去应该也是可以的。

###4，CommitLog.putMessage 的线程安全，是如何保证的？
使用 lockForPutMessage 方法保证的。

###5，外部配置读取
在进行外部配置文件读取后，要覆盖内部默认配置。在覆盖时，并没有使用 setter 方法，而是以 key 相同的方式进行覆盖的。所以有时候有看 broker 配置项目中相关的 setter 方法并没有人调用。具体参看BrokerStartup类中的下面的代码
>// 读取外部配置文件中的配置，并进行设置。
controller.getConfiguration().registerConfig(properties);

###6，DefaultMessageStore#isOSPageCacheBusy
isOSPageCacheBusy 方法是用来判断写消息到 buffer 操作是否繁忙。简单地说，就是写消息时，判断上一个消息是否写完，如果没有写完的话，并且写的时间超过指定时间的话，就算系统繁忙。繁忙的定义就是：
> 系统当前时间 - 当前消息写 Buffer 的开始时间 > 设置繁忙毫秒数


#四、总结
##特点：
- 1，使用了 MMAP 提高文件读写的速度。
- 2，加大系统内存，尽可的使用系统 page cache。
- 3，使用自旋锁。使用自旋锁可以“并发不是特别大”的时候加快处理速度，并发量特别大的时候反而会减慢处理速度。因为是无序的，所以可能会造成某一消息的消费时间过长问题。使用自旋锁还是 ReentrantLock 是可以配置的，但使用 ReentrantLock 时也是使用的“非公平”模式，也是无序的。
- 4，大量使用生成者/消费者模式。消息处理、刷盘、生成 MappedFile。
- 5，同步刷盘使用两个队列来处理要刷盘的消息：requestWrite (接收要落盘的消息) 和 requestRead (把消息落盘时，从从这个队列里读取消息)。有一个队列在刷盘前一起接收消息，在要进行刷盘前，把两个队列的引用交换，这时 requestRead 指向的就是那些要落盘的消息，而 requestWrite 指向一个空的队列，这个空队列负责在刷盘期间继续接收要落盘的消息。 
- 6，ReferenceResource 是一个引用计数类，表示当前类“有几个地方正在使用”、或者当前类“能否使用”。从使用上来看，这个类的作用应该是保证在 broker 进行 shutdown 时候，停止刷盘工作。还有就是在关闭和删除 MappedFile 时候使用。估计是在停机时，因为无法立刻停止所有服务，所以首先让服务无继续进行处理，然后再慢慢停止服务。
- 7，创建 MappedFile 时按顺序创建两个，第一个完成后就返回，第二在在后台进行创建。这样在需要新的 MappedFile 时候，就会立刻得到。
- 8，在用 MMAP 方式得到 ByteBuffer 后进行“预热”，引发 page falut 把内存映射都创建好，减少运行中发生 page fault 问题。并且，把 ByteBuffer 内存锁定，让系统内存清理时无法把这部分清除出去，减少从磁盘读取数据到 page cache 花费的时间。
- 9，使用 sleep 方法，防止 GC。（不太清除这么做的好处有多少）

另外，如果要加快读写速度的话，还可以在以下方面进行优化：

- 1，调整 Linux I/O 调度器。NOOP、DeadLine 等调试方式在不同的磁盘和不同的工作场景，效果都不同。
- 2，使用的磁盘类型（SATA 或 SSD）和 单线程/多线程读写都有关系，可以参考：[聊聊Linux IO](http://0xffffff.org/2017/05/01/41-linux-io/)。
- 3，使用不同的算法。LSM-Tree 的设计便是合理的利用了存储介质的特性，做到了最大化的性能利用（磁盘换成SSD也依旧能有很好的运行效率）。
- 4，可以试试`快速预分配磁盘空间`，即虽然文件没有写满，但先把文件和所占用创建出来(应该用 fallocate)。好处参考：[lseek, fallocate来快速创建一个空洞文件，lseek不占用空间，fallocate占用空间(快速预分配)](https://yq.aliyun.com/articles/11244)
  * （1）可以让文件尽可能的占用连续的磁盘扇区，减少后续写入和读取文件时的磁盘寻道开销；
  * （2）迅速占用磁盘空间，防止使用过程中所需空间不足。
  * （3）后面再追加数据的话，不会需要改变文件大小，所以后面将不涉及metadata的修改。

##问题点：
###1，MappedFile#flush 方法，在同步处理时是否存在丢失消息的可能
感觉有一点奇怪的地方是，如果`hold()`返回 false，就无法刷盘磁盘，但也把 flushedPosition 更新到和 buffer 写位置相同的状态，意思是就是刷新成功，同步消息也会被通知成功。但其实没有刷盘成功，这样同步消息是不是也存在丢失消息的问题呢？

###2，关于 MappedFile 的 warm 和 init 方法有一些疑问。
关于 rocketmq 写消息的处理，个人的一些理解如下：

 - 当 TransientStorePoolEnable 为 true 时，使用 writeBuffer 来写消息；为 false 时，使用 mappedByteBuffer 写消息。
 - writeBuffer 和 mappedByteBuffer 无法同时用来写消息。要么使用 writeBuffer，要么使用 mappedByteBuffer。


如果上面的理解没有问题的话，
前提条件：

 - TransientStorePoolEnable: true
 - FlushDiskType: ASYNC

1，在 MappedFile#init(String, int, TransientStorePool) 方法里，
为什么既调用了 init(String, int) 初始化了 mappedByteBuffer 后，又初始了 writeBuffer。
从方法的签名上看，应该是想使用 writeBuffer，初始化 mappedByteBuffer 的作用是什么呢？
```
public void init(final String fileName, final int fileSize, final TransientStorePool transientStorePool) throws IOException {
    init(fileName, fileSize);
    this.writeBuffer = transientStorePool.borrowBuffer();
    this.transientStorePool = transientStorePool;
}

private void init(final String fileName, final int fileSize) throws IOException {
    ... 省略

    ensureDirOK(this.file.getParent());

    try {
        this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();
        this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);
}
```


2，AllocateMappedFileService#mmapOperation 中有一段代码是调用 mappedFile.warmMappedFile 方法。
在 warmMappedFile 方法中，会对 mappedByteBuffer 指向的文件写数据。
如果是 TransientStorePoolEnable: true 并且 FlushDiskType: ASYNC 的话，向 mappedByteBuffer 写的数据的作用是什么呢？
```
if (mappedFile.getFileSize() >= this.messageStore.getMessageStoreConfig()
    .getMapedFileSizeCommitLog()
    &&
    this.messageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
    mappedFile.warmMappedFile(this.messageStore.getMessageStoreConfig().getFlushDiskType(),
        this.messageStore.getMessageStoreConfig().getFlushLeastPagesWhenWarmMapedFile());
}


public void warmMappedFile(FlushDiskType type, int pages) {
    long beginTime = System.currentTimeMillis();
    ByteBuffer byteBuffer = this.mappedByteBuffer.slice();
    int flush = 0;
    long time = System.currentTimeMillis();
    for (int i = 0, j = 0; i < this.fileSize; i += MappedFile.OS_PAGE_SIZE, j++) {
        byteBuffer.put(i, (byte) 0);
        // force flush when flush disk type is sync
        if (type == FlushDiskType.SYNC_FLUSH) {
            if ((i / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE) >= pages) {
                flush = i;
                mappedByteBuffer.force();
            }
        }
        ... 省略
}
```


参考：
https://blog.csdn.net/prestigeding/article/details/79188383 
https://www.jianshu.com/p/b93604adb8b7
https://blog.csdn.net/mr253727942/article/details/55805876
http://www.iocoder.cn/RocketMQ/message-store/
https://blog.csdn.net/u013160932/article/details/58191057
https://blog.csdn.net/u013160932/article/details/57472832
http://www.sxt.cn/ueditor/php/upload/file/20170901/1504248466272324.pdf
