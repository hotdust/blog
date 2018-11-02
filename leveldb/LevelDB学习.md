- [浅析 Bigtable 和 LevelDB 的实现](https://draveness.me/bigtable-leveldb)：这个文章非常不错，讲了 Bigtable 和 LevelDB，LevelDB 的结构讲的比较清晰。文章最下面还有一些分布式的东西，`分布式DB`和`分布式事务`。
- [LevelDB handbook](https://leveldb-handbook.readthedocs.io/zh/latest/sstable.html#id1)：对 LevelDB 每一部分都说的很详细，很不错。值得每一部分都读一下。
- [leveldb 源码阅读](https://dirtysalt.github.io/html/leveldb.html#org5fabaf4)：读了每个 .h 文件和其它文件的作用。
- [Leveldb_RTFSC](http://www.grakra.com/2017/06/17/Leveldb-RTFSC/)：一个国人用英文写的 LevelDB 源码阅读文章，其中有一些图，讲的还算细，不错。
- [LevelDB原理探究与代码分析](https://blog.csdn.net/icefireelf/article/details/7515816)：有很多图，可以引用一图。
- [半小时学会LevelDB原理及应用](https://my.oschina.net/yangjiannr/blog/1528532)：一个讲的挺全的，有空看看。
- [RockDB Deep Dive](https://www.percona.com/live/plam16/sites/default/files/slides/myrocksdeepdive201604-160419162421.pdf)：RocksDB 的一个 PPT 看起来不错，还没有看。



源码系列博客：
- [leveldb源码剖析](http://www.pandademo.com/category/tech/leveldb/)：还没具体看


# 个人想法：
1，既然 sstable 是不能修改的，sstable 使用 BTree 结构呢？
BTree 有很多变形，例如：B+Tree、B*Tree。只要把数据部分放在一起，结点部分放在一起，保存到文件中，应该会不错。这样做：
- 可以保证 Compaction 时的归并排序的快速性。因为 skiplist 的数据是顺序存储的，在归并排序时非常快。BTree 也做成数据顺序存储，也可以非常快。
- 还可以提高查询速度。因为 BTree 查询速度是 $log_{m}^{n}$，而 skiplist 是接近 $log_{2}^{n}$。
