# Concurrency Control Protocol
1，4 种版本控制有什么不同点


# Version Storage
## 1，这两种 old-to-new，new-to-old 有什么不同。
Append-only storage can have two kinds of version chains: oldest-to-newest and newest-to-oldest


## 2，Time-travel storage 是怎么回事。

## 3，delta storage (MySQL, Oracle, and HyPer) stores




# 关于 MV2PL
MV2PL 就比 2PL 多了一个 read 不加锁的功能。
使用 No-Wait 策略的 MV2PL 不会死锁。

MV2PL 在写时，会生产一条新的记录。在提交时，会看看原记录是否还有正在 read 的事务，如果有就等待；如果没有就更新。


2V2PL
只有 CLock 出现，才会进行提交确认。如果有更新的话，在 CLock 出现后，需要看要被更新的数据 X 有没有其它 Read 操作。如果有的话，需要在 Read 操作完成后，才进行更新操作。


Mysql 只有隔离级别是 serializable 时，才是 MV2PL。
可以测试，3 个事务，2 个进行查询，一个进行 update。只有查询的事务 commit 后，update 才能正常进行。