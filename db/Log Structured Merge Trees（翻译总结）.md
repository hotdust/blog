> 本文是对 [Log Structured Merge Trees](http://www.benstopford.com/2015/02/14/log-structured-merge-trees/) 的翻译和总结。这个文章是在 [Log Structured Merge Trees译文以及LSM调研心得](http://weakyon.com/2015/04/08/Log-Structured-Merge-Trees.html)  看到的。本来没想要翻译，翻译的原因是：1，想要理解的更深一些。2，看中文和英文还是感觉不一样，有一些关键字被翻译中文的话，有时候不好理解。可能因为个人水平原因，原文感觉不是很好理解，在翻译过程中很多地方参考了上面的中文翻译，感谢。
 
自从谷歌发表了他的”big table”论文以来已经过去了十年。论文中很酷的一点是他使用的文件组织方式。在1996年后，这个方式以LSM树被人所熟知（尽管并未被谷歌特别的提出）。

LSM现在被用于很多产品的主流文件组织策略。HBase,Cassandra,LevelDB,SQLite,甚至是MongoDB收购了Wired Tiger引擎后，其3.0附带了一个LSM引擎的选项。

使LSM树有趣的是他们离开二叉树风格文件组织主导的领域已经有几十年的历史了。当你第一次看到LSM，它看起来几乎是违反人类思维的，然而当你考虑到文件是如何工作在现代笨重的存储系统上一切就说得通了。

# 一、一些背景知识
简而言之，LSM树被设计来提供比传统的B+树和ISAM更好的写入性能。它通过消除随机的就地更新操作来实现这一点。

所以为什么这是一个好主意呢？其核心是磁盘的老问题：随机操作缓慢,但当顺序访问时很快。这一点分歧存在在两种不同的访问方式中间，不管磁盘是固态还是磁性材料，尽管这种影响在主存较小。

这一点在ACM的这篇报告中指明了。ACM报告如下：

![在这里插入图片描述-m](https://img-blog.csdn.net/20181019125728638?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

随机和顺序IO的磁盘吞吐
![在这里插入图片描述-m](https://img-blog.csdn.net/20181019125736229?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


他们表明的是违反直觉的，顺序操作在主存上比随机操作更快。他们也进一步表明了，在磁性或固态硬盘上，顺序操作比起随机操作快上三个数量级。这意味着随机操作应当被避免，顺序操作很值得设计。

所以记住这个考虑一个小猜想实验：如果我们对写吞吐量有兴趣，什么是最好的使用方法呢？一个好的出发点是鉴定的追加数据到文件里。这种方式经常被叫做日志，档案或者堆文件，是完全顺序化的因此能提供非常快的相当于理论硬盘速度的写性能（通常平均200-300MB/S）

受益于基于简单性和高性能的日志方法正在许多大数据工具中变的十分受欢迎。然而他们有一个很明显的缺点。从日志中读任意数据会消耗远多于写入的时间，这当中包括了反向的扫描，直到找到需要的关键字。

这意味着日志只适用于数据作为整体被访问的简单情况，例如大部分数据库的预写日志,（WAL,write-ahead logging),或者通过一个已知的偏移量来访问。简单的消息系统Kafka就是这么做的。

所以我们需要不只是日志这样的方式来高效执行更多的复杂读工作，例如基于关键字的访问或范围访问。一般来说，有四种方法在这种情况下是有用的：二分查找，哈希，B+或者外部文件。
> 这之上的内容是拷贝的：[Log Structured Merge Trees译文以及LSM调研心得](http://weakyon.com/2015/04/08/Log-Structured-Merge-Trees.html)


- 二分查找：保存数据到文件的时候以关键字排序。如果数据已经确定`宽度`就使用二分查找；否则使用`page index + scan`（page 意思应该是缓存的意思），应该是因为无法确认宽度，只好利用 Index 确认某个文件包含数据，然后从这个文件开始向后扫描，直到找到要找到数据。
- 哈希:用哈希函数把数据分片，而后可以直接访问读取。一般索引使用 hash 方式，为了能够快速定位。
- B+：使用文件组织如B+树，ISAM等等
- 外部文件：除数据本身按日志形式存储外，另对其单独建立索引加速读取。

这些方式增加了`读性能`( n->O(log(n) )，但这种带有顺序（或排序）的结构影响了`写性能`，无法进行高速写日志。

![在这里插入图片描述](https://img-blog.csdn.net/20181019095136629?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

这 4 种方式都使用了一些`重要的结构`在数据上。数据被巧妙地写到文件里，好让 index 可以快速的找到他。这些结构在数据被写入后非常有用。但这样的写入方式，也会因为增加磁盘的随机写，而降低写的速度。


BTree 有几个具体的问题：
- 第一个问题，每次写数据都需要`两次 IO`，一次读数据（从磁盘上读取要写的数据前后的数据），一次写数据（写到读取到数据的前后位置）。这种情况不会发生在写`log/journal (WAL)`文件上，因为写文件是顺序写，所以只需要一次写 IO。
- 第二个问题，我们需要更新 hash 或 B+ index 的结构，这意味着要更新文件的某一部分。这种方式众所周知，会产生`in-place`更新 (update-in-place)，而且是`很慢的随机 IO`。这点很重要：`这种 in-place 的更新方式，像散弹枪的子弹一样，会在文件系统各处发生（这样到处更新的处理，是非常慢的）`，这就是问题的所在。
  * 解释1：`in-place`是指在读取到数据的地方进行更新，更新成新的值，更新时不会用到大块内存。像归并排序就不是`in-place`的，因为结果要输出到一个新的地方。而插入排序是`in-place`的，它在原数据上进行更新。而在文章中`in-place`的意思感觉是想说明，这种更新不是顺序写，而是随机更新文件系统的某个位置的数据。
  * 解释2：`这种 in-place 的更新方式，像散弹枪的子弹一样，会在文件系统各处发生`，这句的原文是：`in-place approaches like this scatter-gun the file system performing update-in-place`。感觉这里是想说，更新会到处发生，会产生随机写 IO。

一个通用的解决办法是，使用`方式（4）`：为 journal 创建一个 index（索引），但把 index 保存在内存中。例如：Hash Table 可以把 key 映射到`在 journal 文件上的那个最新值的 position 或 offset`。这种方式把随机IO `限制`成小一些：`key 到 offset 的映射`被保存在内存中（这部分不会产生随机 IO），通过 offset 查找 value （只会产生这一次 IO）。

另一方面，有一些扩展限制，特别 value 值小而且还很多的时候。如果 value 是一些简单的数值的话，index 可能会比`数据文件`还要大。这种情况是可以理解的，并且很多产品（Riak、Oracle Coherence）都对这种情况妥协了。

所以，这给我们带来了 Log Structured Merge Trees。LSMs 带来了和以上 4 种方法不同的解决方式。这种方式以磁盘为中心，为了提高效率会使用一点内存，但还会依靠 journal 文件来保证`写性能`。一个缺点就是，和 B+ 树相比，读性能稍微差了那么一点点。
> 本质上，所有做的一切都是了为“磁盘顺序访问”，无“随机访问”。

一些已知的树结构不需要`update-in-place`。最流行的是 append-only Btree，也被称作 copy-on-write tree。工作的方式是，每次写操作发生在文件尾部时，顺序地重写树结构。old tree 的相关部分，包括 top levle node，被孤立起来（不了解 copy-on-write tree 原理，所以不知道具体意思。。。）。通过这种方式，tree 顺序地重新生成他自己随着时间，`update-in-place`被避免。这种方式也会有一些个代价：每次写操作时，都要重写结构，这个过程很冗长。而且带了很显著的写操作。
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181212180613882.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=,size_16,color_FFFFFF,t_70)
 
 
 
# 二、Base LSM 算法
## 1，基础
从概念上来说，base LSM 非常简单。 和`大的 index 结构`（那种分散更新文件系统，并显著增加写的结构）不同，批量写操作被“顺序地”保存到一组`小的 index 文件`。这样每个文件都包含一批在变化，这些变化是一小段时间内的变化。每个文件在被写之前就都被`排序`过了，所以之后再搜索他的时候会很快。这些文件是不可变的，不会被修改，新的变更会放到新的文件中。会定期把文件合并到一起，减少文件个数。

## 2，细节
让我们看看细节。当更新到来时，会被添加到`内存 buffer`中，内存 buffer 通常使用树结构（例如：红黑树）来保持 key 的顺序。这个`内存 buffer`被称为`memtable`，这个 memtable 也会以 write-ahead-log 的方式写到磁盘上一份（磁盘上的 write-ahead-log 可能是顺序写），为了恢复时候使用。
当 memtable 装满`被排序的数据`后，会被保存到一个新文件。这个过程会随着`写操作`到来，不断重复。很重要的是，当文件不允许更新的话，系统就会只做`顺序写`。新的更新和条目简单地创建连续的文件就OK了。

随着数据越来越多，越来越多的`不可变的`、`排序过的`文件被创建。每个文件都是一个`小的`、`按时间顺序`排列的`被排序过`的文件。既然旧文件无法更新，要更新的数据就会创建一个新的数据，来取得原来的数据（或者使用"删除标记"的方式）。这种方式带来了一些冗余。

系统会定期地执行 compaction 。Compaction 把多个文件合并起来，去掉旧的值（对于已经生成新的数据的值）。这一步非常重要，这会去掉上面说的`冗余`。更重要的是`会增加读性能`，因为文件数增加会`降低读性能`。值得庆幸的是，因为文件是排序过的，所以`归并`排序会非常快。

当读操作来时，系统会先检查 memtable。如果没有在 memtable 里找到 key，就会`逆时间顺序地`检查每一个文件（应该是先检查最新，因为更新的值保存在最新的文件中，从最老的文件找到已经被更新的值是没有意义的），直到找到这个 key。每个文件都是被排序的，所以非常方便查找。但是读性能会随着文件数的增多，变的越来越慢。因为每个文件都要被检查。

所以对于读操作，LSM tree 会比其它的`in-place`方式（BTree等）慢。幸运地是，有一些技巧来提高性能。最通常的方法是，保存一个`page-index`在内存中。这会让你离“想要查找的 key”更近（也就是找的更快）。你扫描的数据都是排序过的。LevelDB、RocksDB 和 BigTable 会创建一个`block-index`在文件结束部分。这种方式会比`二分查找`工作的更好，因为它允许使用`变长`字段，这些变长字段也更适合被 compaction 的数据（可能是用变长字段保存`被 compaction 的`字段更方便）。

即使文件被 index 了，`读性能`也会随着文件增加变得越来越慢。通过定期合并文件，来制约文件的增长。通过`Compaction `控制文件数量，保证`读性能`在一个可接受的范围内。

即使使用 compaction ，读操作还会访问很多文件。大部分的实现会通过 Bloom filter 来避免这个问题。Bloom filter 在判断`文件是否包含 key`方面，是一个非常有效的基于内存算法。

从`写操作`的角度来看，所有的`写操作`都被`批量处理`，并且`顺序地`写。另外，每一轮的 compaction 处理，都是对 IO 的损耗（不知道为什么来这么一句，感觉很突兀。。。）。然而读操作可能会在`读取一行数据`时，要读取一堆文件，这也是算法（Bloom filter）应该起作用的时候。我们用增加`随机读`的方式，来避免`随机写`的发生。这种交换是合理的，如果我们可以使用`bloom filters`和`hardware tricks（例如：linux page cache）`来优化读性能。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181212180724431.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=,size_16,color_FFFFFF,t_70)

# 三，Compaction
## 1，Basic Compaction
为了保持 LSM 读性能，在`减少文件数量`方面非常重要。让我们更深入地看一下 compaction 过程。这个过程有点像`垃圾回收`。过程如下：

当一定数量的文件被创建，例如：5 个文件，每个文件有 10 行。这些文件被合并到一个文件中，这个文件有 50 行。每当有 10 行数据被创建，这个过程就会持续进行。每当有 5 个文件被装满，就会把这些文件合并到一起，最终创建出一些有 50 行的文件。这时，50 行的 5 个文件又被合并成一个 250 行的文件。这个处理持续进行，来创建更大的文件。如下图：
![在这里插入图片描述](https://img-blog.csdn.net/20181018092719462?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

上面提到的关于这个方式的问题是，文件数量非常大：`所有文件`要被搜索，来创建最后的结果（最坏情况下）


## 2，Levelled Compaction
LevelDB, RocksDB 和 Cassandra 等 DB 对于这个问题的新的实现是：基于`层`的、而不是基于`大小`的方式来 compaction 。这个方式`减少了文件数量`，也就减少了对读性能的影响，而且还减少了单个 compaction 的相对影响。

这种`level-base`的方式，和上面的`Basic Compaction`相比有两点不同。
1. 每个 level 包含一些文件，这些文件不会有`重复的 key`。也就是说，key 是被分区存储（例如: hash）在这些文件上的。因此在某个 level 上找一个 key 的话，只需要访问一个文件。第一个 level 是个特例，这个 level 没有上面说的限制，key 可以保存到多个文件中。
2. 一次一个文件被合并到`上层 level`。当一个 level 被装满，从这个 level 中取`一个文件`，然后合并到`上一个 level`中。而`上层 level`会为这些数据创建更多的空间。这和`Base Compaction`略有不同，`Base Compaction`是把`几个大小相似的文件`合并成一个大文件中。

这些不同点意味着，`level-based`方式是`随着时间变化`来`分散` compaction 的影响，并且需要的空间变少了。在读性能方面也更好了。但是在大部分的工作的`总的 IO`更高了，这也意味着对`单纯的写处理工作`没有什么帮助。

# 四、总结
LSM tree 处理于`journal/log`文件 和 `传统的、单一的、定长的 Index （例如：B+ tree 或 hash index）`之间的位置。LSM tree 提供了一个机制，来管理一组`小的、独立的 index`文件。

通过管理`一组 index` 而不是 `一个 index`，LSM 方式把` B+ 或 hash index 算法的 update-in-place 方式`产生的`昂贵的随机 IO`，变成`快速的、顺序IO`。代价是：1，`读操作`不得不在很多文件中进行查找，而不是从一个文件上查找。2，Compaction 产生的 IO 操作。

如果还有一些疑惑，请看[levelDB 实现]()和[Leveled Compaction in Apache Cassandra](https://www.datastax.com/dev/blog/leveled-compaction-in-apache-cassandra)

>第一个连接已经失效，原链接为：http://leveldb.googlecode.com/svn/trunk/doc/impl.html

# 五、关于 LSM 的一些思考
> LSM 真的比传统的`单一树`好吗？

我们已经看到，LSM 的写性能更好，尽管有一些代价。LSM 还有一些其它的好处。LSM tree 创建的 SSTable（排序过的文件）是`不可变`的。这让`锁`这个语义变的更加简单。发生竞争的资源只有 memtable。这和`单一树`相比形成反差，`单一树`需要精细的锁机制，来管理不同 level。

所以问题是，`面向写优先`的工作负载是怎么样？如果你关心写性能，那么 LSM 对你的帮助是很大的。大型的互联网公司看起来相当中意这点。例如，Yahoo 因为`事件 log`和`手机数据`的大量增长，从`大量读`转向了`读写负载`。很多传统的数据库公司仍然喜欢`读优化`的文件结构。

对于`Log Structured`文件系统，争论的焦点源于`不断增长的可用内存`。当有更多的可用内存时，读操作很自然地被优化了，通过操作系统提供的大量文件缓存（应该是 linux page cache）。写性能（无法通过增加内存提高）因此变成主要被关心的点。换句话说，硬件提高方面做的更多的是`提高读性能`，而不是`写性能`。因为`选择对写操作进行优化`是非常合情理。

例如 LevelDB 和 Cassandra 的 LSM 实现提供`更好的写性能`和`基于单一树的方法`（[看这](https://s3-eu-west-1.amazonaws.com/benstopford/nosql-comp.pdf)和[看这](https://symas.com/lmdb/technical/)）

# 六、超越 Levelled LSM
现在有很多基于 LSM 方法的东西准备去做。Yahoo 开发了一个叫做“Pnuts”的系统，这个系统结合了 LSM 和 BTree，并且声称性能更好，但我没有看到这个算法被发表出来。IBM 和 Google 最近做了很多工作在类似的分支上。还有很多相关的方法，这些方法也有相似点，但保持了总体的结构，例如：Fractal Trees 和 Stratified Trees。

LSM 结构只是一个选项，数据库使用了很多的不同选项。数据库提供`可插入的引擎`为不同的工作场景。Parquet 是一个非常流行的 HDFS 的替代品，在和 HDFS 相反方向上进展非常多（例如：通过`列格式`提高`聚合性能`）。MySQL 有一个`存储抽象`，这个的抽象可以使用很多不同的引擎，例如：基于 Toku fractal tree 的 Index（MongoDB 也可以使用）。Mongo 3.0 加入了 Wired Tiger 引擎，这个引擎提供了 B+ 和 LSM 方法，这些方法和遗留引擎也兼容。一些关系型数据库有`可配置的 Index 结构`，这些结构使用不同的文件组织。

也可以考虑硬盘的使用。贵的 SSD 磁盘，例如 FusionIO，有非常高的随机写性能，适合`update-in-place`方式。便宜的 SSD 和机械磁盘适合 LSM，LSM 避免`小的、随机的访问`。

LSM 不是没有问题。它的最大问题（例如 GC）是，`collection phases`和`对宝贵的 IO 的影响`。这个有趣的讨论在[这里](https://news.ycombinator.com/item?id=9145197)。

如果你正在看一个数据产品，比较 BDB vs. LevelDB、 Cassandra vs. MongoDB，你可以更多的关注产品使用的`文件结构`，而不是只关注`相对性能`。关注性能的测试值的观点似乎和前面说的观点不一致。当然，还是关注一些性能的权衡。

> 在 SSD 磁盘上，每个写操作对于每个 512k block，都会引发一个`clear-rewrite cycle`。因此`比较小的写操作`可能在磁盘上引发`不成比例的搅动`。当 rewrite 操作达到了预定的限制后，可能会严重影响磁盘的生命。

# 七、个人总结
## 1，LSM 和 BTree 比较
**LSM 优点：**
1，LSM 写 WAL 的目的只是为了恢复使用。
2，使用 Bloom filter 减少文件的访问（ memtable 里没有 key 的话，就需要访问 filter）
3，使用 index 增加对外部文件的访问（level 里的文件），并且把 Index 缓存到内存里。
4，写到磁盘的文件，都是排好顺序的，可以加快文件的访问。
5，写文件使用“顺序写”的方式：
   - 顺序写比随机写快很多。
   - LSM 结构顺序写操作，不需要在写之前读数据，减少 1 次 IO。BTree 写操作之前，要先读取数据，所以有 2 次 IO。

6，通过 Compaction 减少文件个数，增加读性能
7，使用 page cache 增加读速度。
8，减少锁的使用，发生竞争的地方只有 memtable（分布式的优点，减少锁的使用。BTree 可能会发生行锁或表锁）

**LSM 缺点：**
1，读效率比 BTree 慢。读操作不得不在很多文件中进行查找，而不是从一个文件上查找。
2，Compaction 产生的 IO 操作。这种 IO 操作多时，会造成`写放大(write amplification)`，造成`读`和`写`变慢。


## 2，进行的写优化
1，写文件使用“顺序写”的方式：
   - 顺序写比随机写本身就快很多。
   - LSM 结构顺序写操作，不需要在写之前读数据，减少 1 次 IO。BTree 写操作之前，要先读取数据，所以有 2 次 IO。

2，减少锁的使用，发生竞争的地方只有 memtable（分布式的优点，减少锁的使用。BTree 可能会发生行锁或表锁）

**顺序写的地方**
1，写 WAL（顺序写）
2，写第一层　 Level 文件（顺序写）
3，写第其它层 Level 文件（归并排序，也是顺序写）
4，数据更新也是顺序写操作。更新和删除都是增加一条新的数据。


## 3，进行的读优化
总结一下对读的优化：
1，被保存的文件，内容都是排好顺序的。
2，使用 Index 来快速定位外部文件，并把 Index 缓存到内存中。
2，使用 page cache。
3，使用 Bloom filter，减少文件读取。

## 4，为什么要增加“通过减少读性能”来“增加写性能”
1，写需求量变大
2，读需求可以通过各种优化来提升，但写需求很难提升。

## 5，为什么“读取多个文件”比“读取单个文件慢”
1，多个文件保存在不同的地方，磁盘寻址时间变长。单个文件的内容可能存储在不同区域，也可能存储在一个区域里，如果存储在一个区域里，顺序读不需要来回寻址，所以快。
2，每个文件都有 inode，读取多个文件就要读取多个 inode。一是数据量多，二是读取 inode 多了，产生的随机读取也多了，产生的影响和 1 说的问题类似。

参考：
- [为什么单个大文件比总体积相同的多个小文件复制起来要快很多？](https://www.zhihu.com/question/22555963)
- [理解inode](http://www.ruanyifeng.com/blog/2011/12/inode.html)
- [leveldb前奏--理解文件I/O](http://gao-xiao-long.github.io/2016/04/13/file-io/)


## 6，Levelled Compaction 和 Basic Compaction 比较的优点：
1，减少文件数量，加快文件读取速度。

## 7，Btree 为什么比 LSM tree 的读性能高
### （1） 时间复杂度
**BTree 时间复杂度**
二叉树的时间复杂度为：$log_{2}^{n}$。而 BTree 是多叉树，如果分叉为 m，则时间复杂度为 $log_{m}^{n}$。如果 m > 2，而 $log_{m}^{n}$ < $log_{2}^{n}$，因为 BTree 的 m一定大于 2，所以 BTree 时间复杂度要小于二叉树。

**LSM Tree 时间复杂度**
一共有 n 个数，每 m 个数据成型一个 sstable。sstable 的数据结构是类似跳跃表的方式，平均时间复杂度为：$log_{2}^{n}$。那么最差就要对 n/m 个 sstable 进行 $log_{2}^{n}$的查找，也就是 (n/m)$log_{2}^{n}$。

**比较**
$log_{m}^{n}$ < $log_{2}^{n}$ < (n/m)$log_{2}^{n}$，所以 BTree 要比 LSM Tree 快。

### 2，读多个文件
BTree 只需要读取一个文件，而 LSM Tree 要读取多个文件。读的文件越多，速度越慢。

### 3，Compaction 影响读性能
LSM Tree 因为有 Compaction，所以会消耗一些写性能，也会影响一些读性能。其实 Compaction 目的之一就是加快读性能，但对磁盘的读写也会消耗一些读性能，可以说是以小换大吧。

<br>

# Further Reading
- There is a nice introductory post [here](https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/).
- The LSM description in this [paper](http://www.eecs.harvard.edu/~margo/cs165/papers/gp-lsm.pdf) is great and it also discusses interesting extensions.
- These three posts provide a holistic coverage of the algorithm: [here](http://leveldb.googlecode.com/svn/trunk/doc/impl.html), [here](https://www.datastax.com/dev/blog/leveled-compaction-in-apache-cassandra) and [here](http://www.quora.com/How-does-the-Log-Structured-Merge-Tree-work).
- The original Log Structured Merge Tree paper [here](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.44.2782&rep=rep1&type=pdf). It is a little hard to follow in my opinion.
- The Big Table paper [here](http://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf) is excellent.
- [LSM vs Fractal](http://highscalability.com/blog/2014/8/6/tokutek-white-paper-a-comparison-of-log-structured-merge-lsm.html) Trees on High Scalability.
- Recent work on [Diff-Index](https://researcher.watson.ibm.com/researcher/files/us-wtan/DiffIndex-EDBT14-CR.pdf) which builds on the LSM concept.
- [Jay](http://blog.empathybox.com/post/24415262152/ssds-and-distributed-data-systems) on SSDs and the benefits of LSM
- Interesting [discussion](https://news.ycombinator.com/item?id=9145197) on hackernews regarding index structures.

# Footnote on log structured file systems
Other than the name, and a focus on write throughput, there isn’t that much relation between LSM and log structured file systems as far as I can see.

Regular filesystems used today tend to be ‘Journaling’, for example ext3, ext4, HFS etc are tree-based approaches. A fixed height tree of inodes represent the directory structure and a journal is used to protect against failure conditions. In these implementations the journal is logical, meaning it only internal metadata will be journaled. This is for performance reasons.

Log structured file systems are widely used on flash media as they have less write amplification. They are getting more press too as file caching starts to dominate read workloads in more general situations and write performance is becoming more critical.

In log structured file systems data is written only once, directly to a journal which is represented as a chronologically advancing buffer. The buffer is garbage collected periodically to remove redundant writes. Like LSM’s the log structured file system will write faster, but read slower than its dual-writing, tree based counterpart. Again this is acceptable where there is lots of RAM available to feed the file cache or the media doesn’t deal well with update in place, as is the case with flash.


# 参考
- [Log Structured Merge Trees](http://www.benstopford.com/2015/02/14/log-structured-merge-trees/)：原文
- [Log Structured Merge Trees译文以及LSM调研心得](http://weakyon.com/2015/04/08/Log-Structured-Merge-Trees.html)：中文翻译
- [Log Structured Merge Trees(LSM) 原理](http://www.open-open.com/lib/view/open1424916275249.html)：这个也是一个中文翻译，翻译完才看到，没细看。

LevelDB 解读：
- [levelDB 源码解读](http://gao-xiao-long.github.io/page2/)：中间带一些 linux 讲解，不多。

<br>

# 翻译中的笔记
================== 笔记 =================

重点：
- 顺序写方式，不需要读数据。结构写的方式，需要先读取，再写入，这样进行了两次 IO。
- 使用`日志 + 日志 index( index 保存在内存中)`方式，就会减少一次磁盘 IO。
- 结构写的方式，增加了随机写。
- 虽然  4 种方式造成了`写速度`慢，但很多 DB 还是把文件保存了这种形式。因为他们不是在`写数据`的时候就把文件保存这种形式，而是写时候是顺序写。在一定时间后，再把数据保存以这种形式保存成文件，这样还是顺序写。所以以上 4 种形式还是有价值的，只不过是我们如何保存成这种形式。
- 使用了归并排序，在文件 compaction 时候。
- LevelDB 的 compaction 是基于层的，这种方式和基于数据数量的方式相比，减少文件数量，增加读性能。



