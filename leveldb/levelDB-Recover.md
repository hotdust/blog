# 前言
在 Recover 部分，主要是进行两部分的恢复：
- 从 Manifest 文件中恢复 Version
- 从`日志`中恢复 key/value

# Version 的 Recover 流程
## 正常处理
数据库每次启动时，都会有一个 Recover 的过程，简要地来说，就是利用 Manifest 信息重新构建一个最新的 version。
![version_recove](media/version_recover.jpg)


过程如下：
- 利用Current文件读取最近使用的manifest文件；
- 创建一个空的version，并利用manifest文件中的session record依次作apply操作，还原出一个最新的version，注意manifest的第一条session record是一个version的快照，后续的session record记录的都是增量的变化；
- 将非current文件指向的其他过期的manifest文件删除；
- 将新建的version作为当前数据库的version；

注意，随着leveldb运行时间的增长，一个manifest中包含的session record会越来越多，故leveldb在每次启动时都会重新创建一个manifest文件，并将第一条session record中记录当前version的快照状态。其他过期的manifest文件会在下次启动的recover流程中进行删除。

leveldb通过这种方式，来控制manifest文件的大小，但是数据库本身没有重启，manifest还是会一直增长。


## 异常处理
为什么MANIFEST损坏或者丢失之后，依然可以恢复出来？LevelDB如何做到。对于LevelDB而言，修复过程如下：
- 首先处理log，这些还未来得及写入的记录，写入新的.sst文件
- 扫描所有的sst文件，生成元数据信息：包括number filesize， 最小key，最大key
- 根据这些元数据信息，将生成新的MANIFEST文件。

第三步如何生成新的MANIFEST？ 因为sstable文件是分level的，但是很不幸，我们无法从名字上判断出来文件属于哪个level。第三步处理的原则是，既然我分不出来，我就认为所有的sstale文件都属于level 0，因为level 0是允许重叠的，因此并没有违法基本的准则。

当修复之后，第一次Open LevelDB的时候，很明显level 0 的文件可能远远超过4个文件，因此会Compaction。 又因为所有的文件都在Level 0 这次Compaction无疑是非常沉重的。它会扫描所有的文件，归并排序，产生出level 1文件，进而产生出其他level的文件。

从上面的处理流程看，如果只有MANIFEST文件丢失，其他文件没有损坏，LevelDB是不会丢失数据的，原因是，LevelDB既然已经无法将所有的数据分到不同的Level，但是数据毕竟没有丢，根据文件的number，完全可以判断出文件的新旧，从而确定不同sstable文件中的重复数据，which是最新的。经过一次比较耗时的归并排序，就可以生成最新的levelDB。

上述的方法，从功能的角度看，是正确的，但是效率上不敢恭维。Riak曾经测试过78000个sstable 文件，490G的数据，大家都位于Level 0，归并排序需要花费6 weeks，6周啊，这个耗时让人发疯的。

Riak 1.3 版本做了优化，改变了目录结构，对于google 最初版本的LevelDB，所有的文件都在一个目录下，但是Riak 1.3版本引入了子目录， 将不同level的sst 文件放入不同的子目录：
```
sst_0
sst_1
...
sst_6
```
有了这个，重新生成MANIFEST自然就很简单了，同样的78000 sstable文件，Repair过程耗时是分钟级别的。

# 日志的 Recover
日志处理是，在保存成 sstable 后就删除掉。但在恢复`日志`时，存在的日志不一定是没有保存成 sstable 的日志。有可能是已经保存成了 sstable，但在删除时挂了，没有删除掉。所以，需要先恢复 Manifest 文件，因为 Manifest 文件中，记录了 sstable 对应的`日志`文件，找到最新的`日志`文件后，从最新的`日志`后的日志开始恢复就可以了。
> VersionSet 中 log_number_ 是保存当前 Version 中最新的`日志`文件。

在找到哪些`日志`需要被恢复后，从`日志`文件的生成次序开始恢复。把`日志`中的内容插入到 memtable 里，再到 level 0 这个流程。在这个流程中，要记录一下最大的`sequence number`。

最后再把最大的`sequence number`设置到系统上，在插入时使用。


# 参考：
- [leveldb:数据库recover机制 - weixin_36145588的博客 - CSDN博客](https://blog.csdn.net/weixin_36145588/article/details/78029415)


# todo
- sstable 损坏异常处理说明