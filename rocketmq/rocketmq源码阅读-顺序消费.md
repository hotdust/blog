RocketMQ 的顺序消费和 Kafka 的顺序消费的做法不太一样。

- Kafka 的顺序消费是为 topic 只设置一个 partition，这样消息都进入一个 partition 中，所以消费时从这 1 个 partition 只取得消息，就会不造成乱序。
- RocketMQ 是把 key(例如：orderId) 相同的消息都放到一个 queue 里，消费时 consumer 在同一时间只有一个线程消费这个 queue，其它线程就算要消费同一个 queue，也会因为同步问题等一个线程处理完后，再进行消费。也就是说不能多个线程并行消费单个 queue。下面看一下代码。

```
public class Producer {
    public static void main(String[] args) throws UnsupportedEncodingException {
        try {
            MQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
            producer.start();

            String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
            for (int i = 0; i < 100; i++) {
                int orderId = i % 10;
                Message msg =
                    new Message("TopicTestjjj", tags[i % tags.length], "KEY" + i,
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                // send 最后一个参数 orderId 就是用来区分 queue 的参数。
                // selector 里的最后一个参数 arg 就是 orderId，
                // 用 orderId 对 queue 的大小进行取模，
                // 保证同一个 orderId 的消息发送到一个 queue 里
                SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
                    @Override
                    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                        Integer id = (Integer) arg;
                        int index = id % mqs.size();
                        return mqs.get(index);
                    }
                }, orderId); // orderId

                System.out.printf("%s%n", sendResult);
            }

            producer.shutdown();
        } catch (MQClientException | RemotingException | MQBrokerException | InterruptedException e) {
            e.printStackTrace();
        }
    }
}


public class Consumer {

    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_3");

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("TopicTest", "TagA || TagC || TagD");
        
        // 消费时候，使用 MessageListenerOrderly 接口，保证同一时间只有
        // 一个线程能消费 queue 里的消息。
        consumer.registerMessageListener(new MessageListenerOrderly() {
            AtomicLong consumeTimes = new AtomicLong(0);

            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                context.setAutoCommit(false);
                System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                this.consumeTimes.incrementAndGet();
                if ((this.consumeTimes.get() % 2) == 0) {
                    return ConsumeOrderlyStatus.SUCCESS;
                } else if ((this.consumeTimes.get() % 3) == 0) {
                    return ConsumeOrderlyStatus.ROLLBACK;
                } else if ((this.consumeTimes.get() % 4) == 0) {
                    return ConsumeOrderlyStatus.COMMIT;
                } else if ((this.consumeTimes.get() % 5) == 0) {
                    context.setSuspendCurrentQueueTimeMillis(3000);
                    return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                }

                return ConsumeOrderlyStatus.SUCCESS;
            }
        });

        consumer.start();
        System.out.printf("Consumer Started.%n");
    }

}

```

#一，Client 发送拉取请求
发送拉取请求还是由`RebalanceImpl#updateProcessQueueTableInRebalance`方法产生的。这里没有什么特别的，分配 queue 和之前是一样，每个 queue 分配给一个 client 去处理。

#二，Broker 处理拉取请求
这块处理也和之前正常的拉取处理一样，取得消息就返回。

#三，Client 处理拉取到的消息
这里是实现顺序消费的重要点。不需要顺序消费的话，使用的是 ConsumeMessageConcurrentlyService，而顺序消费使用的是 ConsumeMessageOrderlyService。下面说说 ConsumeMessageOrderlyService 这个类。

##1，ConsumeMessageOrderlyService#start
这个 service 在启动时做了一个事，定时 lockMQPeriodically 方法。而 lockMQPeriodically 方法里面又调用了 `RebalanceImpl#lockAll` 方法，这个方法是做什么用的呢？

##2，RebalanceImpl#lockAll
这个方法处理如下：

- 取得当前 consumer 都在哪个 broker 消费哪些 queue 的集合。然后对每个 broker 进行循环
 - client 告诉 broker  要锁定上面那些 queue 集合。
 - 根据 broker 锁定结果，把可以锁定的 queue 对应的 ProcessQueue 设置成`锁定`状态（是设置 lock 属性为 true）。
 - 把未能锁定的 queue 对应的 pq 设置成未锁定状态，并输出 log。

###（1）client 告诉 broker  要锁定上面那些 queue 集合。
这个处理的作用是：`告诉 broker 哪个 client 要锁定哪个 queue，然后 broker 尝试去锁定 queue，并返回锁定成功的 queue`。也就是说，锁定 queue 的动作可能成功，也可能不成功。主要是防止一个 consumer group 下多个 client 去消费同一个 queue，造成消息乱序消费。具体处理如下（RebalanceLockManager#tryLockBatch）：

- 先判断`client 传过来的 queue 是否已经被 client 锁定`。把已经锁定的 queue 放到“已锁定集合”中，未锁定的放到未锁定集合中。
- 循环未锁定集合，做以下处理：
 - 如果当前 queue 如果没有锁，就创建一个锁。记录锁住这个 queue 的 client。
 - 如果当前 queue 已经被当前 client 锁定了，就把更新锁的时间，把 queue 放到“已锁定集合”中（这步应该是对应上面的操作的）。
 - 如果当前 queue 虽然被其它 client 锁定，但锁已经过期的话，就把锁 clientId 更新成当前 client，并把 queue 加入“已锁定集合”中。代表当前 client 成功锁定 queue 了。
 - 如果当前 queue 其它 client 锁定，但锁还没有过期的话，说明当前 client 无法锁定这个 queue。记录一下 log。

这样 broker 就把 client 能锁定的 queue 给锁住了，并把结果返回给 client。

###（2）根据 broker 锁定结果，把 queue 对应 ProcessQueue 设置成`锁定`状态。
根据 broker 的结果，进行锁定 ProcessQueue。有一些 queue 可能已经被别的 client 锁定的话，这里就无法进行锁定这个 queue 对应的 ProcessQueue了。锁定的原因是，对于 Cluster 模式来说，ConsumeRequest 里只能消费已经被锁定的 ProcessQueue。ProcessQueue 被锁定了，说明已经被 broker 锁定了，可以安全消费了。(对应 `ConumeRequest#run` 中的`(this.processQueue.isLocked() && !this.processQueue.isLockExpired())`部分代码)

###（3）把未能锁定的 queue 对应的 pq 设置成未锁定状态，并输出 log。
根据 broker 的返回结果，可能一部分 queue 无法锁定，把无法锁定的 queue 对应的 ProcessQueue 设置成未锁定状态，然后输出 log。


##3，ConsumeMessageOrderlyService#ConsumeRequest
在 PullCallback 处理中，取得消息后，会调用`consumeMessageService.submitConsumeRequest`方法来处理消息。顺序消费使用的是 ConsumeMessageOrderlyService 类，所以处理方法就得看`ConsumeMessageOrderlyService#ConsumeRequest`的 run 方法了。下面说一下具体的处理内容。

- 根据传进来的 MessageQueue，使用 synchronized 锁定具体的 queue。
- 判断 ProcessQueue 是否被锁定，如果被锁定才进行处理。（这就是上面说的锁定 ProcessQueue 的原因）
- 进行各种 check
- 使用 ProcessQueue 里的 consumeLock 进行加锁。这个锁在 `RebalancePushImpl#removeUnnecessaryMessageQueue`方法中使用。那个方法作用是让 broker 解除对 mq 的锁，不太明白为什么要使用这 consumeLock。
- 进行消费消息
- 处理消费结果


###（1）同步的使用
在处理消息前做了两次同步锁：

- messageQueueLock.fetchLockObject(this.messageQueue); synchronized (objLock) {...}：这个是对 MessageQueue 进行锁。保证一个 queue 只有一个线程可以消费。
- this.processQueue.getLockConsume().lock()：这个是对 MessageQueue 对应的 ProcessQueue 里的消息进行锁。为什么要锁住 pq，锁住 mq 不能保存安全吗？这个锁在 RebalancePushImpl#removeUnnecessaryMessageQueue 方法中使用，那个方法作用是让 broker 解除对 mq 的锁，不太明白为什么要使用这 consumeLock。

###（2）处理消费结果
在消费完消息后，处理消费结果。这里的逻辑还是有点复杂的：

- 如果是`自动 commit`
 - 如果消费成功的话，就对 ProcessQueue 进行 commit，也就是把 msgTreeMapTemp 清空（因为这里保存的是被消费的消息），并设置 commitOffset。
 - 如果是 SUSPEND_CURRENT_QUEUE_A_MOMENT（等一会再消费），就把 msgTreeMapTemp 里的消息退回到 msgTreeMap 里，等一会再提交一个 ConsumeRequest 进行消费。
- 如果不是`自动 commit`
 - 如果消费成功的话，什么也不做。（问题：那用户什么时候可以进行 commit 呢？如果不进行 commit 的话，msgTreeMapTemp 是无法清空的。下次再从 msgTreeMap 里向 msgTreeMapTemp 里写消息时，前面消费过的消费还在，就重复消费了）
 - 如果是 SUSPEND_CURRENT_QUEUE_A_MOMENT（等一会再消费），就把 msgTreeMapTemp 里的消息退回到 msgTreeMap 里，等一会再提交一个 ConsumeRequest 进行消费。
- 最后，如果 commitOffset 被更新了，就提交 commitOffset。

其中枚举类型还有 commit 和 rollback，但这两个类型都不建议使用了，原因不太知道。


ConsumeRequest 的 run 方法运行完，同步处理也就结束了。所以`保证只有一个线程进行消费`是通过对`MessageQueue`的同步进行的。在处理中时候（可能处理比较慢)，又重处理的 queue 取得新的消息了，那么新的消息的 ConsumeRequest 只能等待，因为他们具有相同的 MessageQueue。这样就保证了同时只有一个线程能处理这个 queue。而且，就算处理失败了也不会把消息发回 broker，只是在 client 端进行重新消息。因为如果发回 broker 继续处理其它 queue 消息的话，就乱序了。

另外，每个 ConsumeRequest 会把 ProcessQueue 里的消息都处理完。所以可能新的消息到来后，放到了 ProcessQueue 里，被正在运行的 ConsumeRequest 给消费掉了。而随着新的消息创建的 ConsumeRequest 可能会空执行一次。




#细节：
1，broker 在锁定 queue 时，锁设置有一个过期时间，防止一个 client 死了不放锁。而其它 queue 也无法消费。

<br>
2，对于多线程消费问题。ConsumeMessageOrderlyService 中也有线程池，默认 coreSize: 20、maxSize: 64。每个 ConsumeMessageOrderlyService 对应一个 consumer，也就是说一个 consumer 默认就开启 20 个线程。MQPushConsumer 接口有设置线程池大小的方法，而 MQPullConsumer 就没有。但 ConsumeMessageService 接口提供了修改线程池大小的方法（但它的实现类都没有实现这些方法）。

<br>
3，RebalanceImpl 中的 lockAll 和 unlockAll 分别在 ConsumeMessageOrderlyService 进行 start 和 shutdown 时候调用。说明 ConsumeMessageOrderlyService 正常情况下，都是锁定后再进行操作的。

<br>
4，关于 ProcessQueue 类
（1）msgTreeMapTemp 属性
这个属性就是在顺序消费时使用的属性。主要是用来保存"要被 listner 消费的消息"，当消费不成功时，方便再把这些消息放回到 msgTreeMap 属性中。

（2）takeMessags 和 makeMessageToCosumeAgain 方法
这两方法一个是从 msgTreeMap 取得要消费的消息放到 msgTreeMapTemp 里，另一个是把 msgTreeMapTemp 里的消费放回到 msgTreeMap 里。都是只在顺序消费时使用的方法。

（3）msgTreeMap 是一个优先级 Map，排序的方式是以 offset 进行排序的。因为是顺序消费，所以要以 offset 进行排序。

<br>
5，RebalanceLockManager 类
这个类是工作在 Broker 端的类。这个类的主要作用是，记录哪些 queue 被哪些 client 给锁住了。信息都是保存在内存中。

如果一个queue 在被锁的状态中，并且锁没有过期，另一个 client 是无法锁住。client 在锁之前都要和 RebalanceLockManager 进行交互，来取得哪些 queue 可以锁住。

设计是使用的 lease 机制。client 每一段时间都去 broker 更新一下状态，让自己不超时。

<br>
6，MessageQueueSelector 的默认实现
从代码上来看，MessageQueueSelector 默认实现有 3 个：

- SelectMessageQueueByHash：对 arg 参数进行 hash，然后使用 hash 值对 queue 的个数进行取模运算，来选择 queue。
- SelectMessageQueueByRandoom：取个随机值，对 queue 的个数进行取模运算，来选择 queue。
- SelectMessageQueueByMachineRoom：没有实现。

<br>
7，顺序消息的话，是不是只能设置一台 Master 呢？
不是，可以设置多台的。假如有 2 台 Master，都有一个 testTopic，每个 topic 都是有 4 个 queue。在 client 端得到的 topic 的 queue 就是 8 个。在发送消息时，是对 8 进行取模运算，选择向哪个 Queue 发消息。这样就可以保证每种 key 只会发到同一个 Master 的 Queue 上。

把两个 Master 的 Queue 进行合并的处理操作，是在`MQClientInstance#topicRouteData2TopicPublishInfo` 方法中进行的。




#特点：
1，对 queue  的锁定使用的是 lease 机制。client 每一段时间都去 broker 更新一下状态，让自己不超时。

2，顺序消费是使用`同一时间只有一个线程能消费 queue`的方式来实现的。其实也是有线程池的，线程池的作用是，让多个线程同时处理不同的 queue。虽然一个线程只能处理一个 queue，
但多个线程可以处理多个 queue。
例如：topic 有 64 个 queue，如果使用 64 个线程，就可以同时处理这 64 个 queue 的顺序消费。如果像 kafka 一样只使用一个 partition 的话，一个 consumer 只能顺序消息一个 partition。如果使用多线程处理 partition 里的数据的话，有可能造成乱序问题。

3，对于顺序消息，即使有多台 Master，可能保证只向一台 Master 中发数据。

4，

问题：
1，processConsumeResult 中如果不是 autoCommit的时候，status 是 SUCCESS 的话，什么也不做。如果不进行 commit 的话，msgTreeMapTemp 是无法清空的。下次再从 msgTreeMap 里向 msgTreeMapTemp 里写消息时，前面消费过的消费还在，就重复消费了。那用户什么时候可以进行 commit 呢？


参考：
https://www.zhihu.com/question/30195969
https://my.oschina.net/xinxingegeya/blog/956370