# 为什么需要 NameServer，而不使用 ZK
1，name server 符合 AP 原则，即使有过半节点不能使用，还是可以运转的。ZK 是 CP，过半节点不能使用的话，就停止服务了。
2，rocketmq 不需要选举，所以不需要 ZK 的选举功能。
3，负责通知 slave，告诉他 master 地址是多少，这样 slave 就可以和 master 进行同步数据。
4，负责维护 broker 的可用性。当 broker 不可用时，告诉 producer 和 consumer 不要连接这个 broker。
   如果不在 name server 里做的话，这些处理就需要在 producer 和 consumer 里去做。
   
参考：[RocketMQ观后感--NameServer - JJB的博客 - CSDN博客](https://blog.csdn.net/qq_27529917/article/details/79871052)
