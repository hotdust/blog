# Chapter 1

## Abstractions and models
Models 如下：
- System model (asynchronous / synchronous)
- Failure model (crash-fail, partitions, Byzantine)
- Consistency model (strong, eventual)

## Partitioning 的作用
原文如下：
>
Partitioning improves performance by limiting the amount of data to be examined and by locating related data in the same partition
Partitioning improves availability by allowing partitions to fail independently, increasing the number of nodes that need to fail before availability is sacrificed

优点：
- Partitioning 通过把有关联的数据放到一起，加快数据访问速度。
- Partitioning 增加了 availability。因为每个 Partition 都是独立的。

缺点：
- 当没有办法使用 key 条件时，会对所有的 Partition 都进行访问。

## Replication 的作用
原文如下：
>
Replication is about providing extra bandwidth, and caching where it counts. It is also about maintaining consistency in some way according to some consistency model.
>
Replication allows us to achieve scalability, performance and fault tolerance. Afraid of loss of availability or reduced performance? Replicate the data to avoid a bottleneck or single point of failure. Slow computation? Replicate the computation on multiple systems. Slow I/O? Replicate the data to a local cache to reduce latency or onto multiple machines to increase throughput.
>
Replication is also the source of many of the problems, since there are now independent copies of the data that has to be kept in sync on multiple machines - this means ensuring that the replication follows a consistency model.

总结一下：
优点：
- 提高 scalability、performance 和 fault tolerance。
- 避免单点产生的可用性问题。
- 因为数据在不同的机器上，可以使用不同机器上进行计算。
- 避免数据 IO 问题，可以从多个复本上进行 IO 读取。

缺点：
- 数据一致性问题。因为有多个复本，所以在同步数据时会产生数据一致性问题，所以要选择哪种一致性：最终一致性、线性一致性等。

# Chapter 2
## 

## 问题：
1，什么是 Byzantine 问题？
