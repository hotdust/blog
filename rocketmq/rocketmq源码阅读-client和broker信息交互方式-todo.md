
todo
client 端是如何实现 block 的呢？
pull 设置 block
int sysFlag = PullSysFlag.buildSysFlag(false, block, true, false);


client 每次发送消息时，都会带一个 opaque 属性，这个是`发消息请求`的 UID。当 Broker 返回信息给 client 时，client 通过消息中的 opaque ，来识别返回的信息是对应哪个`已发出的请求`的。

todo 写一个 消息发送和接收的文章（包括发消息的发送和接收，还有拉消息的发送和接收）。opaque ，NettyRemotingAbstract#invokeSyncImpl。

这种发送/接收方式的好处是什么呢？解耦，不是每一个 client 对应一个 broker，简单地说，如果一个 client 向一个 broker 发请求，另一个 broker 给 client 回也没有问题，只要 ID 对得上（这种想法是不是对，有没有这种需求？）。就像我们用 MQ 实现推送功能。




