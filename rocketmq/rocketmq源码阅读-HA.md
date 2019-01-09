
> HA 主要作用是，Master 向 Slave 传送消息信息。Slave 接收到消息信息后，向 Master 报告自己现在同步的消息的 commit log offset，来让 Master 判断是否已经同步完成。

#一，HA 介绍
##1，Broker 高可用
启动多个 Broker分组 形成 集群 实现高可用。
Broker分组 = Master节点x1 + Slave节点xN。
类似 MySQL，Master节点 提供读写服务，Slave节点 只提供读服务。

##2，Broker 主从

- 每个分组，Master节点 不断发送新的 CommitLog 给 Slave节点。 Slave节点 不断上报本地的 CommitLog 已经同步到的位置给 Master节点。
- Broker分组 与 Broker分组 之间没有任何关系，不进行通信与数据同步。
- 消费进度 目前不支持 Master/Slave 同步。

集群内，Master节点 有两种类型：Master_SYNC、Master_ASYNC：前者在 Producer 发送消息时，等待 Slave节点 存储完毕后再返回发送结果，而后者不需要等待。

##3，配置方案
目前官方提供三套配置：

2m-2s-async
| brokerClusterName | brokerName | brokerRole | brokerId |
| -- | -- | -- | -- |
| DefaultCluster | broker-a | ASYNC_MASTER | 0 |
| DefaultCluster | broker-a | SLAVE | 1 |
| DefaultCluster | broker-b | ASYNC_MASTER | 0 |
| DefaultCluster | broker-b | SLAVE | 1 |

2m-2s-sync
| brokerClusterName | brokerName | brokerRole | brokerId |
| -- | -- | -- | -- |
| DefaultCluster | broker-a | SYNC_MASTER | 0 |
| DefaultCluster | broker-a | SLAVE | 1 |
| DefaultCluster | broker-b | SYNC_MASTER | 0 |
| DefaultCluster | broker-b | SLAVE | 1 |

2m-noslave

| brokerClusterName | brokerName | brokerRole | brokerId |
| -- | -- | -- | -- |
|DefaultCluster | broker-a | ASYNC_MASTER | 0 |
|DefaultCluster | broker-b | ASYNC_MASTER | 0 |

##4，传输内容

| 对象 | 用途 | 第几位 | 字段 | 数据类型 | 字节数 | 说明 |
| -- | -- | -- | -- | -- | -- | -- |
| Slave=>Master | 上报CommitLog已经同步到的物理位置 | 0 | maxPhyOffset | Long | 8 | CommitLog最大物理位置 |
| Master=>Slave | 传输新的 CommitLog 数据 | 0 | fromPhyOffset | Long | 8 | CommitLog开始物理位置 |
| - | - | 1 | size | Int | 4 | 传输CommitLog数据长度 |
| - | - | 2 | body | Bytes | size | 传输CommitLog数据 |
参考：[RocketMQ 源码分析 —— 高可用](http://www.iocoder.cn/RocketMQ/high-availability/)


#二、基础介绍
##1，Master 相关的类

- HAService：Master 和 Slave 启动后，都会启动这个类。这个类启动后，会继续启动 Master 和 Slave 的相关类。有两个属性是比较重要的：
 - waitNotifyObject：这个属性是用来控制 WriteSocketService 的`唤醒/睡眠`的。WriteSocketService 在没有向 Slave 发的数据时，就会使用 allWaitForRunning 方法使自己睡眠。当有消息要同步到 Slave 时，就会使用 wakeupAll 来唤醒它。
 - push2SlaveMaxOffset：用来保存 Slave 发过来的最大的 commit log offset。
- AcceptSocketService：HAService 启动后，首先启动的是 AcceptSocketService，这个类是负责接收 Slave 的创建连接请求，然后生成连接 HAConnection。它的工作仅此而已，没有其它作用了。(没测试过，Slave 应该也启动这个类，但没有人去连接它)
- GroupTransferService：负责保持那些要同步 Slave 的消息的请求。当同步完成后，就会让 request 继续执行。(没测试过，Slave 应该也启动这个类，但 Slave 在执行 CommitLog 时，不会执行那一段`如果是 Master 就同步 Slave`的处理)
- HAConnection：而 HAConnection 是负责处理`发送/接收`具体数据的类，在这个类中包含了两个子类：ReadSocketService 和 WriteSocketService，这两个类就是负责`发送/接收`的具体类。下面我们看看每个类的功能
- ReadSocketService：用来读取 Slave 发过来的 offset 的。
- WriteSocketService：用来向 Slave 发送已经保存的消息。只有 Master 有新消息，就会向 Slave 发送，不管 ReadSocketService 如何。

##2，Slave 相关的类

- HAService：和 Master 的作用一样。
- HAClient：主要是 Slave 进行 commit log offset `接收/发送`的类。这个类是 Slave 使用的类，Master 也会启动这个类，但在 Master 侧启动的类，是一个在`while()`里空跑的类。因为有一个 HAClient 类里面有一个 masterAddress 属性。 这个属性是在 BrokerController 启动时设置的，启动时判断是否是 Slave，如果不是就是设置这个属性。这个属性为空的话，就不会创建连接，每 5 秒循环一次这样的操作。

##3，时序图
下面两张时序图，是从下面的文章上转过来的，下面的文件写的挺好的，建议大家看看。

- [源码研究RocketMQ主从同步机制(HA)](https://blog.csdn.net/prestigeding/article/details/79600792)
- [RocketMQ源码分析----HA相关(1)](https://blog.csdn.net/u013160932/article/details/80299544)

![这里写图片描述](https://img-blog.csdn.net/20180913092343199?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![这里写图片描述](https://img-blog.csdn.net/20180913092350862?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


#二、Master 和 Slave 流程讲解
##1，Slave 请求连接
HAService 调用 HAClient#run 方法后，启动 HAClient 这个类。HAClient 首先要做的是向 Master 发起连接请求。我们看一下 run 方法：
```
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            // 连接 Master
            if (this.connectMaster()) {
                // 看是否到达心跳时间，到达后就发送心跳。
                if (this.isTimeToReportOffset()) {
                    boolean result = this.reportSlaveMaxOffset(this.currentReportedOffset);
                    if (!result) {
                        // 如果发送出错，就关闭连接
                        this.closeMaster();
                    }
                }

                // 每 1 秒处理一次数据。就算还有数据也不读了，处理已经读到的数据。
                // 如果下面的处理比较慢，要等到下面处理完后，再进行 1 秒读取。
                this.selector.select(1000);

                boolean ok = this.processReadEvent();
                if (!ok) {
                    this.closeMaster();
                }

                if (!reportSlaveMaxOffsetPlus()) {
                    continue;
                }

                long interval =
                    HAService.this.getDefaultMessageStore().getSystemClock().now()
                        - this.lastWriteTimestamp;
                if (interval > HAService.this.getDefaultMessageStore().getMessageStoreConfig()
                    .getHaHousekeepingInterval()) {
                    log.warn("HAClient, housekeeping, found this connection[" + this.masterAddress
                        + "] expired, " + interval);
                    this.closeMaster();
                    log.warn("HAClient, master not response some time, so close connection");
                }
            } else {
                this.waitForRunning(1000 * 5);
            }
        } catch (Exception e) {
            log.warn(this.getServiceName() + " service has exception. ", e);
            this.waitForRunning(1000 * 5);
        }
    }

    log.info(this.getServiceName() + " service end");
}
```

在 run 方法里，有连接和处理等代码，我们先看一下连接代码。
```
private boolean connectMaster() throws ClosedChannelException {
    if (null == socketChannel) {
        String addr = this.masterAddress.get();
        if (addr != null) {
            SocketAddress socketAddress = RemotingUtil.string2SocketAddress(addr);
            if (socketAddress != null) {
                // 连接 master
                this.socketChannel = RemotingUtil.connect(socketAddress);
                if (this.socketChannel != null) {
                    this.socketChannel.register(this.selector, SelectionKey.OP_READ);
                }
            }
        }

        // 初始化
        this.currentReportedOffset = HAService.this.defaultMessageStore.getMaxPhyOffset();

        this.lastWriteTimestamp = System.currentTimeMillis();
    }

    return this.socketChannel != null;
}
```

##2，Master 接收连接请求
AcceptSocketService 方法是用来接收并创建 Slave 来的连接的，然后生成连接 HAConnection。它的工作仅此而已，没有其它作用了。这个类的主要方法有两个：beginAccept 和 run。

- beginAccept：负责生成 NIO 的 Selector 和 ServerSocketChannel。
- run：接收 Slave 的连接创建请求，创建 HAConnection。
```
public void beginAccept() throws Exception {
    this.serverSocketChannel = ServerSocketChannel.open();
    this.selector = RemotingUtil.openSelector();
    this.serverSocketChannel.socket().setReuseAddress(true);
    this.serverSocketChannel.socket().bind(this.socketAddressListen);
    this.serverSocketChannel.configureBlocking(false);
    // 接收连接创建请求
    this.serverSocketChannel.register(this.selector, SelectionKey.OP_ACCEPT);
}


public void run() {
    while (!this.isStopped()) {
        try {
            this.selector.select(1000);
            Set<SelectionKey> selected = this.selector.selectedKeys();

            if (selected != null) {
                for (SelectionKey k : selected) {
                    // 如果有连接创建请求
                    if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
                        SocketChannel sc = ((ServerSocketChannel) k.channel()).accept();

                        if (sc != null) {
                            HAService.log.info("HAService receive new connection, "
                                + sc.socket().getRemoteSocketAddress());

                            try {
                                HAConnection conn = new HAConnection(HAService.this, sc);
                                conn.start();
                                HAService.this.addConnection(conn);
                            } catch (Exception e) {
                                log.error("new HAConnection exception", e);
                                sc.close();
                            }
                        }
                    } else {
                        log.warn("Unexpected ops in select " + k.readyOps());
                    }
                }

                selected.clear();
            }
        } catch (Exception e) {
            log.error(this.getServiceName() + " service has exception.", e);
        }
    }
}
```

##3，Slave 发送数据
Slave 有两种情况向 Master 发送数据：

- 心跳时间到了，发送心跳数据
- 从 Master 读到数据，保存后，向 Master 发回`现在 Slave 最大的 commit log offset`

因为 HAConnection#ReadSocketService 不接收到数据的话，HAConnection#WriteSocketService 无法向 Slave 发送数据。WriteSocketService 不发送数据，也就是说 Slave 无法从 Master 读取数据。所以，Slave 首先发送的是心跳数据（心跳数据就是 Slave 侧`最大的 commit log offset`）。心跳的默认时间为 5 秒，select(1000) 方法经过 5 次后，就可以发送心跳了。我们看一下发送心跳数据的方法：
```
/**
 * 向 master 报告 commitOffset。
 * @param maxOffset
 * @return
 */
private boolean reportSlaveMaxOffset(final long maxOffset) {
    this.reportOffset.position(0);
    this.reportOffset.limit(8);
    this.reportOffset.putLong(maxOffset);
    // 下面代码可不可以换成 reportOffset.flip()? 感觉作用相同。
    this.reportOffset.position(0);
    this.reportOffset.limit(8);

    // 如果 3 次还没有写完，说明缓冲区等可能出现问题，退出发送数据。
    for (int i = 0; i < 3 && this.reportOffset.hasRemaining(); i++) {
        try {
            this.socketChannel.write(this.reportOffset);
        } catch (IOException e) {
            log.error(this.getServiceName()
                + "reportSlaveMaxOffset this.socketChannel.write exception", e);
            return false;
        }
    }

    // 如果数据没有全写完，说明出错了
    return !this.reportOffset.hasRemaining();
}
```

##4，Master 接收数据
ReadSocketService 创建完连接和 HAConnection 后，就由 HAConnection 和 Slave 进行具体的数据交互了。HAConnection 中有两个类：ReadSocketService 和 WriteSocketService。ReadSocketService 是负责读取从 Slave 来的数据的，WriteSocketService 是负责向 Slave 写数据的。我们先看看 ReadSocketService 的代码：
```
public ReadSocketService(final SocketChannel socketChannel) throws IOException {
    this.selector = RemotingUtil.openSelector();
    this.socketChannel = socketChannel;
    // 设置 ReadSocketService 的 SocketChannel 处理读请求
    this.socketChannel.register(this.selector, SelectionKey.OP_READ);
    // 设置守护线程。
    this.thread.setDaemon(true);
}

public void run() {
    HAConnection.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            this.selector.select(1000);
            // 处理读取到的数据
            boolean ok = this.processReadEvent();
            if (!ok) {
                HAConnection.log.error("processReadEvent error");
                break;
            }

            long interval = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now() - this.lastReadTimestamp;
            if (interval > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaHousekeepingInterval()) {
                break;
            }
        } catch (Exception e) {
            break;
        }
    }

    this.makeStop();
    writeSocketService.makeStop();
    haService.removeConnection(HAConnection.this);
    HAConnection.this.haService.getConnectionCount().decrementAndGet();
    SelectionKey sk = this.socketChannel.keyFor(this.selector);
    if (sk != null) {
        sk.cancel();
    }

    try {
        this.selector.close();
        this.socketChannel.close();
    } catch (IOException e) {
        HAConnection.log.error("", e);
    }
}


// 处理数据
private boolean processReadEvent() {
    int readSizeZeroTimes = 0;
    // 如果 byteBufferRead 满了，就清空数据。（为什么不使用 clear 呢？）
    if (!this.byteBufferRead.hasRemaining()) {
        this.byteBufferRead.flip();
        this.processPostion = 0;
    }

    while (this.byteBufferRead.hasRemaining()) {
        try {
            // 通过 readSize 判断是否读取到数据
            int readSize = this.socketChannel.read(this.byteBufferRead);
            if (readSize > 0) {
                readSizeZeroTimes = 0;
                this.lastReadTimestamp = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now();
                if ((this.byteBufferRead.position() - this.processPostion) >= 8) {
                    int pos = this.byteBufferRead.position() - (this.byteBufferRead.position() % 8);
                    long readOffset = this.byteBufferRead.getLong(pos - 8);
                    this.processPostion = pos;

                    HAConnection.this.slaveAckOffset = readOffset;
                    // 通过从 Slave 发过来的 offset，更新`本地的 Slave offset`
                    if (HAConnection.this.slaveRequestOffset < 0) {
                        HAConnection.this.slaveRequestOffset = readOffset;
                        log.info("slave[" + HAConnection.this.clientAddr + "] request offset " + readOffset);
                    }
                    // 通知那些正处于"等待 Slave 反馈状态"的消息
                    HAConnection.this.haService.notifyTransferSome(HAConnection.this.slaveAckOffset);
                }
            } else if (readSize == 0) {
                if (++readSizeZeroTimes >= 3) {
                    break;
                }
            } else {
                log.error("read socket[" + HAConnection.this.clientAddr + "] < 0");
                return false;
            }
        } catch (IOException e) {
            log.error("processReadEvent exception", e);
            return false;
        }
    }

    return true;
}

```

这里展示了两个方法，一个是构造函数，还有一个是 processReadEvent() 方法。构造函数里设置了 SocketChannel 只对 read 事件做出反应。processReadEvent 方法是主要的处理方法，下面我们具体分析一下这个方法的代码：

- 判断 byteBufferRead 是否有还空间接收数据，如果 byteBufferRead 满了，就清空数据。（为什么不使用 clear 呢？）
- 从读取的数据中，取得所有 offset 数据中，最后一个 offset。
- 记录取得的 offset 到 slaveAckOffset。如果 slaveRequestOffset 还没初始化过，把它设置成取得的 offset（slaveRequestOffset 主要在 WriteSocketService 中使用，这里主要是初始化它。不初始化它，WriteSocketService 无法写数据到 Slave。）
- 通知`发送需要同步刷新 Slave 的消息线程`，看当前这个 offset 是否可以判断同步刷新成功。（第一次通信时，没有消息需要通知）

**处理说明：**
（1）在这些处理中，`从读取的数据中，取得所有 offset 数据中，最后一个 offset。`这个处理看似简单，其实当中还处理了`粘包`问题。首先我们看一下下面的代码：
```
if ((this.byteBufferRead.position() - this.processPostion) >= 8) {
    ...
}
```
代码说明：
processPostion 是用来记录上次处理到 byteBufferRead 哪个位置了。这个比较意思是：`如果从网络缓冲区读到的数据不够 offset 数据长度，就跳过处理，再进行读取`。因为无法控制从网络上一定能读取到指定字节的数据，所以每次处理前都要判断是否读到最小能处理字节数了。读到后再进行处理，否则再进行读取。

```
int pos = this.byteBufferRead.position() - (this.byteBufferRead.position() % 8);
```
代码说明：
如果读到可处理的数据之后，还要判断`最后一个可处理的数据的位置在哪`。这里我们要说明两个内容：

- 最后一个可处理数据：为什么要取得最后一个可处理数据呢？因为从 Slave 传过来的是`落盘成功 offset 数据`，假如 Master 一次接收到 3 条 offset 数据，数据为 1000、1010、1020。那么通过最后一条 offset 数据 1020，就可以判断它之前的消息都已经在 slave 落盘成功，不用一条一条读取并判断了。
- 位置在哪：上面提了一下，我们无法控制网络中字节的传输，所以可能一次读到的字节数不是正好的 offset 长度，有可能多了几个字节。例如，我们读取到了 49 个字节的内容，但我们想要的是 41~48 字节的内容，所以我们需要计算出从哪个位置开始读。
 - `this.byteBufferRead.position() % 8`：用来计算出多余的字节数（例子中的第 49 个字节数据是多余的，所以这里计算出来的结果是 1。）
 - `this.byteBufferRead.position() - (this.byteBufferRead.position() % 8)`：把多余的字节去掉，这时指针指向合法的 offset 数据的最后一个位置（48）。
 - `long readOffset = this.byteBufferRead.getLong(pos - 8);`：这里把指针指向最后一个 offset 数据的初始位置后，读取这个 offset 数据。

（2）如果 slaveRequestOffset 还没初始化过，把它设置成取得的 offset
处理完这个后，通信的初始化就完成了，就可以进行向 Slave 同步消息了。


##5，Client 发送消息，并要同步到 Slave
接着介绍一个消息要同步到 Slave 时的流程。当系统设置为`消息同步 Slave`时，会执行 CommitLog#putMessage 方法下面的代码：
```
// Synchronous write double
if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
    HAService service = this.defaultMessageStore.getHaService();
    if (msg.isWaitStoreMsgOK()) {
        // Determine whether to wait
        if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
            if (null == request) {
                request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
            }
            // 提交同步请求。如果数据写到 Slave 成功的话，会通知这个请求。
            service.putRequest(request);
            // 叫醒"向 Slave 写 offset 数据"的线程
            // ("向 Slave 写 offset 数据的线程"如果没有数据可写，会睡一会)
            service.getWaitNotifyObject().wakeupAll();
            // 等待消息向 Slave 同步。如果 timeout 时间之内没有同步完成的话，就算同步失败。
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
```

代码说明：
（1）首先要说明的是`service.putRequest(request);`代码，这个代码的意思是`提交一个 Slave 的同步消息请求`。当消息被同步到 Slave 成功后，就会通知这个请求。`service.putRequest(request);`的内部调用的是`GroupTransferService#putRequest`方法，我们直接看一下这个方法的内容。
```
public void putRequest(final CommitLog.GroupCommitRequest request) {
    synchronized (this) {
        this.requestsWrite.add(request);
        if (hasNotified.compareAndSet(false, true)) {
            waitPoint.countDown(); // notify
        }
    }
}
```
这个方法的内容很简单，把 request 放到 requestsWrite 这个 List 中。然后通过 ServiceThread 类的特性，告诉 GroupTransferService 类不用等待了，可以执行处理逻辑了。

那 request 是在哪里被处理的呢？是在 GroupTransferService#doWaitTransfer 方法里被处理的。处理内容如下：

- 首先判断 requestsRead 是否为空。request 放到了 requestsWrite 里，为什么从 requestsRead 里读取呢？ 这个结构和 GroupCommitService 里的 requestsRead 和 requestsWrite 的结构是一样的。requestsWrite 是用来接收请求的，处理时就把 requestsWrite 里的请求放到 requestsRead 里后，requestsWrite 继续去接收请求。
- 循环每一个 request，判断`当前保存的 Slave 最大的 offset`是否比`request 的offset`大，如果是的话，就通知 request 同步成功，可以继续执行了；如果不是，就等待 1 秒。

一般来说，消息不会那么快传到 Slave，并且再从 Slave 传回来确认信息，所以一般都会先睡 1 秒。
```
private void doWaitTransfer() {
    if (!this.requestsRead.isEmpty()) {
        for (CommitLog.GroupCommitRequest req : this.requestsRead) {
            boolean transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
            for (int i = 0; !transferOK && i < 5; i++) {
                this.notifyTransferObject.waitForRunning(1000);
                transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
            }

            if (!transferOK) {
                log.warn("transfer messsage to slave timeout, " + req.getNextOffset());
            }

            req.wakeupCustomer(transferOK);
        }

        this.requestsRead.clear();
    }
}
```



（2）再看一下`service.getWaitNotifyObject().wakeupAll();`这段代码，这段代码的意思是：叫醒"向 Slave 写 offset 数据"的线程（WriteSocketService）"。WriteSocketService 如果没有数据可写，会睡一会。代码如下：
```
@Override
public void run() {
    HAConnection.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            ...
            SelectMappedBufferResult selectResult =
                HAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);
            if (selectResult != null) {
                ...
            } else {
                // 没有数据可向 Slave 传送的话，就睡 100 ms
                HAConnection.this.haService.getWaitNotifyObject().allWaitForRunning(100);
            }
        } catch (Exception e) {
        ...
}
```

（3）接下来通过 request 等待消息向 Slave 的同步。如果超时时间内没有同步完成的话，就算同步失败。从 SendMessageProcessor 的处理来看，只要保存到 Master 成功了，就算成功。

接下来我们看一下，消息是如何同步到 Slave 的。

##6，消息发送到 Slave
消息发送到 Slave 是由 WriteSocketService 来做的。这个 service 是一个
“与 Slave 连接后”，启动的线程。这个 service 启动后，只有网络缓冲区里有空间可写数据，就从 CommitLog 取得未发送的消息，发送给 Slave。我们看一下代码：
```
@Override
public void run() {
    HAConnection.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            this.selector.select(1000);

            // 如果 slave 还没发过来请求（也就是 还没发送过来 offset），就等待
            // (slave 发送过来 offset 后，slaveRequestOffset 就不为负数了)
            if (-1 == HAConnection.this.slaveRequestOffset) {
                Thread.sleep(10);
                continue;
            }

            // 如果 nextTransferFromWhere 还末初始化，就初始化它。
            if (-1 == this.nextTransferFromWhere) {
                if (0 == HAConnection.this.slaveRequestOffset) {
                    long masterOffset = HAConnection.this.haService.getDefaultMessageStore().getCommitLog().getMaxOffset();
                    // 计算后，masterOffset 变成"最后一个 MappedFile 初始位置"了
                    masterOffset =
                        masterOffset
                            - (masterOffset % HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
                            .getMapedFileSizeCommitLog());

                    if (masterOffset < 0) {
                        masterOffset = 0;
                    }

                    this.nextTransferFromWhere = masterOffset;
                } else {
                    this.nextTransferFromWhere = HAConnection.this.slaveRequestOffset;
                }

                log.info("master transfer data from " + this.nextTransferFromWhere + " to slave[" + HAConnection.this.clientAddr
                    + "], and slave request " + HAConnection.this.slaveRequestOffset);
            }

            // 如果上次传送完成的话
            // (启动后第一次执行时，lastWriteOver 就为 true，所以不执行 transferData() )
            if (this.lastWriteOver) {

                // 距离上次写数据，是否已经过了一定时间，
                long interval =
                    HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now() - this.lastWriteTimestamp;

                // 如果时间到了发送心跳的间隔，就发送心跳。
                if (interval > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
                    .getHaSendHeartbeatInterval()) {

                    // 心跳只发送 8 字节的 header，body size 和 body 都为空
                    // Build Header
                    this.byteBufferHeader.position(0);
                    this.byteBufferHeader.limit(headerSize);
                    this.byteBufferHeader.putLong(this.nextTransferFromWhere);
                    this.byteBufferHeader.putInt(0);
                    this.byteBufferHeader.flip();

                    this.lastWriteOver = this.transferData();
                    if (!this.lastWriteOver)
                        continue;
                }
            } else { // 如果上次传没完成，就再进行传送
                this.lastWriteOver = this.transferData();
                if (!this.lastWriteOver)
                    continue;
            }

            // 上次传送完成情况，正常发下一次的数据。
            // 取得"从 nextTransferFromWhere 到目前所有的消息"
            SelectMappedBufferResult selectResult =
                HAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);
            if (selectResult != null) {
                int size = selectResult.getSize();
                if (size > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize()) {
                    size = HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize();
                }

                long thisOffset = this.nextTransferFromWhere;
                this.nextTransferFromWhere += size;

                selectResult.getByteBuffer().limit(size);
                this.selectMappedBufferResult = selectResult;

                // Build Header
                this.byteBufferHeader.position(0);
                this.byteBufferHeader.limit(headerSize);
                this.byteBufferHeader.putLong(thisOffset);
                this.byteBufferHeader.putInt(size);
                this.byteBufferHeader.flip();

                this.lastWriteOver = this.transferData();
            } else {
                // 没有数据可向 Slave 传送的话，就睡 100 ms
                HAConnection.this.haService.getWaitNotifyObject().allWaitForRunning(100);
            }
        } catch (Exception e) {

            HAConnection.log.error(this.getServiceName() + " service has exception.", e);
            break;
        }
    }
    ...
```
run 方法主要做了以下几个事：

- 如果发送心跳时间间隔到了，就发送心跳。
- 从 CommitLog 中取得消息，如果有新消息，发送给 Slave；如果没有新消息，就睡 100 ms。

##7，Slave 接收消息
我们回过头来看一下 HAClient 的代码。接收消息的代码如下：
```
private boolean processReadEvent() {
    int readSizeZeroTimes = 0;
    // 如果 byteBufferRead 还有空间可用，就从 channel 里读取数据
    while (this.byteBufferRead.hasRemaining()) {
        try {
            // 从 channel 里读取数据
            int readSize = this.socketChannel.read(this.byteBufferRead);
            if (readSize > 0) {
                lastWriteTimestamp = HAService.this.defaultMessageStore.getSystemClock().now();
                readSizeZeroTimes = 0;
                boolean result = this.dispatchReadRequest();
                if (!result) {
                    log.error("HAClient, dispatchReadRequest error");
                    return false;
                }
            } else if (readSize == 0) {
                if (++readSizeZeroTimes >= 3) {
                    break;
                }
            } else {...}
        } catch (IOException e) {...}
    }
    return true;
}

private boolean dispatchReadRequest() {
    final int msgHeaderSize = 8 + 4; // phyoffset + size
    int readSocketPos = this.byteBufferRead.position();

    while (true) {
        int diff = this.byteBufferRead.position() - this.dispatchPostion;
        if (diff >= msgHeaderSize) {
            long masterPhyOffset = this.byteBufferRead.getLong(this.dispatchPostion);
            int bodySize = this.byteBufferRead.getInt(this.dispatchPostion + 8);

            long slavePhyOffset = HAService.this.defaultMessageStore.getMaxPhyOffset();

            if (slavePhyOffset != 0) {
                if (slavePhyOffset != masterPhyOffset) {
                    log.error("master pushed offset not equal the max phy offset in slave, SLAVE: "
                        + slavePhyOffset + " MASTER: " + masterPhyOffset);
                    return false;
                }
            }

            // 如果读取到的数据大小 超过 一条数据完整数据的大小的话，就读取数据内容。
            if (diff >= (msgHeaderSize + bodySize)) {
                byte[] bodyData = new byte[bodySize];
                this.byteBufferRead.position(this.dispatchPostion + msgHeaderSize);
                // 注意：如果 byteBufferRead 的数据不够填充 bodyData 的大小的话，就会抛出异常。
                this.byteBufferRead.get(bodyData);

                // 把读取到的数据，写到 commitLog 里
                HAService.this.defaultMessageStore.appendToCommitLog(masterPhyOffset, bodyData);

                // 把 byteBufferRead 的 position 恢复到初始状态。
                // 在整个处理中，是使用 dispatchPostion 做下一条数据的位置记录的。
                this.byteBufferRead.position(readSocketPos);
                // 修改 dispatchPostion，生成下一条数据的初始位置
                this.dispatchPostion += msgHeaderSize + bodySize;

                // 向 maseter 报告自己的最大 commitLog offset
                if (!reportSlaveMaxOffsetPlus()) {
                    return false;
                }

                continue;
            }
        }

        // 如果 byteBufferRead 没有空间了，就交换
        if (!this.byteBufferRead.hasRemaining()) {
            this.reallocateByteBuffer();
        }

        break;
    }

    return true;
}
```
processReadEvent 方法主要是有没有消息需要处理。如果有消息需要处理，就调用 dispatchReadRequest 方法来处理。所以，主要的处理在 dispatchReadRequest 方法里。dispatchReadRequest 方法细节很多，稍后进行介绍。

在保存完消息后，会调用 reportSlaveMaxOffsetPlus 方法把当前 Slave 最大的 offset 发回给 Master。这个方法处理比较简单，就不再叙述了。



##8，Master 取得 Slave 的反馈
Slave 把最大的 offste 返回给 Master 后，Master 使用 ReadSocketService 来读取 offset。ReadSocketService 代码在上面已经讲过了，我们看一下 run 里的下面段代码，这段代码是从 Slave 取得“最大 offset”后，通知那些正处于"等待 Slave 反馈状态"的消息，看是否可以返回。
```
HAConnection.this.haService.notifyTransferSome(HAConnection.this.slaveAckOffset);
```

接下来我们看一下 notifyTransferSome 方法的具体实现。这段代码也很简单，如果最新的 offset 比“之前保存的 slave 最大 offset 大”，就更新“Slave 最大 offset”变量，然后通知那些正处于"等待 Slave 反馈状态"的消息，看是否可以返回。
```
public void notifyTransferSome(final long offset) {
    for (long value = this.push2SlaveMaxOffset.get(); offset > value; ) {
        boolean ok = this.push2SlaveMaxOffset.compareAndSet(value, offset);
        if (ok) {
            this.groupTransferService.notifyTransferSome();
            break;
        } else {
            value = this.push2SlaveMaxOffset.get();
        }
    }
}
```

通知等待消息是使用的 GroupTransferService#notifyTransferSome 方法。这个方法作用是：告诉 GroupTransferService#doWaitTransfer 方法可以继续运行了。
```
public void notifyTransferSome() {
    this.notifyTransferObject.wakeup();
}
```

我们再回来看 GroupTransferService 接下来执行处理逻辑，我们看一下 doWaitTransfer 方法。如果 GroupTransferService 线程因为走到 waitForRunning(1000) 而等待的话，`this.notifyTransferObject.wakeup()`就会把它叫醒，让它继续向下执行，去确认消息是否同步 Slave 成功。
```
private void doWaitTransfer() {
    if (!this.requestsRead.isEmpty()) {
        for (CommitLog.GroupCommitRequest req : this.requestsRead) {
            boolean transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
            for (int i = 0; !transferOK && i < 5; i++) {
                this.notifyTransferObject.waitForRunning(1000);
                transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
            }

            if (!transferOK) {
                log.warn("transfer messsage to slave timeout, " + req.getNextOffset());
            }

            req.wakeupCustomer(transferOK);
        }

        this.requestsRead.clear();
    }
}
```

如果`Slave 传过来的 offset`比`消息 request 的 offset` 大，就告诉 request 同步结果，并让 request 继续向下执行。
```
public void wakeupCustomer(final boolean flushOK) {
    this.flushOK = flushOK;
    this.countDownLatch.countDown();
}
```

我们再回来看 CommitLog 中的代码，如果当前线程停在 request.waitForFlush 方法的话，上面的 wakeupCustomer 方法会让代码继续向下走，进行后续处理。
```
// Synchronous write double
if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
    HAService service = this.defaultMessageStore.getHaService();
    if (msg.isWaitStoreMsgOK()) {
        // Determine whether to wait
        if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
            ...
            boolean flushOK =
                request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
            if (!flushOK) {
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
```

<br>
#细节
##1，关于 WriteSocketService#transferData 中的 writeSize == 0
SocketChannel#write 方法返回值是表示，写到网络缓冲区里的字节数。但有时候返回值为 0，什么时候返回值为 0 呢？

上网上找了一下，有时候网络缓冲区满了，写不进去了就返回 0。在这种情况下，RocketMQ 做法是如果连续 3 次出现这种情况，就不写了，让 selector 去判断是否可写。如果可以了，再进行写处理。这样好处是把判断交给了 selector，可以节省一些资源（Linux 上 selector 应该是 epoll 注册类型的，等有事件处发了再交给应用执行）。

##2，关于 WaitNotifyObject
在 HA 相关的类中，用到了这个类。他们是怎么使用这个类的呢？主要是用来进行“线程唤醒”解耦。

举个例子：WriteSocketService 在没有消息可以向 Slave 发时，就会睡一会。睡一会这个操作就是调用 WaitNotifyObject#allWaitForRunning 方法，让当前线程进行睡一会（实际上调用的是 wait 方法）。而当有新消息来时，要向 Slave 同步时，会调用 WaitNotifyObject#wakeupAll 方法叫醒正在等待的 WriteSocketService 线程。

GroupTransferService 里的 WaitNotifyObject 对象也是用来做这个事的。当持有的 Slave Max Offset 比当前消费的保存后的 offset 小的话，就让 GroupTransferService 线程等待一会，等 ReadSocketService 读取到了新的 offst 后，再叫醒 GroupTransferService 去比较一下。

##3，关于粘包处理
HAClient#dispatchReadRequest 和 ReadSocketService#processReadEvent 方法是使用 NIO 接收数据的两个方法，这两个方法都做了粘包处理，而使用的方式不太一样，因为数据是不是固定长的关系。下面就说说两种处理粘包的方式。

###（1）ReadSocketService#processReadEvent
ReadSocketService 接收的数据是一个` 8 字节的 commit log offset`。因为数据的长度是固定的，所以处理起来相对简单一些。

**前期准备：**
这里定义了一个 ByteBuffer 变量来接收数据，变量名字为：byteBufferRead。长度为 1024 * 1024 正好为 8 的倍数。如果长度不为 8 的倍数的话，可能会产生无法完整读取一条数据的问题，这一点要注意。

**处理：**
每次读到数据后，就判断一下`读取到的数据 >= 数据固定长度(8)`

- 如果小于的话，说明数据没有读取完成，需要再进行读取数据。
- 如果大于的话，说明`至少已经读取到一条完整数据了`。但至于读取到的数据是`N 条完成数据`，还是`N 条完整数据 + 一部分不完整数据`呢？这就还需要判断了。
 - 如果`读取到的数据 % 数据固定长度(8)`余数为 0，则读取到的数据为`N 条完整数据`；如果不为 0，则为`N 条完整数据 + 一部分不完整数据`。


RocketMQ 业务逻辑是，只取得`最后一条完整数据`。所以使用下面的代码来完成的：
```
// 取得最后一条完整数据的尾部位置
int pos = this.byteBufferRead.position() - (this.byteBufferRead.position() % 8);
// 从尾部向前读取 8 个字节
long readOffset = this.byteBufferRead.getLong(pos - 8);
```

固定长度的逻辑就写完了，下面看一下 HAClient 中的粘包处理


###（2）HAClient#dispatchReadRequest

HAClient 接收的数据格式如下：
> Header + BodySize + BodyBytes 
> 数据长主为：8 + 4 + BodySize(BodySize 的值记录着 BodyBytes 的长度)。

需要消息整体长度不是固定的，但 Header 和 BodySize 长度是固定的，通过 BodySize 可以取得 BodyBytes 的长度。

**前期准备：**
由于每个消息的长度不是固定的，所以这里定义了 2 个 ByteBuffer 变量来接收数据，变量名字为：byteBufferRead 和 byteBufferBackup。

一般情况下使用 byteBufferRead，什么情况下使用 byteBufferBackup呢？当 byteBufferRead 的剩余空间不够存储一条消息时，就把 byteBufferRead 中读取完的最后一条消息的尾部位置开始，到 byteBufferRead 尾部位置的空间，复制到 byteBufferBackup 中，然后再交换 byteBufferRead 和 byteBufferBackup 的引用。
例如：byteBufferRead 整个大小为 100，存储内容大小为 96，最后一条数据的位置为 84，下一条数据总长度为 30。这样，如果要保存下最后一条数据的话，需要 byteBufferRead 长度为 114（84+30）。因为 byteBufferRead 长度为 100，不够存储下一条数据，然后就从`最后一条数据的位置(84)`开始，到 100 位置的数据复制到 byteBufferBackup 的 0 开始的位置，然后再从网络缓冲区内读取`下一条数据的剩余数据`，直到读取到（30-12）这么多数据为止，算是读取到下一条完整数据。

**处理：**
因为 Header 和 BodySize 长度是固定的，所以我们每次首先要读取到 Header + BodySize = 12 这么长的数据。如果读取不到就再次从 socketChannel 读取，直到读取到这长的数据才能继续处理。

在读取到 Header + BodySize 后，看看 byteBufferRead 中读取到的数据长度够不够下一条数据的总长度（Header + BodySize + BodySize值）。

- 如果够的话，就从上一条数据的尾部（dispatchPostion）开始读取数据进行处理。然后把处理的最后一条数据的尾部指针，记录到 dispatchPostion 里。然后进行读取下一条消息，直到没有数据可以被处理。
- 如果不够的话，再多 socketChannel 中读取数据到 byteBufferRead 中，直到数据够被处理。
 - 如果在读取数据时，byteBufferRead 不够存储 BodyBytes 那么长的数据的话，就利用 byteBufferBackup 进行移动数据（请参考`前期准备`中的说明）。

###粘包总结
以上就是 RocketMQ 在处理`固定长度`和`部分固定长度`数据时的做法，没有这方面经验的人，可以参考学习一下。

在 byteBufferBackup 的使用上，感觉使用不使用这个中间变量，使用 rewind 等方法也能实现。没有仔细考虑做法的不同和性能上的问题。



##4，错误处理
在一些死循环处理的地方，都有一些错误处理来跳出循环。
（1）HAClient#reportSlaveMaxOffset
```
for (int i = 0; i < 3 && this.reportOffset.hasRemaining(); i++) {
    try {
        this.socketChannel.write(this.reportOffset);
    } catch (IOException e) {
        log.error(this.getServiceName()
            + "reportSlaveMaxOffset this.socketChannel.write exception", e);
        return false;
    }
}
```
如果写 3 次都没有写完，就算失败跳出循环。网上的一般例子，是 `while(reportOffset.hasRemaining())` 这种写完。

（2）WriteSocketService#transferData
```
while (this.byteBufferHeader.hasRemaining()) {
    int writeSize = this.socketChannel.write(this.byteBufferHeader);
    if (writeSize > 0) {
        writeSizeZeroTimes = 0;
        this.lastWriteTimestamp = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now();
    } else if (writeSize == 0) {
        // 当缓冲区满了时，写返回的 size 就为 0
        // 如果 3 次后还是 0，可能说明缓冲区紧张，就等一会再写。
        if (++writeSizeZeroTimes >= 3) {
            break;
        }
    } else {
        throw new Exception("ha master write header error < 0");
    }
}
```
向缓冲区写 3 次都没写进去的话，就跳出循环。

（3）GroupTransferService#doWaitTransfer
```
private void doWaitTransfer() {
    if (!this.requestsRead.isEmpty()) {
        for (CommitLog.GroupCommitRequest req : this.requestsRead) {
            boolean transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
            for (int i = 0; !transferOK && i < 5; i++) {
                this.notifyTransferObject.waitForRunning(1000);
                transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
            }

            if (!transferOK) {
                log.warn("transfer messsage to slave timeout, " + req.getNextOffset());
            }

            req.wakeupCustomer(transferOK);
        }

        this.requestsRead.clear();
    }
}
```
进行 5 次 offset 比较，都不成功就算失败。


##5，关于同步到 Slave 的比较方法
在判断是否消息已经同步到 Slave 成功时，是通过比较是否 `Slave 传回来的 offset`>= `当前等待的消息的 offset`。但取得`Slave 传回来的 offset`时，有可能一次接收到 Slave 传回来的多个 offset，在这种情况下，Slave 只取得最后一个 offset，其它 offset 都不需要。因为最后一个 offset 代表传过来的数据中最大的 offset，如果大的 offset 消息都保存成功了，就代表比这个 offset 小的消息都保存成功了。这样就不需要拿 Slave 传回来的一个个 offset 进行比较了。


##6，关于 selector
RemotingUtil#openSelector 方法是取得 selector 的方法。代码如下：
```
public static Selector openSelector() throws IOException {
    Selector result = null;

    if (isLinuxPlatform()) {
        try {
            final Class<?> providerClazz = Class.forName("sun.nio.ch.EPollSelectorProvider");
            if (providerClazz != null) {
                try {
                    final Method method = providerClazz.getMethod("provider");
                    if (method != null) {
                        final SelectorProvider selectorProvider = (SelectorProvider) method.invoke(null);
                        if (selectorProvider != null) {
                            result = selectorProvider.openSelector();
                        }
                    }
                } catch (final Exception e) {
                    log.warn("Open ePoll Selector for linux platform exception", e);
                }
            }
        } catch (final Exception e) {
            // ignore
        }
    }

    if (result == null) {
        result = Selector.open();
    }

    return result;
}
```
从代码中可以看出，如果是 Linux 平台的话，就使用`sun.nio.ch.EPollSelectorProvider`。但看了下 jdk8 的 DefaultSelectorProvider 类的代码，也做了相同的事，可能是在其它部分代码上有和 jdk 不同的作法吧。
```
src.solaris.classes.sun.nio.ch.DefaultSelectorProvider.java

public static SelectorProvider create() {
    String osname = AccessController
        .doPrivileged(new GetPropertyAction("os.name"));
    if (osname.equals("SunOS"))
        return createProvider("sun.nio.ch.DevPollSelectorProvider");
    if (osname.equals("Linux"))
        return createProvider("sun.nio.ch.EPollSelectorProvider");
    return new sun.nio.ch.PollSelectorProvider();
}
```

##7，关于 SocketChannel#write 返回 0 的问题
WriteSocketService 在写数据时，有判断 write 返回值是否为 0。产生 0 的原因是因为“缓冲区”满了，无法再进行写了。为了解决这个问题，可以一直写，还可以像代码里一样 select(1000) 一下，因为“write socketChannel”已经向 selector 注册了 OP_WRITE 事件，所以如果缓冲区有空间，可以写数据了，就会立刻返回；如果不能写数据，就等待 1 秒后返回。这样就不用一直去尝试写，可以让程序节省 cpu 1 秒的使用。


##8，几个 TCP 参数
###（1）serverSocketChannel.socket().setReuseAddress(true)
如果端口忙，但TCP状态位于 TIME_WAIT ，可以重用 端口。如果端口忙，而TCP状态位于其他状态，重用端口时依旧得到一个错误信息， 抛出“Address already in use： JVM_Bind”。如果你的服务程序停止后想立即重启，不等60秒，而新套接字依旧 使用同一端口，此时 SO_REUSEADDR 选项非常有用。必须意识到，此时任何非期 望数据到达，都可能导致服务程序反应混乱，不过这只是一种可能，事实上很不可能。 

###（2）this.socketChannel.socket().setSoLinger(false, -1)

这个Socket选项可以影响close方法的行为。在默认情况下，当调用close方法后，将立即返回；如果这时仍然有未被送出的数据包，那么这些数据包将被丢弃。如果将linger参数设为一个正整数n时（n的值最大是65，535），在调用close方法后，将最多被阻塞n秒。在这n秒内，系统将尽量将未送出的数据包发送出去；如果超过了n秒，如果还有未发送的数据包，这些数据包将全部被丢弃；而close方法会立即返回。如果将linger设为0，和关闭SO_LINGER选项的作用是一样的。

如果底层的Socket实现不支持SO_LINGER都会抛出SocketException例外。当给linger参数传递负数值时，setSoLinger还会抛出一个IllegalArgumentException例外。可以通过getSoLinger方法得到延迟关闭的时间，如果返回-1，则表明SO_LINGER是关闭的。

参考：[Java Socket 几个重要的TCP/IP选项解析(一)](http://elf8848.iteye.com/blog/1739598)

##9，关于 Selector.select(int timeout) 的使用
在 HAClient、WriteSocketService、ReadSocketService、AcceptSocketService 几个类里使用 select 方法时都带了 timeout 参数。为什么要带这个参数呢？如果没有数据就一直等待这样不好吗？

###（1）在 HAClient 里使用
HAClient 在一方法里做的`和 Master 的 读/写`操作，但它在 Selector 上只注册了 READ 事件。所以如果不设置超时的话，是无法向 Master 写数据的。这里的`写数据`主要是指发心跳操作。

###（2）在 WriteSocketService、ReadSocketService、AcceptSocketService 里使用
这两个类里使用主要是为了安全停机。因为在 Master 正常关闭时，会告诉其它 service 要停止处理，其它 service 检测到后（例如：while (!this.isStopped())）要做一些后续处理。如果 select() 不使用超时的话，就会一直在等待，无法进行这些处理。

##10，如果 Master 发送 3 个消息数据到 slave，第 2 个消息落盘失败了，master 如何处理？
Master 在发送数据时，是发送一个 offset 开始的所有 CommitLog 中的数据。也就是说，如果 3 条消息一起被发送到了 Slave，要么全成功，要么全不成功。成功的话，Slave 会把当前 commit log offset 发回来。

##11，rocketmq 同步或异步结构
RocketMQ 的 Slave 是同步或异步保存，是需要修改 broker 配置的。Kafka 和 ActiveMQ 没有这种结构。

##12，各种 Master 和 Slave 成功和失败的情况总结


| 条件 | 结果 |
| -- | -- |
| Slave 保存成功，回传 offset 因为网络问题丢失 | 消息同步等待机制，在没有其它消息继续传时候，会最多等待 5 秒。一般来说 5 秒之内 TCP 重传机制已经把丢失的数据重传了。如果有其它消息继续传，那其因为它消息返回的 offset 也可以确认当前消息。|
| Slave 保存成功，回传 offset 时失败，失败后 Master 挂了 | 这种情况下 cosnumer 变成从 Slave 消费了，可能会消费到`发送失败的消息`。|
| Slave 保存成功后，Slave 机器机器挂掉 | 这种情况下，如果 Slave 不能在短时间内启动起来，那消息同步就失败。但从现在实现来看，就算 Slave 同步失败，也算消息保存成功。|
| Master 发送`消息1`和`消息2`，Slave 保存`消息1`失败 | Slave 在接收`消息2`时也会失败，并且关闭连接，因为在接收时有一个`slave current offset == master message offset`的判断。关闭连接是处理数据不一致的很简单有效的办法，因为重新创建连接后，Slave 会告诉 Master 自己`当前的 offset`，让 Master 从这个 offset 开始传数据 |
| Slave 保存 CommitLog  成功，ConsumeQueue 失败 | Slave 启动后，会通过 CommitLog 恢复 ConsumeQueue 和 Index 的数据。在恢复完后，Slave 把自己的最大 offset 告诉 Master，Master 向 Slave 传消息 |



##13，producer 和 consumer 如何选择
###（1）Producer
Producer 只会向 Master 发送消息，而且是对多个 Master 进行`轮询发送`。

###（2）Consumer
从程序上来看，一般情况下都是从 Master 拉取消息。有两种情况，从 Slave 拉取消息：

- Master 挂了
- Master 建议 consumer 从 Slave 拉取

而 Slave 定期从 Master 取得`consumer 消费到的 offset`（具体看 SlaveSynchronize#syncAll 方法）。（如果同步的时间比较长，可能会造成消息的重复消费）
具体代码和说明请参看：[RocketMQ源码分析----HA相关(1)](https://blog.csdn.net/u013160932/article/details/80299544)








#问题
1，而 Slave 定期从 Master 取得`consumer 消费到的 offset`（具体看 SlaveSynchronize#syncAll 方法）。（如果同步的时间比较长，可能会造成消息的重复消费）



#改进
1，在 consumer 批量消费失败时，可不可以动态把批量处理变小。例如，每次处理 100 条消息，其中第 30 条消息处理失败，这样 100 条消息都要退回 broker。下次再进行处理时，这条消息又在前面的位置，又处理失败。
可不可以，在处理失败后，把消费条数改成`原来条数/2`（或其它条数），这样可以减少失败影响的条数。（确认一下，消息退回到 broker 后，只有失败那条消息进入重试队列，还是这批消息都进入重试队列）	

2，问题。如果 Master 挂了，Slave 启动的话，应该如何做。

- 数据同步问题。现在只是 Master 向 Slave 同步数据。Master 的数据只有一份，一旦 Master 挂了，就可能有数据没有同步到 Slave 上。Slave 当主后，因为 Master 没有办法启动，所以无法同步 Master 中的数据。
- CommitLog 一致问题。当 Master 挂掉后，Master 上可能保存了消息，但这条消息没有同步到 Slave 上。如果 Slave 启动当主接收消息的话，那么 Master 和 Slave 的 CommitLog 内容会不一致。Master 再启动时，就要和 Slave 进行数据同步处理。

#参考

- [RocketMQ 源码分析 —— 高可用](http://www.iocoder.cn/RocketMQ/high-availability/)
- [源码研究RocketMQ主从同步机制(HA)](https://blog.csdn.net/prestigeding/article/details/79600792)
- [RocketMQ源码分析----HA相关(1)](https://blog.csdn.net/u013160932/article/details/80299544)
- [深入浅出NIO之Selector实现原理](https://www.jianshu.com/p/0d497fe5484a)：都是使用 NIO，AcceptSocketService 和 HAClient 使用 selector 的方式不一样，看一下使用方式的不同的原因。
- 关于 write 返回 0: https://blog.csdn.net/a34140974/article/details/48464845

<br>

=================== 过程 ===================
OK 1，this.selector.select(1000); 作用是什么？
2，`AtomicReference<String> masterAddress` 为什么要使用 AtomicReference 类型？
OK 3，client 启动后，是和 master 进行连接，还是 slave 进行连接？
OK 4，Class.forName("sun.nio.ch.EPollSelectorProvider"); 为什么要这么做？取得的是 Linux 的 selector 吗？为什么这个类找不到。
OK 5，为什么要设置两回 position.
```
this.reportOffset.position(0);
            this.reportOffset.limit(8);
            this.reportOffset.putLong(maxOffset);
            this.reportOffset.position(0);
            this.reportOffset.limit(8);
```
OK 6，this.serverSocketChannel.socket().setReuseAddress(true); reuseAddress 是什么意思？
OK 7，this.socketChannel.socket().setSoLinger(false, -1) soLinger 是什么意思？

OK 9，slave 向 master 传 offset 时，不管 master 是否接收到，继续传吗？master 没有接收到 slave 的信息，也继续向 slave 传 offset 吗？他们之间数据有没有同步操作？


OK 10，reallocateByteBuffer 方法里是使用两块 ByteBuffer 进行交互。因为这个写数据整个操作是同步的，所以可不可以使用一块 ByteBuffer，数据不够时使用 compact 方法，把数据前移？

OK 11，为什么要调用两次 reportSlaveMaxOffsetPlus，在 run 方法里调用的目的是什么？

OK 12，this.thread.setDaemon(true); 守护线程的作用是什么了？生成它的线程关了，它自己就关闭？

OK 13，Slave 时候，AcceptSocketService 和 Master 那几个类启动不启动？

OK 14，会不会发生在 master 向 slave 写数据时，写了一半 master 发生问题，而 slave 只接收到了一半的数据。因为使用的是 NIO，所以数据在写的时候，有可能一个 10 字节的数据分两次发送（可能是系统缓存区满只写了一部分，结果先写进去那一部分就被发送出去了）？


OK 16，AcceptSocketService 为什么要设置成“每秒中断一次”？一直等待连接请求到来，有什么问题吗？
select() 时，服务器连接断了，会有什么影响？会不会一直停在那，如果是那样的话，就需要 timeout 了。


OK 17，HAClient 为什么 selector(1000)。一直等待数据不行吗？有数据进来后自然就继续向下走了。是不是因为如果不中断，就一直等到缓冲区满了才返回吗？

OK 18， rocketmq 同步或异步是需要修改 broker 配置的，kafka 和 amq 需要不需要？
OK todo 向 slave 同步数据是有 timeout 的。如果 CommitLog 类中同步数据时，如果超时了，但 Slave 那边却成功了，有没有这种情况发生？如果有的话，消费者从 slave 消费消息的话，是不是会消费到发送失败的消息。
（RocketMQ 消费者是如何消费的？从 master 还是 slave。看 SendMessageProcessor 处理，同步 slave 失败也算发送成功）
OK todo master 和 slave 不停发送心跳，使用心跳做什么处理了吗？例如多长时间没有心跳的话，就断开连接什么的？

