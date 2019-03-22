#一，延迟消息发送和接收
**Client：**
想发送延迟消息的话，只要调用`setDelayTimeLevel`方法，设置 delay level 后，发送的消息就是延迟消息。例如：
```
Message msg = new Message("TopicTest" /* Topic */, "TagA" /* Tag */...);
msg.setDelayTimeLevel(3);
```

**Broker：**
RocketMQ 延迟消息处理方式是：在 broker 接收到延迟消息后，根据 delay level 放到相对应的 delay queue 中。然后为每个延迟消息设置定时任务，到时间后就把消息从 delay queue 里取出来，放到消息应该存放的队列中。下面进行详细说明一下。

#二，延迟消息接收处理细节
##1，延迟 Topic 构造
延迟消息并不是直接放到对应的 topic 和 queue 中的，而是首先放到`延迟 Topic`中。延迟 Topic 下的每个 queue 对应着一个延迟时间。下面是默认延迟时间配置：
```
private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m" + 
                                   "10m 20m 30m 1h 2h";
```
`1s` 就是第 1 个 queue，它的延迟时间是 1 秒。`1m` 是第 5 个 queue，延迟时间 5 分钟。这么看的话，延迟 topic 下有一共有 18 个 queue。

##2，延迟处理

###（1）接收消息
在 broker 接收到消息后，先检查消息的 delay level 属性是不是大于 0。如果大于 0，就说明是延迟消息，然后 broker 把消息放到延迟消息专用的 topic 中。具体做法是，把消息的 topic 和 queueId 当做属性保存到消息里，然后按下面方式修改 topic 和 queueId：

- topic：SCHEDULE_TOPIC_XXXX
- queueId：delayLevel - 1

###（2）延迟服务
ScheduleMessageService 是关于延迟消息的 service。它启动时，会对每个 delay level 启动一个定时任务，每个定时任务的第一次的启动时间为“1 秒后”，之后为 100ms 启动一次。大部分处理都在定时任务中，下面说一说定时任务的处理。

- 当定时任务启动后，会通过 delay level 取得相对应的 consumer queue，然后根据 offset 把 offset 之后的数据取出来。
- 取出消息后，检查每条消息是否已经到了可以发送的时间，如果到了时间，就把消息发送到原来应该发送到的 topic 和 queue 中；如果时间没有到，为这条消息设置一个定时任务，启动时间为`从当前时间开始，距离消息可以发送的时间差`，例如：如果一个延迟 10 秒的消息，从当前时间计算 3 秒后才能发送，启动时间就设置为 3 秒。
- 如果发送消息到原来 topic 成功，就保存 queue offset。
- 如果发送消息失败，就为这条失败消息做一个定时任务，在一定时间（默认 10s)后，再重新处理这条消息。（感觉这里可能会产生问题。）

##3，细节：
###（1）关于延迟消息的`计划消费时间`
当定时任务启动后，判断延迟消息是否到达了`计划消费时间`时，这个`计划消费时间`是从 tagsCode 属性里取得的。tagsCode 应该保存的是 tags，为什么从这里取得`计划消费时间`呢？因为在把消息写完后，在写到 consume queue 之前，把 tagsCode 属性设置成了`计划消费时间`（在`CommitLog#checkMessageAndReturnSize`方法里做的）。而原本的 tagsCode 被当成属性保存到消息里了。

`计划消费时间`是一个绝对时间，是由 `broker 当前时间 + 延迟时间`组合而成的。

###（2）如果 client 指定的 delay level 在 broker 不存在怎么办？
如果 client 指定的 delay level 在 broker 不存在，把 delay level 设置成`最大的delay level`。（在 CommitLog#putMessage 方法中做的）

###（3）delay level 和 延迟 queue
每个 delay level 的消息保存到哪个 queue 里呢？对应关系为：
`queueId = delay level - 1`



todo 和 activemq相比，处理方式有什么不同？

问题：
1，当 executeOnTimeup 方法中，从延迟 queue 取得消息，发送消息到原来 topic 时失败（putMessageResult != null && putMessageResult.getPutMessageStatus() == PutMessageStatus.PUT_OK）时，会不会发生重复发送消息问题？即失败这条消息后面的延迟消息发送成功了，但重新处理失败消息时，会把失败消息后面的消息再重新发送一回。








