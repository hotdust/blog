# Mysql
## 查询
[InnoDB的表类型,逻辑存储结构,物理存储结构 - wade&luffy - 博客园](https://www.cnblogs.com/wade-luffy/p/6288656.html)


## 存储
[InnoDB的表类型,逻辑存储结构,物理存储结构 - wade&luffy - 博客园](https://www.cnblogs.com/wade-luffy/p/6288656.html)


# KV 存储
## 学习
- [如何从零写一个kv数据库？ - 知乎](https://www.zhihu.com/question/59469744)：说了学习写 kv 存储的方法
- [Implementing a Key-Value Store | Code Capsule](http://codecapsule.com/2012/11/07/ikvs-implementing-a-key-value-store-table-of-contents/)：一个教你写 kv 存储的博客
- 

## redis
Redis 内存结构是把各个 key 存在了 Hashtable 中。

Redis 在 rehashing 时，是使用两个 hashtable，和《Dynamic-Sized Nonblocking Hash Tables》很像。
那他们有什么不同呢？Redis 是单线程的，不存在并发问题。
[渐进式 rehash — Redis 设计与实现](http://redisbook.com/preview/dict/incremental_rehashing.html)：讲 Redis 的 rehashing 的文章。

# 临时
## 表的主键为什么要用分布式唯一ID
原因之一，如果主键使用自增主键的话，主库在使用一段时间挂了后，主键到 10000。从库的数据没有完全从主库同步过来，从库的主键到了 9950。这时如果从库变成主库的话，新生成的数据的主键就和主重复了，就会产生问题。使用分布式的唯一ID就不会生成这种问题。


## Index-range locks 或 gap lock 或 Next-Key lock 目的
对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP)”，InnoDB也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁 （Next-Key锁）。 

