# 内存数据库
在介绍完跳表这种数据结构的组织原理以后，我们介绍leveldb如何利用跳表来构建一个高效的内存数据库。

键值编码
在介绍内存数据库之前，首先介绍一下内存数据库的键值编码规则。由于内存数据库本质是一个kv集合，且所有的数据项都是依据key值排序的，因此键值的编码规则尤为关键。

内存数据库中，key称为internalKey，其由三部分组成：

用户定义的key：这个key值也就是原生的key值；
序列号：leveldb中，每一次写操作都有一个sequence number，标志着写入操作的先后顺序。由于在leveldb中，可能会有多条相同key的数据项同时存储在数据库中，因此需要有一个序列号来标识这些数据项的新旧情况。序列号最大的数据项为最新值；
类型：标志本条数据项的类型，为更新还是删除；

键值比较
内存数据库中所有的数据项都是按照键值比较规则进行排序的。这个比较规则可以由用户自己定制，也可以使用系统默认的。在这里介绍一下系统默认的比较规则。

默认的比较规则：

首先按照字典序比较用户定义的key（ukey），若用户定义key值大，整个internalKey就大；
若用户定义的key相同，则序列号大的internalKey值就小；
通过这样的比较规则，则所有的数据项首先按照用户key进行升序排列；当用户key一致时，按照序列号进行降序排列，这样可以保证首先读到序列号大的数据项。

数据组织
以goleveldb为示例，内存数据库的定义如下：

type DB struct {
    cmp comparer.BasicComparer
    rnd *rand.Rand

    mu     sync.RWMutex
    kvData []byte
    // Node data:
    // [0]         : KV offset
    // [1]         : Key length
    // [2]         : Value length
    // [3]         : Height
    // [3..height] : Next nodes
    nodeData  []int
    prevNode  [tMaxHeight]int
    maxHeight int
    n         int
    kvSize    int
}
其中kvData用来存储每一条数据项的key-value数据，nodeData用来存储每个跳表节点的链接信息。

nodeData中，每个跳表节点占用一段连续的存储空间，每一个字节分别用来存储特定的跳表节点信息。

第一个字节用来存储本节点key-value数据在kvData中对应的偏移量；
第二个字节用来存储本节点key值长度；
第三个字节用来存储本节点value值长度；
第四个字节用来存储本节点的层高；
第五个字节开始，用来存储每一层对应的下一个节点的索引值；
基本操作
Put、Get、Delete、Iterator等操作均依赖于底层的跳表的基本操作实现，不再赘述。


# 个人总结
## 1，为什么使用 skiplist 做为数据结构
因为 skiplist 的插入和删除操作比 balanced tree 快很多，这也是利于`大量写操作`的。
todo 把时间复杂度什么的补全。
