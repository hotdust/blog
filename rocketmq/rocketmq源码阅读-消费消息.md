
#基础介绍
在读源码时，有一些类的功能和一些属性的作用不明白的话，阅读起来可能会有点费事，先介绍一些：

1，PullMessageRequestHeader
这个类是 consumer 的 pull 请求的具体条件和内容。

2，SubscriptionData
这个类是保存 Filter 具体的过滤条件的。具体的过滤条件就是消息的 tags，是由 consumer 在 pull 时候提供。

3，几个配置类

- TopicConfigManager：保存 topic 相关的配置。对应的文件是：topics.json。
- ConsumerOffsetManager：保存每个 topic 对应的 每个 queue 的消费点，也就是 offset。对应的文件是：consumerOffset.json。
- SubscriptionGroupManager：保存 filter subscriptionGroup.json
- ScheduleMessageService：用来处理延迟消息的 service，同时也是配置类其中之一。对应的文件是：delayOffset.json

4，MessageQueue 是保存 topic 下面的 queue 的信息的，有 topic、brokerName、queueId。

5，ProcessQueue 是用来保存`client 拉取到的消息`。




- ConcurrentHashMap&lt;String, SubscriptionData&gt; subscriptionInner：是用来保存每个 consumer 订阅的 topic 和 tags 的信息。每个 cosnumer 调用 subscribe 来订阅 topic 时，都会把订阅的 topic 和 订阅条件(就是 tags) 都加到这个属性中来。Map 第一个参数是 topic，第二个参数是 tags。
- ConcurrentHashMap&lt;String, Set&lt;MessageQueue&gt;&gt; topicSubscribeInfoTable：用来保存 topic 和 MessageQueue 的对应关系。这个变量的初始值，是 consumer 在启动时（`DefaultMQPushConsumerImpl#start`），从 name server 取得 topic 信息，然后根据 subscriptionInner 属性取得当前实例上订阅的所有 topic，把从 name server 取得 topic 信息转化成我们需要的 topic 信息，保存到这个属性中。
关于这个属性的更新，在调用 MQClientInstance#updateTopicRouteInfoFromNameServer 时都有可能更新这个属性。但在 MQClientInstance#startScheduledTask 的方法中会定时调用 updateTopicRouteInfoFromNameServer 方法来更新这个属性。
- ConcurrentHashMap&lt;MessageQueue, ProcessQueue&gt; processQueueTable：
这个属性的内容在 RebalanceService 启动，进行了设置。ProcessQueue 是如何初始化的呢？
从 subscriptionInner 属性取得当前实例上订阅的所有 topic，对于 topic 的每个 queue（对应代码中的 MessageQueue）都创建一个 ProcessQueue。创建完 ProcessQueue 后，就把 MessaegQueue 和 ProcessQueue 放到当前这个属性中。关于这个属性的更新，会随着 RebalanceService 的定时启动而更新。


7，MQClientInsatnce 类中 consumerTable 属性是用来保存所有 consumer 的。consumer 在调用 start 方法启动时，把自己注册到这个属性中。

8，ConsumeMessageConcurrentlyService 是用来消费消息的具体类。





#一，Client 发送拉取请求
对于一个消费流程，client 和 broker 是如何运作的呢？
当像下面一样写完 conumer 具体的处理，并进行 start 后，client 就开始工作了。它是如何一步一步工作的呢？下面就一步步介绍。
```
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
consumer.subscribe("TopicTest", "*");
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
        ConsumeConcurrentlyContext context) {
        System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
consumer.start();
```

##1，RebalanceService
在运行 start 方法后，会调用`DefaultMQPushConsumerImpl#start() -> MQClientInstance#start()`方法。在这两个 start 方法中启动的很多 service，非常重要的一个 service 是 RebalanceService，不启动它的话，是无法向 broker 发送拉取消息请求的。

这个类虽然重要，但它只是实现了调用 MQClientInstance#doRebalance 方法，具体的实现在 RebalanceImpl 类中。

##2，RebalanceImpl
一步一步向方法里进，最后我们最后会进入到 RebalanceImpl#rebalanceByTopic 方法中，在这个方法进行具体的 rebalance 和 发送拉取请求操作。我们先忽略 rebalance 操作，主要看如何发送拉取请求的。在这之前，先介绍一下 RebalanceImpl 中的重要属性：


- ConcurrentHashMap&lt;String, SubscriptionData&gt; subscriptionInner：是用来保存每个 consumer 订阅的 topic 和 tags 的信息。每个 cosnumer 调用 subscribe 来订阅 topic 时，都会把订阅的 topic 和 订阅条件(就是 tags) 都加到这个属性中来。Map 第一个参数是 topic，第二个参数是 tags。
- ConcurrentHashMap&lt;String, Set&lt;MessageQueue&gt;&gt; topicSubscribeInfoTable：用来保存 topic 和 MessageQueue 的对应关系。这个变量的初始值，是 consumer 在启动时（`DefaultMQPushConsumerImpl#start`），从 name server 取得 topic 信息，然后根据 subscriptionInner 属性取得当前实例上订阅的所有 topic，把从 name server 取得 topic 信息转化成我们需要的 topic 信息，保存到这个属性中。
关于这个属性的更新，在调用 MQClientInstance#updateTopicRouteInfoFromNameServer 时都有可能更新这个属性。但在 MQClientInstance#startScheduledTask 的方法中会定时调用 updateTopicRouteInfoFromNameServer 方法来更新这个属性。
- ConcurrentHashMap&lt;MessageQueue, ProcessQueue&gt; processQueueTable：
用来保存 MessageQueue 和 ProcessQueue 的对应关系。MessageQueue 就是上面属性中的 MesageQueue，而 ProcessQueue 是 client 端的一个保存`拉取到的消息`的 queue。这个属性的内容在 RebalanceService 启动，进行了设置。ProcessQueue 是如何初始化的呢？
从 subscriptionInner 属性取得当前实例上订阅的所有 topic，对于 topic 的每个 queue（对应代码中的 MessageQueue）都创建一个 ProcessQueue。创建完 ProcessQueue 后，就把 MessaegQueue 和 ProcessQueue 放到当前这个属性中。关于这个属性的更新，会随着 RebalanceService 的定时启动而更新。


发送拉取请求的方法是`updateProcessQueueTableInRebalance`。这个方法做了以下的事情：

- 如果 rebalance 后，client 不需要对 topic 的某个 queue 进行拉取的话，就把这个 queue 对应的 ProcessQueue 从 processQueueTable 中删除掉。
- 如果 rebalance 后，某个 topic 的 queue 不在拉取范围之内，则把这个 queue 信息添加到 processQueueTable 中，并且把新增加的 queue 信息添加到一个`拉取请求集合`中。
- 如果`拉取请求集合`中有数据的话，就发送这些拉取请求。

```
private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet, final boolean isOrder) {
    boolean changed = false;

    Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<MessageQueue, ProcessQueue> next = it.next();
        MessageQueue mq = next.getKey();
        ProcessQueue pq = next.getValue();

        // 如果 rebalance 后，client 不需要对 topic 的某个 queue 进行拉取的话，就把这个 queue 对应的 ProcessQueue 从 processQueueTable 中删除掉。
        if (mq.getTopic().equals(topic)) {
            // 如果 processQueueTable 中的当前这条数据的 mq 不在 topic 对应的 mq 集合里的话，
            // （说明经过 这个 queue 已经不存在了，就不需要对这个 queue 进行 pull 了。
            //   可能是 rebalance 后，client 不再需要拉取这个 topic 对应的 queue 了。）
            // （mqSet 是 topic 和 Set<MessageQueue> 的映射。）
            if (!mqSet.contains(mq)) {
                pq.setDropped(true);
                if (this.removeUnnecessaryMessageQueue(mq, pq)) {
                    it.remove();
                    changed = true;
                    log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
                }
            } else if (pq.isPullExpired()) {
                switch (this.consumeType()) {
                    case CONSUME_ACTIVELY:
                        break;
                    case CONSUME_PASSIVELY:
                        pq.setDropped(true);
                        if (this.removeUnnecessaryMessageQueue(mq, pq)) {
                            it.remove();
                            changed = true;
                            log.error("[BUG]doRebalance, {}, remove unnecessary mq, {}, because pull is pause, so try to fixed it",
                                consumerGroup, mq);
                        }
                        break;
                    default:
                        break;
                }
            }
        }
    }




    List<PullRequest> pullRequestList = new ArrayList<PullRequest>();
    // 如果 rebalance 后，某个 topic 的 queue 不在拉取范围之内，则把这个 queue 信息添加到 processQueueTable 中。
    // (检查一下 mqSet 中的哪个 mq 没有对应的 ProcessQueue，
    // 找到没有对应的 ProcessQueue 后，创建一个对应的 ProcessQueue。)
    for (MessageQueue mq : mqSet) {
        if (!this.processQueueTable.containsKey(mq)) {
            if (isOrder && !this.lock(mq)) {
                log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
                continue;
            }

            this.removeDirtyOffset(mq);
            ProcessQueue pq = new ProcessQueue();
            long nextOffset = this.computePullFromWhere(mq);
            if (nextOffset >= 0) {
                ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
                if (pre != null) {
                    log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
                } else {
                    log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
                    PullRequest pullRequest = new PullRequest();
                    pullRequest.setConsumerGroup(consumerGroup);
                    pullRequest.setNextOffset(nextOffset);
                    pullRequest.setMessageQueue(mq);
                    pullRequest.setProcessQueue(pq);
                    pullRequestList.add(pullRequest);
                    changed = true;
                }
            } else {
                log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
            }
        }
    }

    this.dispatchPullRequest(pullRequestList);

    return changed;
}
```

最后一步，就是发送请求的地方了。这个方法的实现有 2 个，我们看一下 RebalancePushImpl 中的实现。
实现很简单，就是循环请求，然后调用`DefaultMQPushConsumerImpl#executePullRequestImmediately`方法，下面我们进去看一看具体的实现。
```
@Override
public void dispatchPullRequest(List<PullRequest> pullRequestList) {
    for (PullRequest pullRequest : pullRequestList) {
        this.defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest);
        log.info("doRebalance, {}, add a new pull request {}", consumerGroup, pullRequest);
    }
}
```

##3，PullMessageService
跟着上面的`DefaultMQPushConsumerImpl#executePullRequestImmediately`实现，一步步会进入 PullMessageService 类的 executePullRequestImmediately 方法中。在这个方法中，只做了一件事：把拉取请求放到 pullRequestQueue 中。下面介绍一下 PullMessageService 类的结构。

PullMessageService 是一个典型的`生产者/消费者`模式的类，pullRequestQueue 这个属性就是用于连接生产者和消费者的队列。这个 service 在启动时，就启动了消费进程，如果 pullRequestQueue 里有内容，就取得进行消费。而这消费处理，就是发送拉取请求。

```
// 把请求放到 pullRequestQueue 属性中
public void executePullRequestImmediately(final PullRequest pullRequest) {
    try {
        this.pullRequestQueue.put(pullRequest);
    } catch (InterruptedException e) {
        log.error("executePullRequestImmediately pullRequestQueue.put", e);
    }
}

// service 启动后，就从 pullRequestQueue 中取得请求。
@Override
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            // TODO Q: 2018/4/25 pullRequestQueue 里的数据，是哪来的，找了半天感觉死循环了？
            PullRequest pullRequest = this.pullRequestQueue.take();
            if (pullRequest != null) {
                this.pullMessage(pullRequest);
            }
        } catch (InterruptedException e) {
        } catch (Exception e) {
            log.error("Pull Message Service Run Method exception", e);
        }
    }

    log.info(this.getServiceName() + " service end");
}

// 取得请求后，进行处理
private void pullMessage(final PullRequest pullRequest) {
    final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
    if (consumer != null) {
        DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
        impl.pullMessage(pullRequest);
    } else {
        log.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
    }
}
```

##4，DefaultMQPushConsumerImpl
这个类的 pullMessage 方法，定义了拉取到消息后的处理（实际是一个回调函数的形式），并执行拉取方法。

其中，PullCallback 的实现就是拉取到消息后处理部分，等介绍完 broker 流程后再进行介绍。而 PullAPIWrapper#pullKernelImpl 方法是执行具体发送拉取请求的入口，不做过多介绍。


#二，Broker 处理拉取请求
##1，PullMessageProcessor
这个类是处理拉取请求的实现类，在 broker 启动时被注册到了 netty 上。rejectRequest 方法是处理拉取请求的具体方法（这个方法非常长，300+）。在这个方法中，重要的是下面这段代码：
```
final GetMessageResult getMessageResult =
    this.brokerController.getMessageStore().getMessage(
        requestHeader.getConsumerGroup(), requestHeader.getTopic(),
        requestHeader.getQueueId(), requestHeader.getQueueOffset(),
        requestHeader.getMaxMsgNums(), subscriptionData);
```
这段代码是用来用来取得消息的，通过 client 传过来的 topic、queueId、queueOffset、requestHeader、最大取得消息数、订阅的 tags 信息等。

##2，DefaultMessageStore#getMessage
取得消息流程如下：

- 通过 queueOffset 的位置，从 consume queue 中取得到现在所有的 consume queue 格式的消息。
- 对所有 consume queue 格式消息进行循环，如果达到用户请求的最大消息数，就退出循环。
 - 通过 consume queue 格式消息，从 CommitLog 中取得我们所需要的具体的消息内容。
 - 如果消息 tags 符合拉取请求的 tags 条件，就把这条消息加入到返回值中。
- 判断是否有符合条件的消息，如果没有就返回空，然后把当前 client 的 netty channel 保存起来，等有消息了就发给这个 client。
- 返回拉取到的消息。


#三，Client 处理拉取到的消息
##1，PullCallback
在上面介绍 client 时已经介绍了 PullCallback 实现的作用，它是处理`已经拉取到的消息`。处理内容如下：

- 如果拉取到消息
 - 把取得的消息，放到 processQueue 中。
 - 异步处理取得的消息（ConsumeMessageConcurrentlyService#submitConsumeRequest）
 - 再发送一个拉取请求，进行下一次拉取。
- 如果没拉取到消息， 再发送一个拉取请求，进行下一次拉取。
- 如果拉取错误（省略）

我们着重看一下异步处理。

##2，ConsumeMessageConcurrentlyService
ConsumeMessageConcurrentlyService 是异步处理的实现类之一，还有一个是 ConsumeMessageOrderlyService。它的结构是一个线程池结构，当有异步处理请求来了，就把请求放到线程池里执行。请求本身（ConsumeRequest）就是一个可运行的类（实现了 Runnable 接口）。处理内容如下：

- 如果取得的消息数 小于 每次处理限制，就直接处理
- 如果取得的消息数 大于 每次处理限制，就拆分成限制的大小，循环处理。

ConsumeRequest#run 方法是具体的处理消息的地方，在这里调用了使用者实现的 listener 中的业务逻辑。处理内容如下：

- 调用前置回调。
- 调用用户实现的 listener 实现，消费消息。
- 调用后置回调
- 处理`消费消息结果`（processConsumeResult() 方法）。
 - 如果消费成功，就记录统计数据
 - 如果消费失败，就把消息返回给 broker。如果返回给 broker 失败，就把返回失败的消息让 client 重新消费。
 - 把消费完的消息，从 processQueue 中删除掉。
 - 记录消费的 queueOffset（就是进行 ack 确认）

###有一些细节的地方：
####1，记录消费到的 queueOffset。
queueOffset 是如何生成的呢？从 processQueue 删除掉消费完的消息后，如果 processQueue 里还有消息，就剩余消息的第一个（queueOffset最小）的 queueOffset；如果没有，就返回删除消息中，最后一个消息的 queueOffset + 1，即下一个消息的位置。注意：这里的 queueOffset 是每个消息进入 broker 后，这个 queueOffset 都是递增 1 的。

####2，关于 ack
所谓的 ack 就是把`被消费的最后一个消费的 queueOffset` 同步到 broker。对于非顺序消费，当返回值为 SUCCESS，代表消费成功，进行 ack。如果返回值为 RECONSUME_LATER，或者 exception 发生时（exception 发生的话，返回值为RECONSUME_LATER），代表消费失败不进行 ack，要把消息返回给 broker 进行重新消费。

####3，关于某个消息消费时间过长问题
如果某一条消息消费时间过长，会卡住后面的消费进行消费。所以在 ProcessQueue 中有一个 cleanExpiredMsg 方法来解决卡住消息问题。这个方法是定时启动的，每次启动后都会确认 ProcessQueue 里的消息是否`消息时间过长`，如果过长，把消息发回 broker 延迟队列中（delayLeve=3 的延迟队列），然后把这个消息从 ProcessQueue 删除掉。从 ProcessQueue 删除掉就相关于 ack 这个消息了。

但这个 cleanExpiredMsg 无法立刻停止卡住的消息，还需要 listener 自己完成后才能停止。如果是消息失败返回，client 会把这批消息返回到 broker。至于 broker 如何处理这些消息，client 是可以控制，是在 listener 设置 ConsumeConcurrentlyContext 的 delayLevelWhenNextConsume 属性。属性值如下：

- -1：no retry,put into DLQ directly
- 0：broker control retry frequency
- >0：client control retry frequency

由此可见，cleanExpiredMsg 作用是`消息在一定时间没消费掉的话，ack 这个消息，并放到延迟队列中，一定时间后再进行激活线上`，这也是解决卡住消息的办法。这个办法无法终止消息处理，只能等待消息处理完事。



#四、Broker 接收返回来的消息
当消费失败时，会是把消息发回给 broker 的。发回给 broker 后，broker 是如何处理的呢？
（处理逻辑在 `SendMessageProcessor#consumerSendMsgBack` 方法中）

- 首先通过消息的 commit log offset，取得具体的消息内容
- 然后看是否超过重试最大次数，如果没有超过，就把消息发送 retry 队列；如果超过了，就把消息发到死信队列里。

##关于消息的 delayLevel 属性
在发回的消息中，有一个 delayLevel 属性。这个属性只有在返回的队列为 retry 队列时才设置。这个属性的意义是，告诉 broker 这条消息放到哪个延迟队列中。当消息返回 broker 中的 retry 队列时，并不是直接把消息放到 retry 队列中，而是放到延迟队列中。等延迟的时间到了，再放到 retry 队列中。

这个属性的值为：重试次数 + 3，例如：如果是第 1 次重试，就是 1 + 3 = 4；第 2 次重试，就是 2 + 3 = 5。值对应是`第几个`延迟队列，例如，4 对应的就是第 4 个延迟队列。那第 4 个延迟队列是延迟多长时间呢？这要看是如何定义的了，下面是默认延迟队列的个数和延迟时间：
```
private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m" + 
                                   "10m 20m 30m 1h 2h";
```
`1s` 就是第 1 个延迟队列，它的延迟时间是 1 秒。`1m` 是第 5 个延迟队列，延迟时间 5 分钟。


#其它细节
##1，PullRequestHoldService
这个类的作用是，当 consumer 拉取不到消息时，把当前 consumer 挂起（就是把 consumer 发的 request 保存起来)，返回的 response 设置为 null，这样 consumer 就不会再发拉取请求过来（NettyRemotingAbstract#processRequestCommand 方法可以看出来）。当有消息进来后，broker 使用刚才`挂起的 consumer 的 request`再去拉取一次消息（让 consumer 拉取消息的运作，是在 broker 这边做的，和 client 无关）。


**挂起 consumer**
使用 suspendPullRequest 方法来挂起 consumer。所谓的“挂起”，就是把 consumer 的 pull 请求对象（pullRequest）保存起来，当有消息时，能直接把消息通过 pull 请求直接发给 consumer。



**通知被挂起的 consumer**
notifyMessageArriving 方法是通知`被挂起的 consumer`去再次拉取消息。这个方法调用有两种方式：

- 当有消息保存后，就会调用此方法。在`MessageArrivingListener#arriving`方法中调用的 notifyMessageArriving 方法。
- 定时启动，调用此方法。启动时间根据否是 long polling 来决定，如果是的话，5 秒一启动；如果不是，1 秒一启动。

虽然 notifyMessageArriving 方法被调用，但只是通知被挂起的 consumer 可以去拉取消息，consumer 是否拉取消息，是由两种情况：

- 如果`当前 consumer queue 的 offset` 比 `consumer 的 request 中的 queueOffset`大的话，就去拉取。
- 如果上面条件不符合，再判断`当前时间`是否已经超过`超时时间`，如果超过就去拉取，但如果拉取不到就不再进行挂起了(brokerAllowSuspend 设置成了 false)。超时时间 = 挂起consumer 的时间 + 默认超时时间（BROKER_SUSPEND_MAX_TIME_MILLIS：15 秒）

为什么要有这个超时时间呢？因为如果 broker 很长时间收不到消息，就会一直保存着呢 consumer 的 request。可能是防止 broker 保存的 reuqest 过多，设置了这个超时时间。`PullRequestHoldService#notifyMessageArriving`方法中有下面这样一段代码：
```
List<PullRequest> requestList = mpr.cloneListAndClear();
```
这段代码就是取出 topic 和 queueId 对应的 consumer request，并清空一直保存的 request。
> 但就算清空了，如果是 consumer 使用的是 push 方式的话，还是会再立刻发拉取请求来的。不是又变成挂起的状态了，还需要保存这些 request 么？



##2，topicSubscribeInfoTable 的更新

topicSubscribeInfoTable 是通过 updateTopicRouteInfoFromNameServer 方法，从 name server 取得 topic 和信息后，把 topic 信息转化成 topicSubscribeInfoTable 需要的信息。

topicSubscribeInfoTable 信息在 consumer 进行 start 时进行更新：

```
DefaultMQPushConsumerImpl#start
  -> DefaultMQPushConsumerImpl#updateTopicSubscribeInfoWhenSubscriptionChanged
    -> 通过 rebalanceImpl.getSubscriptionInner() 取得所有 subscribe 的 topic 信息，
       对这些 topic 循环调用下面的更新方法。
      -> MQClientInstance#updateTopicRouteInfoFromNameServer
```

##3，RebalanceService 的平衡策略
这个 service  主要是用来根据设置的`平衡策略`，来调整 consumer 消费哪条 topic 的 queue（代码的角度来看，就是如何更新 processQueueTable）。几个平衡策略如下：


###（1）AllocateMessageQueueAveragely：
分配结果如下：

| - | Consumer 2 可以整除* |	Consumer 3 不可整除* | Consumer 5 无法都分配* |
| - | - | - | - |
|消息队列[0] | Consumer[0] | Consumer[0] | Consumer[0] |
| 消息队列[1] | Consumer[0] | Consumer[0] | Consumer[1] |
| 消息队列[2] | Consumer[1] | Consumer[1] | Consumer[2] |
| 消息队列[3] | Consumer[1] | Consumer[2] | Consumer[3] |
引用：[RocketMQ 源码分析 —— Message 拉取与消费（下）](http://www.iocoder.cn/RocketMQ/message-pull-and-consume-second/)

下面说一下代码：

- mqAll：topic 对应的 queue 的集合。一般 topic 对应的 queue 的个数默认为 4 个。
集合内元素是已经排序过的。
- cidAll：消费某个 topic 的 consumer group 里的 client 集合。例如：两个 client 消费某个 topic 时，consumer group 相同的话，这个集合里就有两个元素。集合内元素是已经排序过的。
- cid：client 的 id。默认为：IP + "@" + 进程号。例如：192.168.1.2@46885
- index：当前 client 在 cidAll 集合中排在第几位。
- mod：mqAll % cidAll 的余数。

从例子可以看出，如果能 queue 能被 client 平均分配还好计算，如果不能平均分配就复杂了。在不能平均分配时，client id 在集合中靠前的，是要承担更多的 queue 的，比如上面的 consumer 3。知道这些后，我们看看具体的一些代码：

```
int averageSize =
            mqAll.size() <= cidAll.size() ? 1 : 
              (mod > 0 && index < mod ? 
                mqAll.size() / cidAll.size() + 1 : mqAll.size() / cidAll.size());
```
`mqAll.size() <= cidAll.size() ? 1` 这段简单，我们看看复杂的部分，就是后半部分。

```
(mod > 0 && index < mod ? 
    mqAll.size() / cidAll.size() + 1 : mqAll.size() / cidAll.size()
```
这段的意思是，在不能平均分配 queue 的情况下，让第一个 client 消费比平均多 1 个的 queue。因为在不能平均分配的时候，肯定会有某些 client 比另一些 client 多消费 queue，那就让在 cidAll 集合中位置先前的 client 多消费 queue。即使多消费，和没有多消费的 client 相比，只是多消费 1 个。

```
int startIndex = (mod > 0 && index < mod) ? 
  index * averageSize : index * averageSize + mod;
```
在不能平均分配 queue 的情况下， 大于或等于 mod 的 client 起步值比平均步数多 1，因为小于 mod 的 client 是比`大于等于 mod 的 client`多 1 个 queue。


```
int range = Math.min(averageSize, mqAll.size() - startIndex);
```
range 是每个 client 要消费的 queue 的个数。没进行严格举例，从代码上来看，可能是为了

- 防止`所剩 queue 个数 比 average 小`的情况。当 queue 个数为 9， client 个数为 5 的时候，可能会发生这种情况。
- mqAll.size() <= cidAll.size() 时，最后几个 Consumer 分配不到消息队列。




###（2）AllocateMessageQueueAveragelyByCircle
分配结果如下：

| - | Consumer 2 可以整除* |	Consumer 3 不可整除* | Consumer 4,5 无法都分配* |
| - | - | - | - |
|消息队列[0] | Consumer[0] | Consumer[0] | Consumer[0] |
| 消息队列[1] | Consumer[1] | Consumer[1] | Consumer[1] |
| 消息队列[2] | Consumer[0] | Consumer[2] | Consumer[2] |
| 消息队列[3] | Consumer[1] | Consumer[0]　 | Consumer[3] |


###（3）AllocateMessageQueueByMachineRoom
平均分配消息队列。该平均分配方式和 AllocateMessageQueueAveragely 略有不同，其是将多余的结尾部分分配给前 rem 个 Consumer。在使用这个策略之前，要设置`consumeridcs`属性。


###（4）AllocateMessageQueueByConfig
返回已经设置好的 queue 配置。使用场景可能是为了编程式设置？或者是测试？


##4，拉取消费位置设置（RebalancePushImpl#computePullFromWhere）
在设置拉取消息位置时，有 3 个选项：

- CONSUME_FROM_LAST_OFFSET ：跟着上次的消费进度，消费后面的消费。
- CONSUME_FROM_FIRST_OFFSET ：从第一个位置开始消费。
- CONSUME_FROM_TIMESTAMP ：从指定的时间点后开始消费。


##5，ConsumeMessageConcurrentlyService 的 消息清理
ConsumeMessageConcurrentlyService 类还有消息清理功能。主要是对超时消息进行清理，逻辑如下：

- 从 processQueue 里取得要处理的第一条消息。如果消息不超时，结束本次清理。
- 如果消息超过一定时间还没有被 consumer 处理，就把消息返回给 broker。（一次最多清理 16 条消息）

##6，consumer 拉取消息时提交的 offset
每次取得消息时，使用的 offset 是 consume queue 的 offset。broker 通过 consume queue 的 offset 从 consume queue 上取得消息所在 commit log 上的位置和大小，然后再从 commit log 上取得消息具体内容返回给 consumer。

##7，关于 PullConsumer 的使用
下面是 PullConsumer 中的一段代码。由代码可以看出，pull 的使用方式是：

- 先取得 topic 的所有 queue。
- 对每个 queue 进行拉取消息。

至于每个 queue 拉取到什么位置（也就是 offset）了，需要 client 端自己记录一下。

```
public static void main(String[] args) throws MQClientException {
    DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("please_rename_unique_group_name_5");

    consumer.start();

    Set<MessageQueue> mqs = consumer.fetchSubscribeMessageQueues("TopicTest1");
    for (MessageQueue mq : mqs) {
        System.out.printf("Consume from the queue: " + mq + "%n");
        SINGLE_MQ:
        while (true) {
            try {
                PullResult pullResult =
                    consumer.pullBlockIfNotFound(mq, null, getMessageQueueOffset(mq), 32);
                System.out.printf("%s%n", pullResult);
                putMessageQueueOffset(mq, pullResult.getNextBeginOffset());
                switch (pullResult.getPullStatus()) {
                    case FOUND:
                        break;
                    case NO_MATCHED_MSG:
                        break;
                    case NO_NEW_MSG:
                        break SINGLE_MQ;
                    case OFFSET_ILLEGAL:
                        break;
                    default:
                        break;
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    consumer.shutdown();
}
```

##8，关于 Push 和 Pull
RocketMQ 有两种方式取得消息：Push 和 Pull。但实现方式上，都是用 pull 的方式来做的。

- Push 的实现方式是，向 broker 发送一个请求，当请求结果返回后，再发送一个请求，如此循环。
- Pull 的实现方式是，每次调用 pullXXX 方法只向 broker 发送一次请求。调用多少次 pullXXX 方法是在 client 端由用户自己控制。

所以，你可以看到 PullAPIWrapper、PullRequest、PullMessageRequestHeader 类，却看不到这些类的 Push 版本。

##9，关于 Retry 
###（1）关于 retry topic
每个 consumer group 都对应一个 retry topic，命名规则为：`%RETRY% + consumer group`，例如：`%RETRY%please_rename_unique_group_name_4`。因为 topic 名字是以 consumer gropu 命名的，所以所有 consumer group 内的 retry 消息都会放到这个 topic 中。这个 topic 只有一个 queue，具体 topic 配置如下：
```
"%RETRY%please_rename_unique_group_name_4":{
	"order":false,
	"perm":6,
	"readQueueNums":1,
	"topicFilterType":"SINGLE_TAG",
	"topicName":"%RETRY%please_rename_unique_group_name_4",
	"topicSysFlag":0,
	"writeQueueNums":1
}
```

###（2）retry topic 的订阅
在 consumer 调用 start 方法启动时，会调用`DefaultMQPushConsumerImpl#copySubscription()`方法，在这个方法中对 retry topic 进行订阅。
```
    private void copySubscription() throws MQClientException {
        try {
            ... 省略一部分代码
            switch (this.defaultMQPushConsumer.getMessageModel()) {
                case BROADCASTING:
                    break;
                case CLUSTERING:
                    // 创建当前 topic 的 retry topic
                    final String retryTopic = MixAll.getRetryTopic(this.defaultMQPushConsumer.getConsumerGroup());
                    SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(), //
                        retryTopic, SubscriptionData.SUB_ALL);
                    // 把 retry topic 加入到总结的 topic 集合中，稍后对 topic 集合里的 topic 发请求，取得消息。
                    this.rebalanceImpl.getSubscriptionInner().put(retryTopic, subscriptionData);
                    break;
                default:
                    break;
            }
        } catch (Exception e) {
            throw new MQClientException("subscription exception", e);
        }
    }

```


###（3）消息如何放到 retry topic 中
`SendMessageProcessor#consumerSendMsgBack` 方法是处理`返回到 broker 的消息`的。因为是返回给 broker 的消息，所以认为这些消息都是重试消息。如果消息超时重试最大次数，就放到死信队列中；如果没超过，就根据设置放到对应的延迟队列中。所以重试消息不是直接放到对应的`retry topic`中，而是放到延迟队列中。延迟消息到时间后，再从延迟队列取出，放到相应的 retry topic 中。



##10，string2File 和 string2FileNotSafe
这两个方法是用来写文件的，第一个方法是安全的，第二个方法是不安全的。为什么第二个方法是不安全的呢？

OS和磁盘读写的基本单位是块，也可以称之为(page size)block size，OS的块则一般为4K；IO块则更小，linux内核要求IO block size<=OS block size。磁盘IO除了IO block size，还有一个概念是扇区(IO sector)，扇区是磁盘物理操作的基本单位，而IO 块是磁盘操作的逻辑单位，一个IO块对应一个或多个扇区，扇区大小一般为512个字节。

所以各个块大小的关系可以梳理如下：
>DB block > OS block >= IO block > 磁盘 sector

在写数据时，则写完几个磁盘 sector 后就突然停电了，结果一部分数据写到磁盘上了，而还有一部分数据没有写到磁盘上，造成数据损坏。所以第二个方法是不安全的。

像第一个方法那样，先把要写的数据`不直接写到目标文件上`，而写成`目标文件.tmp`，然后把目标文件改名为`目标文件.bak`，再把`目标文件.tmp`改名成`目标文件`，这样就算写时候断电造成数据损坏，也坏的是`目标文件.tmp`，而不影响`目标文件`。

但还有一个可能性，就是在改名时候发生问题。为了解决这个问题，在读文件地方要做好如果`目标文件`不存在，要读取`目标文件.bak`的准备。

这个方法和 mysql 的 double write 有些像。关于这方面请看：[【mysql】Innodb三大特性之double write](https://www.cnblogs.com/chenpingzhao/p/4876282.html)





#特点： 
1，DefaultMessageStore#checkInDiskByCommitOffset 来检查消息是在内存中，还是在磁盘上。判断逻辑是：未消费的消息字节数，是不是超过物理内存的 40%。如果超过了，有一部分消息就是在磁盘中了；如果没有，那消息就全部在内存中。这个逻辑如果在一般环境中（一台机器只有 RocketMQ，没有其它服务或中间件），可能还是可行的
个人想了一下，使用内存的地方可能如下：（现在对 RocketMQ 的整体还不太了解，可能不准确）

1. 启动 rocketmq 的 jvm 内存。记得官方推荐 16G 内存的机器，runbroker.sh 里默认是 8G。这样的内存就剩下 50%了。
2. 除了 CommitLog 外，Index、ConsumeQueue 等会占用一部分 page cache 内存，这些文件不算太大，我们估计成占用 10% 的内存。
3. 系统占用内存忽略。

上面占用内存统计后，大约是 60%，剩下的 40% 内存是 CommitLog 所占用，所以 checkInDiskByCommitOffset 方法里，计算是否在内存中时，使用的是系统内存的 40%。不知道这样从结果推原因，是不是准确。

2，如果还没有消费的消息的字节数 > 系统内存的40%，就建议去 slave 读取消息。
这个地方可以减少 master 的压力

3，ProcessQueue 是 consumer 用来存储从 pull 来发消息。具体存储时，使用的是 TreeMap，这样可以保证 offset 小的最先被消费。而且 TreeMap 使用的是红黑树，性能也非常好。

5，如果还没有消费的消息的字节数 > 系统内存的40%，就建议去 slave 读取消息

6，在异步处理方面，有些地方使用线程池，有些地方使用的是`生产者/消费者`。ConsumeMessageConcurrentlyService 是线程池，PullMessageService 是`生产者/消费者`。
从纯技术角度看，`生产者/消费者`模式可以把生产者和消费者解耦，而线程池不行。在串行和并行方面来看，RocketMQ 使用的`生产者/消费者`是串行操作，即消费完一个才能消费另一个，很多场景都是在保存数据到 MappedFile 时候；而使用线程池时，使用的是并行操作，比如消费消息时候。

7，Pull 时的限流控制
在拉取到消息后，要判断：

- 是否 ProcessQueue 消息大于指定数量
- 是否 ProcessQueue 第一条消息和最后一条消息的 offset 之间差过大

符合任何一个条件都要`等待一会`，再去拉取消息。

8，在写文件时（string2File），使用类似于 mysql 的 double write 方式保证写数据完整性。

9，IO 调度算法
在《RocketMQ》开发指南上说到，为了加快读效率，可以把 Linux 的 IO 调度算法改成 NOOP，原文如下：
> 随机访问 Commit Log 磁盘数据，系统 IO 调度算法设置为 NOOP 方式，会在一定程度上将完全的随机读变成顺序跳跃方式，而顺序跳跃方式读较完全的随机读性能会高 5 倍以上，可参见以下针对各种 IO 方式的性能数据。

在文章《[调整 Linux I/O 调度器优化系统性能](https://www.ibm.com/developerworks/cn/linux/l-lo-io-scheduler-optimize-performance/index.html)》中说了，DeadLine 适合特别是写入较多的文件服务器。但由于 RocketMQ 都是顺序写，所以写 IO 这块不是问题。读数据可能是随机读，所以这块是需要加强的，所以他们推荐 NOOP 算法。

《[Linux IO Scheduler（Linux IO 调度器） - CobbLiu - 博客园](https://www.cnblogs.com/cobbliu/p/5389556.html)》也说了一些 DeadLine 和 NOOP 各自的好处，没看到上面说的 NOOP 的好处。所以，最好还是测试一下为好。


#问题
1，有一种消息被跳过消费的可能性
先向 topic 的 queue 1 放入一条消息，消息的 tag 为 tagA。再往这个 queue 1 里放一条消息，消息的 tag 为 tagB。然后启动一个 consumer，消费条件为 tagB。结果，tagB 的消费被正常消费了，而 tagA 的消费被跳过了，就算再启动一个消费 tagA 的 consumer（offset 为 CONSUME_FROM_LAST_OFFSET），也无法消费到 tagA 的消息了，因为 queue 1 的消费 offset 已经被设置到了 tagB 的消息之后。

可能从使用方式上来看，这种情况不太可能发生，但要注意一下。






todo:
OK 1，看一下 SubscriptionGroupConfig subscriptionGroupConfig，这个类的作用
PASS 2，requestHeader.getSysFlag() 每个位是做什么用的？
OK 3，TopicConfigManager 是做什么用的？
OK 4，ConsumerManager 是做什么用的？
PASS 5，哪里对 RunningFlags 进行的设置？在 get 方法中进行设置了，但 flagBits 最开始的值是在哪设置的呢？
OK 6，requestHeader.getQueueOffset() 是如何设置的呢？这个 offset 保存在哪里？
PASS 7，从 findConsumeQueue 看一下 ConsumeQueue 是如何保存的？
OK 8，从 DefaultMessageStore#checkInDiskByCommitOffset 方法来看，是把总内存的 40% 的大小的消息保存在内存中，这个是怎么做的？在什么地方做的？
OK 9，if (this.isTheBatchFull 不满足时，就不给 consumer 发消息，让 consumer 一直等待到满足条件吗？
OK 10，diskFallRecorded 的作用是什么？统计剩余可拉取消息字节数
OK 11，consumerGroupInfo.findSubscriptionData 取得的 subscriptionData 作用是什么？
OK 12，看看 DefaultMessageFilter#isMessageMatched 中的逻辑
OK 13，nextPhyFileStartOffset = this.commitLog.rollNextFile(offsetPy); 作用是什么？
PASS 14，MomentStatsItemSet 作用是什么？为什么 if (diskFallRecorded) 时候要调用一下？
OK 15，nextBeginOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE); 是为了算出下一会 pull 时的 offset 吗？
OK 16，getResult.setSuggestPullingFromSlave(diff > memory); 作用是什么？
17，if (getMessageResult.isSuggestPullingFromSlave()) { 是为了让 client 从其它 slave 中取得消息吗？
18，recordDiskFallBehindTime 方法是做什么的？
19，FileRegion 是做什么用的？怎么使用？
20，FileRegion fileRegion = new ManyMessageTransfer(response.encodeHeader 为什么要这么做，这么做有什么好处？
21，有时间看一下 netty 的 ReferenceCounted。
22，如果可以从 master 或 slave 消费消息的话，是如何控制呢？如何同步的呢？
23，去 slave 读消费消息的话，是如何做的？消费完是如何同步给 master 的呢？
PASS 24，如何设置回调？if (this.hasConsumeMessageHook()) {
PASS 25，记录一下 BrokerStatsManager 中都记录 MQ 的哪些状态。
26，看看 netty 的 AbstractReferenceCounted
PASS 27，boolean storeOffsetEnable = brokerAllowSuspend; 下面那段代码，是为了记录消费的 offset 吗？但还没有传回给 consumer 呢？
PASS 28，PullRequestHoldService 是属于长连接，还是 push？看看别的文章有没有解释？介绍一下这个类
OK 29，MixAll#string2File 和 string2FileNotSafe 是如何保证安全和非安全的？
30，FileRegion fileRegion = new ManyMessageTransfer 这个地方为什么要这么用，有什么好处？
31，看 RebalanceService：均衡消息队列服务，负责分配当前 Consumer 可消费的消息队列( MessageQueue )。当有新的 Consumer 的加入或移除，都会重新分配消息队列。
32，Rebalance 各种策略都用在什么情况下？
33，如何判断`switch (this.defaultMQPushConsumer.getMessageModel()) {`后面是失败消息？
OK 34，消费消息时 push 和 pull 都是如何做的？
OK 35，ConsumeMessageOrderlyService 和 ConsumeMessageConcurrentlyService 有什么不同。
36，retry consumer group(%RETRY%consumergroup) 是什么时候使用的？
OK 37，retry topic 里的数据是如何消费的？
OK 38，retry 操作其实是对一个消息保存多个复本，假如第 3 次重试时候被消费了，那之前在 retry topic 中的 2 条消息，怎么办？有没有过期一说？影响不影响 commit log 的回收。
39，activemq 的延迟消息是怎么做的？



#参考：
[RocketMQ 源码 分析 —— Message 拉取与消费（上）](http://www.iocoder.cn/RocketMQ/message-pull-and-consume-first/)：
[RocketMQ 源码分析 —— Message 拉取与消费（下）](http://www.iocoder.cn/RocketMQ/message-pull-and-consume-second/)：
[RocketMQ——消息ACK机制及消费进度管理](http://jaskey.github.io/blog/2017/01/25/rocketmq-consume-offset-management/)