todo



#零、基础类介绍
##1，Message
消息的基础类，我们传递的消息的内容，放到这个类的 body 属性里。
```
public class Message implements Serializable {
    private static final long serialVersionUID = 8445773977080406428L;
    // topic 名
    private String topic;
    // 完全由应用来设置，RocketMQ 不做干预(3.2.4 )
    private int flag;
    // 保存 message 中的一些属性。例如，tag 或其它的一些属性。这个属性的使用有两点要注意一下：
    // 1，有些地方，把这个属性的拼接成一个长字符串后，做一些操作。可能是因为 map 在序列化时兼容问题。
    // (使用 MessageDecoder.messageProperties2String 方法来生成字符串。)
    // 2，使用 map 可以解决兼容性问题。如果不使用 map，而使用 java 具体的属性，在 message 类加入新属性时，在序列化时会出现问题，不知道应该序列化成哪个版本的 Message。如果想解决，可能需要自己定义 Message 类的协议，例如，byte 前两位是 Message 的版本号。
    private Map<String, String> properties;
    // 具体的消息内容
    private byte[] body;
}
```

##2，TopicPublishInfo
Topic 的路由信息类
```
public class TopicPublishInfo {
    private boolean orderTopic = false;
    // 是否具有路由信息。看是否可用时，有的地方还使用它的 ok() 方法来判断。因为在 updateTopicRouteInfoFromNameServer 方法里设置 TopicPublishInfo 时，它没有判断 messageQueueList 就直接设置了 haveTopicRouterInfo，所以还需要使用 ok() 方法判断是否可用。
    private boolean haveTopicRouterInfo = false;
    // 保存 topic 相关的 MessageQueue 的信息。
    private List<MessageQueue> messageQueueList = new ArrayList<MessageQueue>();
    // 在轮循时，使用的 index 类。
    private volatile ThreadLocalIndex sendWhichQueue = new ThreadLocalIndex();
    // 这个信息是所有的 queue 和 broker 信息。从这个信息，取得 topic 想要的信息。
    private TopicRouteData topicRouteData;
}
```

##3，MeesageQueue
像 kafka 中的 partition，即一个 Topic 被分成多少个 Queue。
```
public class MessageQueue implements Comparable<MessageQueue>, Serializable {
    private static final long serialVersionUID = 6191200464116433425L;
    // MessageQueue 对应的 Topic 名
    private String topic;
    // MessageQueue 所在的的 Broker 名
    private String brokerName;
    // MessageQueue 的 ID
    private int queueId;
}
```
##4，ThreadLocalIndex
一个挺有用的工具类，主要是用做循环时，保证轮循次数的线程安全问题。不知道为什么有几个地方会判断`是否小于0`，从代码上来看感觉不太可能出现负数。
```
public class ThreadLocalIndex {
    private final ThreadLocal<Integer> threadLocalIndex = new ThreadLocal<Integer>();
    private final Random random = new Random();

    public int getAndIncrement() {
        Integer index = this.threadLocalIndex.get();
        if (null == index) {
            index = Math.abs(random.nextInt());
            if (index < 0)
                index = 0;
            this.threadLocalIndex.set(index);
        }

        index = Math.abs(index + 1);
        if (index < 0)
            index = 0;

        this.threadLocalIndex.set(index);
        return index;
    }

    @Override
    public String toString() {
        return "ThreadLocalIndex{" +
            "threadLocalIndex=" + threadLocalIndex.get() +
            '}';
    }
}
```
##5，TopicRouteData
Client（Producer和Consumer）和 NameServer 交互 Topic 路由信息的原始类型。
```
public class TopicRouteData extends RemotingSerializable {
    // TODO Q：作用是什么？
    private String orderTopicConf;
    // Topic 相关的所有 Queue 的信息
    private List<QueueData> queueDatas;
    // Topic 相关的所有的 broker 信息
    private List<BrokerData> brokerDatas;
    // filterServer 信息
    private HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
}

public class QueueData implements Comparable<QueueData> {
    // broker 名
    private String brokerName;
    // TODO Q：作用是什么？
    private int readQueueNums;
    // TODO Q：作用是什么？
    private int writeQueueNums;
    // TODO Q：作用是什么？
    private int perm;
    // TODO Q：作用是什么？
    private int topicSynFlag;
}

public class BrokerData implements Comparable<BrokerData> {
    private String cluster;
    // broker 名
    private String brokerName;
    // broker 的地址
    // TODO Q：多个是怎么回事？是 Master/Slave 的地址吗？
    private HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs;
}
    
```
##6，DefaultMQProducer
这个是 Producer 的实现类，一个 Producer 对应一个 DefaultMQProducer。它主要是保存一些发送的基础信息：

- producerGroup
- Topic 对应的 Queue 的默认数量
- 发送超时时间
- 失败次数
- 重试次数等。

它继承了 ClientConfig。ClientConfig 主要保存一些 Client 的设置信息（注意，这些信息对应所有的 producer。如果是对应某一个 producer 的信息的话，就放到 DefaultMQProducer 里了），例如：

- name server 地址
- 本机 IP 地址
- 应用实例的名字
- 发送心跳时间间隔
- 处理 callback 的线程数



##7，DefaultMQProducerImpl
这个类处理的内容相对 DefaultMQProducer，更具体一些。它和 DefaultMQProducer 也是一对一的关系，而且包含生成它的 DefaultMQProducer 的引用。它主要包含以下信息：

- 所有 topic 的 TopicPublishInfo 信息
- 一些发送时的前置/后置方法
- 消息体压缩信息
- MQClientInstance 实例（发送的核心实例)

##8，MQClientInstance
是一个发送消息的核心类。从内容来看，管理所有的 producer、consumer、MQClientAPIImpl、MQAdminImpl、所有 Topic 的路由信息等。生成这个类的实例，是使用 MQClientManager 生成的，从生成的方法来看，应该是每个应用“共用”一个 MQClientInstance，所以 MQClientInstance 要保证线程安全。
（在 MQClientManager 中，是按`IP + “@” + 进程号`(例如：172.23.1.236@7260) 来区分的。从前面的区分条件来看，一个进程里启动多个 producer 的话，`IP + “@” + 进程号`始终是相同的。如果 2 个 producer 在不同的进程里的话，肯定不会放到一个对象里。这个是不是不可能有两个 instance 存在？）

##9，MQClientAPIImpl
这个类是负责信息的网络通信的类，内部是用 netty 实现的。所有 producer 都是用这一个类发进行发送的？


**类图如下：**
![这里写图片描述](https://img-blog.csdn.net/20180503081120650?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


#一、调用方代码
```
public static void main(String[] args) throws MQClientException, InterruptedException {
    // 1，生成 producer
    DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
    // 2，生成消息
    Message msg = new Message("TopicTest",
            "TagA",
            ("Hello RocketMQ ").getBytes(RemotingHelper.DEFAULT_CHARSET)
    );
    // 发送消息
    SendResult sendResult = producer.send(msg);
    System.out.printf("%s%n", sendResult);
    // 关闭 producer
    producer.shutdown();
}
```


从上面的发送代码来看，可以看出来`DefaultMQProducer`是我们分析的入口。但 DefaultMQProducer 只是一个包装类，里面执行的全都是 DefaultMQProducerImpl 的方法，所以我们直接从 DefaultMQProducerImpl 类进行分析。

#二、启动
##1，DefaultMQProducerImpl#Start
因为例子中，先要进行 start 所以先分析这个方法。这个方法主要做以下工作：

- 修改当前 producer 运行状态为 START_FAILED（防止多次初始化）。
- 对 producer group 名字进行各种 check。
- 修改 producer 名字为`进程号`。
- 生成一个 MQClientInstance（从内容来看，是管理所有的信息的，例如：producer、consumer、工具类等，进程内所有 producer 和 cosnumer 都用这一个实例）
- 把当前 producer 保存到 MQClientInstance 中。保存时，每个一 producer group 对应一个 producer。
- 添加一个默认的 topic 信息，到 topic info 集合里。
- 启动 MQClientInstance
- 把运行状态修改成 RUNNING。
- 向所有 broker 发送心跳。


```
public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            this.serviceState = ServiceState.START_FAILED;

            // 对 producer group 名字进行 check
            this.checkConfig();

            if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                // 修改 instance(就是Producer) 的名字。
                // 先从系统属性里取，看看设置没有设置名字。
                // 如果没有设置名字，默认是"DEFAULT"，要修改成当前进程 PID。
                this.defaultMQProducer.changeInstanceNameToPID();
            }

            // 生成一个 MQClientInstance
            // TODO: 2018/4/25 这个 MQClientInstance 是干什么用的呢？
            this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);

            // 把 producer 保存到 mQClientFactory 中。保存时，每个一 producer group 对应
            // 一个 producer
            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }

            // 添加一个 topic 信息
            // TODO Q: 2018/4/25 为什么要添加这个，不添加行吗？
            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

            // 为了防止死循环调用。
            // 有一些地方调用 mQClientFactory.start()，而不调用 DefaultMQProducerImpl.start
            // 而有一些地方调用 DefaultMQProducerImpl.start ，而不调用 DefaultMQProducerImpl.start
            // 但两者都是要 start 的，所以要互相调用 start。为了防止死循环调用，用了下面的判断。
            if (startFactory) {
                mQClientFactory.start();
            }

            log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                this.defaultMQProducer.isSendMessageWithVIPChannel());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The producer service state not OK, maybe started once, "//
                + this.serviceState//
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                null);
        default:
            break;
    }

    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
}
```

**细节：**
1，为什么要上来把 serviceState 修改成`START_FAILED`，为了防止多次初始化，在 case 后面会把状态修改成 running。
2，在启动 MQClientInstance 时，要先判断，为什么判断呢？为了防止死循环调用。
有一些地方调用 mQClientFactory.start()，而不调用 DefaultMQProducerImpl.start。而有一些地方调用 DefaultMQProducerImpl.start ，而不调用 DefaultMQProducerImpl.start。但两者都是要 start 的，所以要互相调用 start。为了防止死循环调用，用了判断。


##1.1，MQClientInstance#Start
这个类从内容来看，是管理所有的信息的，例如：producer、consumer、工具类等。进程内所有 producer 和 cosnumer 都用这一个实例。主要工作如下：

- 修改当前 producer 运行状态为 START_FAILED（防止多次初始化）。
- 如果没有设置 name server 地址，就去指定的地址去取得 name server 的地址。（这个地址是阿里地址，所以如果没有设置 name server 地址就会报错。）
- 启动 MQClientAPIImpl。主要是启动 netty，准备进行数据传输
- 启动各种定时任务
- 启动 pull 任务，定时去 broker 取数据
- 启动 rebalance 任务
- 启动 defaultMQProducer（因为 defaultMQProducer 是必须启动的，所以这里启动一下）
- 把运行状态修改成 RUNNING。


```
public void start() throws MQClientException {

    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;

                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) {
                    // 如果没有设置 name server 地址，就从系统属性"rocketmq.namesrv.domain"
                    // 里取得地址，然后去连接 name server
                    // TODO Q: 2018/4/25 为什么要取得 namer server 地址，取回来的是什么样的？
                    this.clientConfig.setNamesrvAddr(this.mQClientAPIImpl.fetchNameServerAddr());
                }
                // 启动 MQClientAPIImpl。主要是启动 netty，准备进行数据传输。
                // Start request-response channel
                this.mQClientAPIImpl.start();
                // 启动各种定时任务
                // Start various schedule tasks
                this.startScheduledTask();
                // 启动 pull 任务，定时去 broker 取数据。
                // Start pull service
                this.pullMessageService.start();
                // todo 还没有细看。
                // Start rebalance service
                this.rebalanceService.start();
                // 启动 defaultMQProducer
                // Start push service
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
                break;
            case SHUTDOWN_ALREADY:
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```

##1.1.1，MQClientInstance#startScheduledTask
主要工作如下：

- 定时取 name server 地址（取到后，更新 Netty 发送对象保存的地址）。
- 定时更新 topic 路由信息。
- 定时去掉不在线的 broker
- 定时发送心跳 并且 上传filter类
- 上传 consumer offset

```
private void startScheduledTask() {
    if (null == this.clientConfig.getNamesrvAddr()) {
        // 每 2 分钟去 取 name server 地址。
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.mQClientAPIImpl.fetchNameServerAddr();
                } catch (Exception e) {
                    log.error("ScheduledTask fetchNameServerAddr exception", e);
                }
            }
        }, 1000 * 10, 1000, TimeUnit.MILLISECONDS);
//            }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
    }

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                // 更新 producer 和 consumer 中的 topic 路由信息
                MQClientInstance.this.updateTopicRouteInfoFromNameServer();
            } catch (Exception e) {
                log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
            }
        }
    }, 10, this.clientConfig.getPollNameServerInteval(), TimeUnit.MILLISECONDS);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                // 去掉不在线的 broker
                MQClientInstance.this.cleanOfflineBroker();
                // 发送心跳 并且 上传filter类
                MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
            } catch (Exception e) {
                log.error("ScheduledTask sendHeartbeatToAllBroker exception", e);
            }
        }
    }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                // 上传 consumer offset
                MQClientInstance.this.persistAllConsumerOffset();
            } catch (Exception e) {
                log.error("ScheduledTask persistAllConsumerOffset exception", e);
            }
        }
    }, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                // TODO Q: 2018/4/25 为什么要调整线程池
                MQClientInstance.this.adjustThreadPool();
            } catch (Exception e) {
                log.error("ScheduledTask adjustThreadPool exception", e);
            }
        }
    }, 1, 1, TimeUnit.MINUTES);
}
```

#三、发送
调用的时序图请参看：http://www.iocoder.cn/RocketMQ/message-send-and-receive/

##2，DefaultMQProducerImpl#sendDefaultImpl
这个方法是实际的发送功能的一部分，发送功能是多层的调用，每一层都实现了不同的功能。而这一层主要是做了`发送 Topic 信息的准备`、`MessageQueue的选择`和`失败重发`。下面我们看一下这个方法的功能：

- 发送前 check
 - 确认是否是 RUNNING 状态
 - 对发送内容进行 check
- 取得 topic 相关的 TopicPublishInfo。每个 topic 第一次发送时，都去 name server 取得一下。
- 如果是同步发送，就取得发送重试次数
- 进行发送
 - 选出一个 MessageQueue，向它发送消息（在发送次数限制内，每次发送失败，都再选择一次）
 - 调用发送消息的实际方法 (sendKernelImpl)
 - 发送完成后，更新接收消息的 broker 的状态。在上面的 selectOneMessageQueue 选择MessageQueue 时，主要参考这个更新的内容。而更新的内容是"发送花费的总时间"
 - 如果发送失败，判断是设置了“换一个 broker 重新发送”。如果设置了，就重新发送一次。如果没有，就返回发送结果。（这个可能是为了保证消息顺序）
 

```
private SendResult sendDefaultImpl(//
    Message msg, //
    final CommunicationMode communicationMode, //
    final SendCallback sendCallback, //
    final long timeout//
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // 确认是否是 RUNNING 状态
    this.makeSureStateOK();
    // 对发送内容进行 check
    Validators.checkMessage(msg, this.defaultMQProducer);

    final long invokeID = random.nextLong();
    long beginTimestampFirst = System.currentTimeMillis();
    long beginTimestampPrev = beginTimestampFirst;
    long endTimestamp = beginTimestampFirst;
    // 取得 topic 相关的 TopicPublishInfo。每个 topic 第一次发送时，都去 name server 取得一下。
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        MessageQueue mq = null;
        Exception exception = null;
        SendResult sendResult = null;
        // 如果是同步发送，就取得发送重试次数
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
        // 具体的发送次数
        int times = 0;
        String[] brokersSent = new String[timesTotal];
        for (; times < timesTotal; times++) {
            String lastBrokerName = null == mq ? null : mq.getBrokerName();
            // 选出一个 MessageQueue，向它发送消息（在发送次数限制内，每次发送失败，都再选择一次）
            MessageQueue tmpmq = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            if (tmpmq != null) {
                mq = tmpmq;
                // 记录发送的 MessageQueue 的 broker 名字
                brokersSent[times] = mq.getBrokerName();
                try {
                    beginTimestampPrev = System.currentTimeMillis();
                    // 调用发送消息的实际方法
                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);
                    endTimestamp = System.currentTimeMillis();
                    // 发送完成后，更新接收消息的 broker 的状态。在上面的 selectOneMessageQueue 选择
                    // MessageQueue 时，主要参考这个更新的内容。而更新的内容是"发送花费的总时间"
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    switch (communicationMode) {
                        case ASYNC:
                            return null;
                        case ONEWAY:
                            return null;
                        case SYNC:
                            // 发送失败时候，判断是否要换一个 broker 发送。
                            // 如果是，就把上面的过程再做一次；不是，就直接返回结果。
                            // 这个可能是为了保证消息顺序
                            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                    continue;
                                }
                            }

                            return sendResult;
                        default:
                            break;
                    }
                } catch (RemotingException e) {
                   ...
                } catch (MQClientException e) {
                   ...
                } catch (MQBrokerException e) {
                   ...
                } catch (InterruptedException e) {
                   ...
                }
            } else {
                break;
            }
        }

        // 超过发送次数限制后，看发送结果是否为 null，如果不是就返回结果。
        if (sendResult != null) {
            return sendResult;
        }

        // 写 log，并抛出 exception
        String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s",
            times,
            System.currentTimeMillis() - beginTimestampFirst,
            msg.getTopic(),
            Arrays.toString(brokersSent));

        info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

        MQClientException mqClientException = new MQClientException(info, exception);
        if (exception instanceof MQBrokerException) {
            mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
        } else if (exception instanceof RemotingConnectException) {
            mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);
        } else if (exception instanceof RemotingTimeoutException) {
            mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);
        } else if (exception instanceof MQClientException) {
            mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);
        }

        throw mqClientException;
    }

    // 如果走到这里，说明没有取得 topic 相关路由信息
    // 下面就是抛出 exception
    List<String> nsList = this.getmQClientFactory().getMQClientAPIImpl().getNameServerAddressList();
    if (null == nsList || nsList.isEmpty()) {
        throw new MQClientException(
            "No name server address, please set it." + FAQUrl.suggestTodo(FAQUrl.NAME_SERVER_ADDR_NOT_EXIST_URL), null).setResponseCode(ClientErrorCode.NO_NAME_SERVER_EXCEPTION);
    }

    throw new MQClientException("No route info of this topic, " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
        null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
}
```

##2.1，MQClientInstance#tryToFindTopicPublishInfo
处理内容如下：

- 从 TopicPublishInfo 集合里取得 topic 的 TopicPublishInfo
- 如果取不到 或 TopicPublishInfo 状态不对，就从 name server 取得 TopicPublishInfo。
- 如果从 name server 取得的 TopicPublishInfo 正常的话，就返回 TopicPublishInfo。如果还是不正常的话，就取得"默认Topic(TBW102)"的路由信息，向这个 topic 发送。
```
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    // 从 TopicPublishInfo 集合里取得 topic 的 TopicPublishInfo
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    // 如果取不到 或 TopicPublishInfo 状态不对，就从 name server 取得 TopicPublishInfo
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        // 这里为什么放一个空的 TopicPublishInfo 进去？
        // 可能是为了下面的 else，else 再从 name server 取得信息后，直接从集合里取出然后返回了，没有判断。
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }

    // 如果取得的 TopicPublishInfo 正常的话，就返回 TopicPublishInfo。
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        // 如果还是不正常的话，就取得"默认Topic(TBW102)"的路由信息，向这个 topic 发送。
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

##2.1.1，MQClientInstance#updateTopicRouteInfoFromNameServer
这个方法是，从 name server 取得 topic 的路由信息。处理内容如下：

- 根据参数判断，如果是想要取得"默认Topic"信息的话，就去取得"默认Topic"的路由信息。否则取得相应的 topic 的路由信息。
- 如果取得路由信息成功，判断新的路由信息 和 之前保证保存的 是否有改变。如果有改变，就更新 Broker、Producer、Consumer 的信息，并把新取得的路由信息保存到 TopicPublishInfo 集合里。


```
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault, DefaultMQProducer defaultMQProducer) {
    try {
        if (this.lockNamesrv.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
            try {
                TopicRouteData topicRouteData;
                // 根据参数判断，如果是想要取得"默认Topic"信息的话，就去取得"默认Topic"的路由信息。
                if (isDefault && defaultMQProducer != null) {
                    topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),
                        1000 * 3);
                    if (topicRouteData != null) {
                        for (QueueData data : topicRouteData.getQueueDatas()) {
                            int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
                            data.setReadQueueNums(queueNums);
                            data.setWriteQueueNums(queueNums);
                        }
                    }
                } else {
                    // 否则取得相应的 topic 的路由信息
                    topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);
                }

                // 如果取得路由信息成功，判断新的路由信息 和 之前保证保存的 是否有改变。
                // 如果有改变，就更新 Broker、Producer、Consumer 的信息
                if (topicRouteData != null) {
                    TopicRouteData old = this.topicRouteTable.get(topic);
                    boolean changed = topicRouteDataIsChange(old, topicRouteData);
                    if (!changed) {
                        changed = this.isNeedUpdateTopicRouteInfo(topic);
                    } else {
                        log.info("the topic[{}] route info changed, old[{}] ,new[{}]", topic, old, topicRouteData);
                    }

                    if (changed) {
                        TopicRouteData cloneTopicRouteData = topicRouteData.cloneTopicRouteData();

                        for (BrokerData bd : topicRouteData.getBrokerDatas()) {
                            this.brokerAddrTable.put(bd.getBrokerName(), bd.getBrokerAddrs());
                        }

                        // Update Pub info
                        {
                            TopicPublishInfo publishInfo = topicRouteData2TopicPublishInfo(topic, topicRouteData);
                            publishInfo.setHaveTopicRouterInfo(true);
                            Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
                            while (it.hasNext()) {
                                Entry<String, MQProducerInner> entry = it.next();
                                MQProducerInner impl = entry.getValue();
                                if (impl != null) {
                                    impl.updateTopicPublishInfo(topic, publishInfo);
                                }
                            }
                        }

                        // Update sub info
                        {
                            Set<MessageQueue> subscribeInfo = topicRouteData2TopicSubscribeInfo(topic, topicRouteData);
                            Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
                            while (it.hasNext()) {
                                Entry<String, MQConsumerInner> entry = it.next();
                                MQConsumerInner impl = entry.getValue();
                                if (impl != null) {
                                    impl.updateTopicSubscribeInfo(topic, subscribeInfo);
                                }
                            }
                        }
                        log.info("topicRouteTable.put TopicRouteData[{}]", cloneTopicRouteData);
                        this.topicRouteTable.put(topic, cloneTopicRouteData);
                        return true;
                    }
                } else {
                    log.warn("updateTopicRouteInfoFromNameServer, getTopicRouteInfoFromNameServer return null, Topic: {}", topic);
                }
            } catch (Exception e) {
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX) && !topic.equals(MixAll.DEFAULT_TOPIC)) {
                    log.warn("updateTopicRouteInfoFromNameServer Exception", e);
                }
            } finally {
                this.lockNamesrv.unlock();
            }
        } else {
            log.warn("updateTopicRouteInfoFromNameServer tryLock timeout {}ms", LOCK_TIMEOUT_MILLIS);
        }
    } catch (InterruptedException e) {
        log.warn("updateTopicRouteInfoFromNameServer Exception", e);
    }

    return false;
}
```

##2.1.2，MQFaultStrategy#selectOneMessageQueue
MQClientInstance#selectOneMessageQueue 里实际上是调用的 MQFaultStrategy#selectOneMessageQueue 方法，这个方法的主要作用是：如果 FaultStrategy 生效了，就选择一个比较优的 MessageQueue 进行发送。如果没有生效，就“轮循选择”一个。处理内容如下：

- 判断 FaultStrategy 是否生效，如果生效
 - 就根据 MessageQueue 集合进行轮循发送。
 - 如果轮循发送时，MessageQueue 所在 Broker 不可用，或者发送时指定的 broker 不是当前 MessageQueue 的 broker
   - 就选择一个“比较好”的 broker 进行发送。“比较好”的算法：可用的、延迟小的。
   - 选择出来后，如果这个 broker 有“可写 Queue” 或 topic有相关的 MessageQueue 在这个 broker上，就“轮循选择”一个 MessageQueue。如果没有，就把这个 broker 从 FaultStrategy 里把这个 broker 去掉，以后不做为备选。

- 如果 FaultStrategy 不生效，就轮循选择一个 MessageQueue。

```
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    if (this.sendLatencyFaultEnable) {
        try {
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }

            // 选择一个比较好的 MessageQueue 返回。好的定义，请看 FaultItem compareTo 的实现。
            // 第一次发送时，pickOneAtLeast 返回 null。
            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            if (writeQueueNums > 0) {
                final MessageQueue mq = tpInfo.selectOneMessageQueue();
                if (notBestBroker != null) {
                    mq.setBrokerName(notBestBroker);
                    mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                }
                return mq;
            } else {
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }

        // 当 producer 第一次发送时，因为 pickOneAtLeast 返回 null 所以会走到这里。
        // TODO Q: 2018/4/24 发生异常时，也会走到这里。什么时候会发生异常呢？
        return tpInfo.selectOneMessageQueue();
    }

    return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```

##2.1.3，MQFaultStrategy#sendKernelImpl
这个也是核心发送方法之一，这个方法的主要工作是，`对消息进行处理`、`同步或异步方法的调用`和`调用前置和后置方法`。处理如下：

- 根据参数 MessageQueue，取得 broker 地址。
- 如果没有取得到 broker 地址，就取得 TopicPublishInfo 信息。然后再一次取得 broker 地址。
- 如果 broker 还是没有，就抛出异常。
- 如果取得到了 broker 地址
 - 如果打开了 broker vip 通道，就使用它。broker会开启两个端口对外服务。
 - 生成一个“唯一ID”
 - 如果消息超过了大小限制，就进行压缩
 - 设置事务类型
 - 进行 ForbiddenHook check （这个应该是发送前的 自定义check 处理，但没有见到哪里使用）
 - 如果有 SendMessageHook，执行 send before hook。（这个应该也是一种自定义发送前处理，和上面的类型可能不同）
 - 设置 request header，就是发送消息的请求的 header。
 - 根据发送 Mode（同步、异步） 进行发送。
 - 执行 send after hook。

```
private SendResult sendKernelImpl(final Message msg, //
    final MessageQueue mq, //
    final CommunicationMode communicationMode, //
    final SendCallback sendCallback, //
    final TopicPublishInfo topicPublishInfo, //
    final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // 根据参数 MessageQueue，取得 broker 地址
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    if (null == brokerAddr) {
        // 如果没有取得到 broker 地址，再取得一次
        tryToFindTopicPublishInfo(mq.getTopic());
        brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    }

    SendMessageContext context = null;
    if (brokerAddr != null) {
        // 是否使用broker vip通道。broker会开启两个端口对外服务。
        // TODO Q: 2018/4/24 vip 通道的好处是什么？
        brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

        // 记录消息内容。下面逻辑可能改变消息内容，例如消息压缩。
        byte[] prevBody = msg.getBody();
        try {

            // 设置 唯一ID
            MessageClientIDSetter.setUniqID(msg);

            int sysFlag = 0;
            // 如果超过限制大小，就对消息进行压缩
            if (this.tryToCompressMessage(msg)) {
                sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
            }

            // 事务处理
            final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
            if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
                sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
            }

            // TODO Q: 2018/4/24 这个判断是做什么用的
            if (hasCheckForbiddenHook()) {
                CheckForbiddenContext checkForbiddenContext = new CheckForbiddenContext();
                checkForbiddenContext.setNameSrvAddr(this.defaultMQProducer.getNamesrvAddr());
                checkForbiddenContext.setGroup(this.defaultMQProducer.getProducerGroup());
                checkForbiddenContext.setCommunicationMode(communicationMode);
                checkForbiddenContext.setBrokerAddr(brokerAddr);
                checkForbiddenContext.setMessage(msg);
                checkForbiddenContext.setMq(mq);
                checkForbiddenContext.setUnitMode(this.isUnitMode());
                this.executeCheckForbiddenHook(checkForbiddenContext);
            }

            // TODO Q: 2018/4/24 这个 sendMessageHook 是做什么用的？
            if (this.hasSendMessageHook()) {
                context = new SendMessageContext();
                context.setProducer(this);
                context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
                context.setCommunicationMode(communicationMode);
                context.setBornHost(this.defaultMQProducer.getClientIP());
                context.setBrokerAddr(brokerAddr);
                context.setMessage(msg);
                context.setMq(mq);
                String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
                if (isTrans != null && isTrans.equals("true")) {
                    context.setMsgType(MessageType.Trans_Msg_Half);
                }

                if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
                    context.setMsgType(MessageType.Delay_Msg);
                }
                this.executeSendMessageHookBefore(context);
            }

            // 设置 request header
            SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
            requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
            requestHeader.setTopic(msg.getTopic());
            requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
            requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
            requestHeader.setQueueId(mq.getQueueId());
            requestHeader.setSysFlag(sysFlag);
            requestHeader.setBornTimestamp(System.currentTimeMillis());
            requestHeader.setFlag(msg.getFlag());
            requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
            requestHeader.setReconsumeTimes(0);
            requestHeader.setUnitMode(this.isUnitMode());
            if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
                if (reconsumeTimes != null) {
                    requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
                }

                String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
                if (maxReconsumeTimes != null) {
                    requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
                }
            }

            SendResult sendResult = null;
            switch (communicationMode) {
                case ASYNC:
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(//
                        brokerAddr, // 1
                        mq.getBrokerName(), // 2
                        msg, // 3
                        requestHeader, // 4
                        timeout, // 5
                        communicationMode, // 6
                        sendCallback, // 7
                        topicPublishInfo, // 8
                        this.mQClientFactory, // 9
                        this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(), // 10
                        context, //
                        this);
                    break;
                case ONEWAY:
                case SYNC:
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        msg,
                        requestHeader,
                        timeout,
                        communicationMode,
                        context,
                        this);
                    break;
                default:
                    assert false;
                    break;
            }

            if (this.hasSendMessageHook()) {
                context.setSendResult(sendResult);
                this.executeSendMessageHookAfter(context);
            }

            return sendResult;
        } catch (RemotingException e) {
            if (this.hasSendMessageHook()) {
                context.setException(e);
                this.executeSendMessageHookAfter(context);
            }
            throw e;
        } catch (MQBrokerException e) {
            if (this.hasSendMessageHook()) {
                context.setException(e);
                this.executeSendMessageHookAfter(context);
            }
            throw e;
        } catch (InterruptedException e) {
            if (this.hasSendMessageHook()) {
                context.setException(e);
                this.executeSendMessageHookAfter(context);
            }
            throw e;
        } finally {
            // TODO Q: 2018/4/24 这个设置的作用是什么？
            msg.setBody(prevBody);
        }
    }

    throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
}
```

##2.1.3.1，MQClientAPIImpl#sendMessage
这个发送方法的功能是，`调整Header` 和 `调用进一步的发送方法`，这个算是对`同步`和`异步`方法的一个封装，在这个方法里调用具体的`同步`或`异步`方法。主要处理如下：

- 如果是设置 sendSmartMsg 条件，就使用 v2 版本的 header。v2 和 v1 相比，只是属性名字长度变短了，减少传输量。
- 把要发送的消息体内容，设置到 request 中
- 根据 通信Mode 进行调用同步、异步或 OneWay 发送方法
- 返回发送结果（只有同步发送返回，异步和 OneWay 都返回 null）

```
public SendResult sendMessage(//
    final String addr, // 1
    final String brokerName, // 2
    final Message msg, // 3
    final SendMessageRequestHeader requestHeader, // 4
    final long timeoutMillis, // 5
    final CommunicationMode communicationMode, // 6
    final SendCallback sendCallback, // 7
    final TopicPublishInfo topicPublishInfo, // 8
    final MQClientInstance instance, // 9
    final int retryTimesWhenSendFailed, // 10
    final SendMessageContext context, // 11
    final DefaultMQProducerImpl producer // 12
) throws RemotingException, MQBrokerException, InterruptedException {
    RemotingCommand request = null;

    // 如果是设置 sendSmartMsg 条件，就使用 v2 版本的 header。
    // v2 和 v1 相比，只是属性名字长度变短了，减少传输量。
    if (sendSmartMsg) {
        SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
        request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE_V2, requestHeaderV2);
    } else {
        request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);
    }

    // 把要发送的消息体内容，设置到 request 中
    request.setBody(msg.getBody());

    // 根据 通信Mode 进行调用同步、异步或 OneWay 方法
    switch (communicationMode) {
        case ONEWAY:
            this.remotingClient.invokeOneway(addr, request, timeoutMillis);
            return null;
        case ASYNC:
            final AtomicInteger times = new AtomicInteger();
            this.sendMessageAsync(addr, brokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance,
                retryTimesWhenSendFailed, times, context, producer);
            return null;
        case SYNC:
            // 同步发送
            return this.sendMessageSync(addr, brokerName, msg, timeoutMillis, request);
        default:
            assert false;
            break;
    }

    return null;
}
```

##2.1.3.1.1，MQClientAPIImpl#sendMessageSync
这个方法的主要功能就是：`调用同步发送`和`组装返回结果`。处理内容如下：

- 发送数据，得取得 RemotingCommand。
- 根据 RemotingCommand，组装 SendResult 并返回。
```
private SendResult sendMessageSync(//
    final String addr, //
    final String brokerName, //
    final Message msg, //
    final long timeoutMillis, //
    final RemotingCommand request//
) throws RemotingException, MQBrokerException, InterruptedException {
    // 调用同步发送接口
    RemotingCommand response = this.remotingClient.invokeSync(addr, request, timeoutMillis);
    assert response != null;
    // 组装 SendResult
    return this.processSendResponse(brokerName, msg, response);
}
```

##2.1.3.1.1.1，NettyRemotingClient#invokeSync
处理内容如下：

- 取得 netty channel
- 如果设置了 rpc 调用前置方法，就进行调用
- 调用发送接口
- 如果设置了 rpc 调用后置方法，就进行调用

```
@Override
public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
    throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
    // 取得 netty channel
    final Channel channel = this.getAndCreateChannel(addr);
    // 如果 channel 可以使用
    if (channel != null && channel.isActive()) {
        try {

            // 如果设置了 rpc 调用前置方法，就进行调用
            if (this.rpcHook != null) {
                this.rpcHook.doBeforeRequest(addr, request);
            }

            // 调用发送接口
            RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis);

            // 如果设置了 rpc 调用后置方法，就进行调用
            if (this.rpcHook != null) {
                this.rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
            }
            return response;
        } catch (RemotingSendRequestException e) {
            log.warn("invokeSync: send request exception, so close the channel[{}]", addr);
            this.closeChannel(addr, channel);
            throw e;
        } catch (RemotingTimeoutException e) {
            if (nettyClientConfig.isClientCloseSocketIfTimeout()) {
                this.closeChannel(addr, channel);
                log.warn("invokeSync: close socket because of timeout, {}ms, {}", timeoutMillis, addr);
            }
            log.warn("invokeSync: wait response timeout exception, the channel[{}]", addr);
            throw e;
        }
    } else {
        this.closeChannel(addr, channel);
        throw new RemotingConnectException(addr);
    }
}
```

##2.1.3.1.1.1.1，NettyRemotingClient#invokeSync
这个方法的作用是，进行具体的网络发送，和处理发送后的结果。处理内容如下：

- 新建一个 SendResult 对象
- 使用 channel 进行发送。因为是异步发送，发送无的回调如下：
 - 如果发送成功，就设置 responseFuture 为 OK 状态。
 - 如果不成功，就设置 responseFuture 为不 OK 状态。responseFuture 集合类里删除当前这个 responseFuture。（responseFuture 集合是 NettyRemotingClient 的一个属性，用来管理所有发送的状态）
- 在异步发送后时，设置了一个超时时间。如果异步调用完成了 或者 过了超时时间，就断续向下走。
- 查看 responseFuture 里的 RemotingCommand。如果否为空，就认为是没有返回，抛出异常。（这里 RemotingCommand 的值，是在 processResponseCommand 方法里设置的，这个方法的调用，是在他的实现类 NettyRemotingClient 启动时，注册的一个 handler 里调用的。这个 handler 就是接收处理返回来的结果的。）
- 从 responseFuture集合里删除本次发送用的 responseFuture。
- 返回 RemotingCommand
```
public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis)
    throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {

    // TODO: 2018/4/26 这个变量作用是什么？
    final int opaque = request.getOpaque();

    try {
        // 新建一个 返回结果 对象
        final ResponseFuture responseFuture = new ResponseFuture(opaque, timeoutMillis, null, null);
        this.responseTable.put(opaque, responseFuture);
        final SocketAddress addr = channel.remoteAddress();
        // 使用 channel 进行发送
        channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) throws Exception {
                // 如果发送成功，就直接返回
                if (f.isSuccess()) {
                    responseFuture.setSendRequestOK(true);
                    return;
                } else {
                    // 发送不成功，就设置相关内容
                    responseFuture.setSendRequestOK(false);
                }

                responseTable.remove(opaque);
                responseFuture.setCause(f.cause());
                responseFuture.putResponse(null);
                PLOG.warn("send a request command to channel <" + addr + "> failed.");
            }
        });

        // 等待返回结果，如果在指定的超时时间内还没有结果返回，就继续向下走。
        RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
        // 如果 responseCmd 为空，就认为是没有返回，抛出异常。
        // 这里 responseCmd 的值，是在 processResponseCommand 方法里设置的，
        // 这个方法的调用，是在他的实现类 NettyRemotingClient 启动时，注册的一个 handler 里调用的。
        // 这个 handler 就是接收处理返回来的结果的。
        if (null == responseCommand) {
            if (responseFuture.isSendRequestOK()) {
                throw new RemotingTimeoutException(RemotingHelper.parseSocketAddressAddr(addr), timeoutMillis,
                    responseFuture.getCause());
            } else {
                throw new RemotingSendRequestException(RemotingHelper.parseSocketAddressAddr(addr), responseFuture.getCause());
            }
        }

        // 返回"返回结果"
        return responseCommand;
    } finally {
        // 删除"结果集合"中的结果。
        this.responseTable.remove(opaque);
    }
}
```

##2.1.3.1.2，MQClientAPIImpl#processSendResponse
这个方法的作用是，组装网络返回的结果。处理内容如下：

- 解析 response 的 header
- 根据发送的 msg，生成 MessageQueue
- 根据 msg id、responseHeader id、上面生成的 MessageQueue、responseHeader的QueueOffset，生成 SendResult
- 根据 response 设置 transactionid、regionId、trance等。
- 返回 SendResult

```
private SendResult processSendResponse(//
    final String brokerName, //
    final Message msg, //
    final RemotingCommand response//
) throws MQBrokerException, RemotingCommandException {
    switch (response.getCode()) {
        case ResponseCode.FLUSH_DISK_TIMEOUT:
        case ResponseCode.FLUSH_SLAVE_TIMEOUT:
        case ResponseCode.SLAVE_NOT_AVAILABLE: {
            // TODO LOG
        }
        case ResponseCode.SUCCESS: {
            SendStatus sendStatus = SendStatus.SEND_OK;
            switch (response.getCode()) {
                case ResponseCode.FLUSH_DISK_TIMEOUT:
                    sendStatus = SendStatus.FLUSH_DISK_TIMEOUT;
                    break;
                case ResponseCode.FLUSH_SLAVE_TIMEOUT:
                    sendStatus = SendStatus.FLUSH_SLAVE_TIMEOUT;
                    break;
                case ResponseCode.SLAVE_NOT_AVAILABLE:
                    sendStatus = SendStatus.SLAVE_NOT_AVAILABLE;
                    break;
                case ResponseCode.SUCCESS:
                    sendStatus = SendStatus.SEND_OK;
                    break;
                default:
                    assert false;
                    break;
            }

            // 解析 response 的 header
            SendMessageResponseHeader responseHeader =
                (SendMessageResponseHeader) response.decodeCommandCustomHeader(SendMessageResponseHeader.class);

            // 根据发送的 msg，生成 MessageQueue
            MessageQueue messageQueue = new MessageQueue(msg.getTopic(), brokerName, responseHeader.getQueueId());

            // 根据 msg id、responseHeader id、上面生成的 MessageQueue、responseHeader的QueueOffset，生成 SendResult
            SendResult sendResult = new SendResult(sendStatus,
                MessageClientIDSetter.getUniqID(msg),
                responseHeader.getMsgId(), messageQueue, responseHeader.getQueueOffset());
            sendResult.setTransactionId(responseHeader.getTransactionId());
            String regionId = response.getExtFields().get(MessageConst.PROPERTY_MSG_REGION);
            String traceOn = response.getExtFields().get(MessageConst.PROPERTY_TRACE_SWITCH);
            if (regionId == null || regionId.isEmpty()) {
                regionId = MixAll.DEFAULT_TRACE_REGION_ID;
            }
            if (traceOn != null && traceOn.equals("false")) {
                sendResult.setTraceOn(false);
            } else {
                sendResult.setTraceOn(true);
            }
            sendResult.setRegionId(regionId);
            return sendResult;
        }
        default:
            break;
    }

    throw new MQBrokerException(response.getCode(), response.getRemark());
}
```
##异步发送代码
最后我们直接看一下异步发送是如何做的。

- 生成 ResponseFuture
- 使用 channel 进行发送
 - 发送成功直接返回
 - 发送不成功，进行处理并调用 callback 方法。

这里有两个注意的地方：
1，在使用 channel 进行发送消息后，如果成功就直接返回，没有调用 callback 方法；而发送失败后，却调用了 callback 方法。这是为什么呢？
其实成功时候也调用了，是在`NettyRemotingAbstract#processResponseCommand`里调用的，这个方法是初始化 Netty 时候，注册处理器`NettyClientHandler`，在这个 Handler 里一步步调用的`processResponseCommand`。（异步编程真是看起来不方便）

2，在方法里使用了一个类`SemaphoreReleaseOnlyOnce`，应该是防止`Semaphore`被在方法中释放两次。其实可以使用一变量来解决这个问题，但看了一下代码，有两个地方都要做这个事，所以可能包装起来更好一些。

```
public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,
    final InvokeCallback invokeCallback)
    throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    final int opaque = request.getOpaque();
    boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
    if (acquired) {
        // 这个类的作用，从名字上看是"只释放一次"。从代码上看，下面的两个 SemaphoreReleaseOnlyOnce
        // 在一次执行过程中，可能都会被执行，所以使用这种方式？
        final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);

        final ResponseFuture responseFuture = new ResponseFuture(opaque, timeoutMillis, invokeCallback, once);
        this.responseTable.put(opaque, responseFuture);
        try {
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    // 为什么成功后，就直接返回了？不调用 callback 方法了？
                    // 如果不调用 callback 的话，send 的后置方法如何调用？
                    if (f.isSuccess()) {
                        responseFuture.setSendRequestOK(true);
                        return;
                    } else {
                        responseFuture.setSendRequestOK(false);
                    }

                    responseFuture.putResponse(null);
                    responseTable.remove(opaque);
                    try {
                        executeInvokeCallback(responseFuture);
                    } catch (Throwable e) {
                        PLOG.warn("excute callback in writeAndFlush addListener, and callback throw", e);
                    } finally {
                        responseFuture.release();
                    }

                    PLOG.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
                }
            });
        } catch (Exception e) {
            responseFuture.release();
            ...
        }
    } else {
        ...
    }
}

```




##问题
1. name server 有变化时，是如何与 producer 交互信息的。
##sendDefaultImpl


1，todo 为什么 `if (topicPublishInfo != null && topicPublishInfo.ok())` 不成功的话，不去抛出 exception，而是在 if 后面确认 name server 是否存在？在`tryToFindTopicPublishInfo`方法中，创建 topic 失败，返回来的 topicPublishInfo 可能是 null 或 !ok，为什么不去判断一下呢？

2，`ThreadLocalIndex#getAndIncrement`方法中，有下面一段代码，判断 `if( index<0 )`是否有必要，因为`Math.abs`结果不可能小于0吧。（很多地方都有 <0 的判断，为什么呢？）
```
index = Math.abs(random.nextInt());
if (index < 0)
    index = 0;
this.threadLocalIndex.set(index);
```

3，ThreadLocalIndex 这个类的作用应该是，在`轮循发送`时，取得要发送的 queue 的 index。mbus 中的`轮循发送`是如何做的，比较一下。

- 这个 ThreadLocalIndex 是一起向上增长，有没有溢出的可能？
- rocketmq 的 producer 中的 topic info 是不是不变的？

4，`MQFaultStrategy#selectOneMessageQueue` 中的 `for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++)`，在取得 messageQueueList 的 size 时，是否会出问题。比如，在取得的时候 size 是 2，但取得完后，size 变成了 1，这样在`tpInfo.getMessageQueueList().get(pos)`时候可能会出 IndexOutOfBoundsException 异常。

4，`MQFaultStrategy#selectOneMessageQueue` 的倒数第 2 个`return tpInfo.selectOneMessageQueue()`在 queue size 变化时，会不会出错；上面出错后，这个会不会一定成功？

5，`tpInfo.selectOneMessageQueue(lastBrokerName)`方法的作用是什么，在什么时候使用？（在 sendDefaultImpl 方法中，没见到哪里对 mq 这个变量进行初始化，所以一直是空的？）

6，为什么要有 `LatencyFaultTolerance` 这个的目的是“暂缓使用发送慢的 broker”吗？

7，`LatencyFaultTolerance#updateFaultItem`从代码上来看，有可能发生并发操作。但这里没有使用锁，而是使用 ConcurrentHashMap 来解决同步问题。分析一下这块的同步设置，会不会产生问题？（主要在`old = this.faultItemTable.putIfAbsent(name, faultItem);`这块，这块如果有值的时候，ord.setter 方法进行设置时，如果有其它线程也进行设置，会不会产生问题？一个线程设置 Current 一个线程设置 startTimestamp）

8，LatencyFaultTolerance 里面的 faultItemTable 的内容，都是通过 `updateFaultItem` 方法放进去的。

9，LatencyFaultTolerance 类是做什么的用的？从现在的理解上来看，是用作记录`发送延迟`的。如果是这样的话，是不是不应该选择有延迟的 MessageQueue 呀，还是所有的 MessageQueue 都进行记录，然后选择延迟最小的呢？

10，message 中的很多属性使用 map 进行保存，因为方便 message 进行升级迭代。如果使用具体属性的话，进行序列化等操作就不容易做。

11，MQClientManager 和 它里面的 MQClientInstance 的关系是什么？什么时候会有多个 MQClientInstance？

12，FaultStrategy 是选择一个比较优的 MessageQueue 来发送。这个和保证顺序发送有没有冲突？

13，ActiveMQ 在异步发送时，可以进行限流（Client 和 Broker 两端都可以），RocketMQ 如何做的呢？http://shift-alt-ctrl.iteye.com/blog/2034440
RocketMQ 好像没有这个问题，因为 RocketMQ 都是落盘。但好像在代码里看到 system busy 的错误返回字样了。[RocketMQ开发指南](http://www.sxt.cn/ueditor/php/upload/file/20170901/1504248466272324.pdf) 中的 4.10 写了 broker buffer 问题，不知道是不是有关。
RocketMQ中 System Busy代码：
```
NettyRemotingAbstract中
if (pair.getObject1().rejectRequest()) {


rejectRequest 的 实现在 SendMessageProcessor 中
@Override
    public boolean rejectRequest() {
        return this.brokerController.getMessageStore().isOSPageCacheBusy() ||
            this.brokerController.getMessageStore().isTransientStorePoolDeficient();
    }
```


##已解决

1, MQClientAPIImpl 这个类是负责信息的网络通信的类，内部是用 netty 实现的。所有 producer 都是用这一个类发进行发送的？
回答：实际的发送是使用 NettyRemotingClient 类进行发送的，它里面有一个 Map 保存着“IP地址/ChannelFuture”对，向哪个地址发送，就取得哪个 channel。





#类

