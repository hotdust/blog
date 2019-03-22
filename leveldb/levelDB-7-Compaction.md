# 为什么需要 Compaction
为什么需要 Compaction，原因有两个：
- 提高查询速度：打开的文件越多，查询的速度越慢。而且，level 0 层查找时，是每个文件都需要查找，所以通过减少文件来加快 level 0 层的查询速度。
- 删除重叠数据：Insert A 和 Delete A 是两条数据，并且可能不在同一个文件中。通过 Compaction 把他们变成 1 条数据，减少磁盘空间。


# Compaction 分类
Compaction 主要分为两种：
- Minor Compaction：把 Immutable MemTable 变成 level 0 的 sstable。
- Major compaction：合并 level 0 ~ 6 之间 sstable。


# Minor Compaction
一次 Minor compaction 非常简单，其本质就是将一个内存数据库中的所有数据持久化到一个磁盘文件中。

## Minor Compaction 触发条件
触发条件这里有点复杂，触发条件分为两种场景：
- 恢复时触发
- 运行中触发

### 恢复时触发
恢复时触发的场景，只会在`DB 打开`时进行一次。因为在`打开 DB`时，会进行 log 恢复，如果有 log 中的数据没有写到 sstable 里的话，就会恢复这些数据到 memtable 或 sstable。

如果在 log 恢复到 memtable 时，超过了 memtable 的大小限制，就会把 memtable 直接写成 memstable。而不会先把引用设置到 immemtable 上，再把 immemtable 写到 sstable 上。

### 运行中触发
运行时触发条件如下：
- memtable 的使用空间超过 4M（这个触发条件是在 Write 时候做的）
- immemtable 转成 sstable 过程已经结束。（如果没有结束，会一直等到结束，然后再 check 一词遍是否结束）

## Minor compaction 处理内容
Minor compaction 处理主要是在 CompactMemTable 方法中进行的。主要处理内容如下：
- 根据 immemtable 创建 sstable
- 根据最小 key 和 最大 key，计算 sstable 应该放在哪个 level 上
- 放入 VersionEdit 中
- 删除 memtable 对应的 log 文件

### 1，根据 immemtable 创建 sstable
主要就是把 key/value 写到 sstable 上，然后再把 index_block、filter_block 等信息再写上。把`最大 key`和`最小 key`还有`文件大小`等信息写到 metaData 上，并返回给后面流程用。

### 2，根据最小 key 和 最大 key，计算 sstable 应该放在哪个 level 上
这块是处理中的重点，因为这些逻辑比较复杂。主要逻辑如下：

- 如果 level 0 层有 key 重叠部分，就直接把 sstable 放到 level 0 层。
- 如果 level 0 层没有 key 重叠部分，从 level 0 开始，到 level 2 做下循环判断。如果下面条件都不符合，就返回 level 2 作为要 sstable 要放的地方。
  * 如果`level 当前 + 1`有 key 重叠部分，sstable 就放到`level 当前`里。
  * 如果`level 当前 + 2`有 key 重叠 并且 `重叠文件大小超过指定值(20M)`，就放到`level 当前`

上面的循环判断的要点是：
> 如果条件满足，就返回`level 当前`。如果不满足，就在`level 当前+1`看看，是否可以满足，如此循环。
 
下面是上面流程的过程图。引用自：[leveldb之Compaction (1) --从MemTable到SSTable文件](http://bean-li.github.io/leveldb-compaction/)
![PickLevelForMemTableOutput](media/PickLevelForMemTableOutput.png)

#### 复杂的原因
为什么比较复杂呢？下面是关于 kMaxMemCompactLevel 常量上的一段注释：
```
// Maximum level to which a new compacted memtable is pushed if it
// does not create overlap.  We try to push to level 2 to avoid the
// relatively expensive level 0=>1 compactions and to avoid some
// expensive manifest file operations.  We do not push all the way to
// the largest level since that can generate a lot of wasted disk
// space if the same key space is being repeatedly overwritten.
static const int kMaxMemCompactLevel = 2;
```


##### (1) 为什么`level 较小的`有 key 重叠就放到`level 较小的`的上面呢？（例如，如果 level 0 上有重叠，就放到 level 0 上）
~~对于 level 0，因为 level 0 是可以有重叠的 sstable 的，所以要顺序查找每个 sstable。这样放在 level 0 可以更快的查找到重叠的 key。~~
~~既然 level 0 上有重叠的 key，那么在 minor compaction 时可以合并 key 范围重叠的文件，避免空间浪费。~~

如果 level 0 上有重叠，而不放在 level 0 上，则可能发生`取得旧 key`问题。例如：在 level 0 上有一个 sstable 文件中，有一个 key=aaa 的数据。如果新的 sstable 也有一个 key=aaa 的数据，却放在了 level 1 或 2 上，这样在查找 key=aaa 的数据时，就会从 level 0 的 sstable 上取得旧的数据。

对于 level 1,2，因为 level 0 以外的 level 上，不允许有 key 重叠的 sstable，所以如果在 level 1,2 有重叠的话，就要放到 level 0上。

##### (2) 为什么 level 0,1,2 都没有重叠，要放到 level 2 呢？
因为在 level 0 上进行 sstable 查找时，需要查找每个 sstable，不能进行二分查找，所以尽量不放在 level 0 上。

不放在 level 0 上，为什么尽量也不放在 level 1 上呢？level 越小，也就是说`文件数越少`/`compaction 容量越小`，compaction 机率越大。上面说了，把 sstable 放到 level 2，可以尽量避免`相对比较昂贵的 compaction`和`昂贵的 mainfest 操作`。


##### (3) 为什么不放到`最大的 level(leve 6)`上呢？
不放到最大 level 的原因是，避免空间浪费。因为 key 和 value 是可以重叠的，如果放在最大 level，很久不进行 compaction 的话，重叠的 key 和 value 会一直存在，不会被回收。

而且，新 key 的访问率一般会很高，放在最大 level 的话，可能需要查找很多 sstable 才能找到。



# Major compaction
Major compaction 的主要作用是把 level 0~6 层的 sstable 进行合并操作。如果不合并的话，文件个数多，并且重叠的 key/value 也很多，影响查询速度和使文件过于庞大。

## Major compaction 触发条件
- level 0 文件总个数（4个）
- level 1~6 文件总大小超过阈值
- 文件 Seek（查找）次数超过阈值（文件长度／16KB）

下面说说每个条件。

### 触发条件1: level 0 文件总个数（4个）
为什么 level 0 的触发条件是文件个数，而其它 level 而是以总文件大小呢？
在 VersionSet::Finalize 方法里，有一段注释，说明了原因。在自己理解的基础上，翻译了一下。
原文：
```
// We treat level-0 specially by bounding the number of files
// instead of number of bytes for two reasons:
//
// (1) With larger write-buffer sizes, it is nice not to do too
// many level-0 compactions.
//
// (2) The files in level-0 are merged on every read and
// therefore we wish to avoid too many files when the individual
// file size is small (perhaps because of a small write-buffer
// setting, or very high compression ratios, or lots of
// overwrites/deletions).
```
自己理解：
- 如果使用`文件个数`的方式的话，write-buffer(memtable 的 buffer) 大小越大，compaction 次数就越少。
- 因为 level 0 上的 sstable 是可以发生重叠的，所以 read 操作需要遍历每个 sstable。所以为了保证读取速度，避免因为文件大小设置过小，造成文件过多。例如：如果不是以文件个数，而以总文件大小作为处发条件的话，在总文件大小不变的情况下，把 sstable 文件大小改的越小，文件就越多，read 操作就越慢。如果以文件个数的话，sstable 数量就不会过多（但也会产生 compaction 过多的问题，所以 sstable 不能太小）。

> memtable 的大小到了 4M 就会写成 sstable，这样看 level 0 的 sstable 大小应该是 4M。但 options 里的 max_file_size 定义是 2M，而且在 memtable 写成 sstable 时也没有大小限制。


### 触发条件2：level 1~6 文件总大小超过阈值
level 1~6 的文件总大小是不是超过阈值，是通过 VersionSet::Finalize 进行计算的（level 0 的文件个数也是在这里计算的）。在这个方法中，调用了一个 MaxBytesForLevel 方法，此方法中给出了每个 level 文件总大小的阈值。

level | 阈值
--- | ---
1 |      10M 
2 |     100M
3 |    1000M
4 |   10000M
5 |  100000M
6 | 1000000M

> todo 如果每个 sstable 大小为 2M，那 level 最多有 5 个 sstable？

### 触发条件3：文件 Seek（查找）次数超过阈值（文件长度／16KB）
为什么会因为 seek 不到数据次数过多，而进行 compaction 呢？下面的注释中进行了解释。
```
// We arrange to automatically compact this file after
// a certain number of seeks.  Let's assume:
//   (1) One seek costs 10ms
//   (2) Writing or reading 1MB costs 10ms (100MB/s)
//   (3) A compaction of 1MB does 25MB of IO:
//         1MB read from this level
//         10-12MB read from next level (boundaries may be misaligned)
//         10-12MB written to next level
// This implies that 25 seeks cost the same as the compaction
// of 1MB of data.  I.e., one seek costs approximately the
// same as the compaction of 40KB of data.  We are a little
// conservative and allow approximately one seek for every 16KB
// of data before triggering a compaction.
f->allowed_seeks = (f->file_size / 16384);
if (f->allowed_seeks < 100) f->allowed_seeks = 100;
```

大概意思是说，如果一个 sstable seek 不到 N 次的话，那么这 N 所花费的时间 和 把这个 sstable 进行 compaction 所花费的时间差不多。为了不在这个 sstable 上继续浪费时间，就把它 compaction，这样就会减少文件打开。

什么场景下会产生这种查很多次查不到呢？可能会有两个场景：
- key 范围严重重叠。文章（[leveldb之Compaction (2)--何时需要Compaction](http://bean-li.github.io/leveldb-compaction-2/)）上说是原因是有严重的重叠。确实重叠才产生这种问题的一个原因。例如：level 1 sstable 的 key 小范围是 30~100，level 2 sstable 范围是 40~110，但 level 1 上只有 30 和 100 两个 key，而 level 2 上有 很多 key，这时候是范围重叠原因。但如果 level 1 有很多 key，而 level 2 上 key 很少，还发生了 level 1 上 seek miss，这时候可能是热点数据的原因。

~~- 热点数据：如果在一个 sstable 上进行查找的话，`要找的 key`肯定在此 sstable 的`最大 key`和`最小 key`之间。但经过查找还找不到数据，说明此 sstable 可能不包含`热点数据`(就是经常被使用的数据)。如果不包含热点数据，还经常读它的话，浪费文件打开时间和缓存等资源。~~
应该和热点数据没有太多关系，热点数据一般都会保存到 cache 里。

> seek 是怎么一个过程呢？seek 过程就是一个 sstable 的查找过程，先使用 filter_block 查找，有可能还对具体的 data_block 进行查找。

### 触发条件的设置和判断代码
触发条件 1 和 2 是在`VersionSet::Finalize`方法中设置的。触发条件 3 是在 `Version::UpdateStats`方法中进行设置的。

判断代码在`VersionSet::PickCompaction`方法中。而`PickCompaction`方法是在`MaybeScheduleCompaction`中一层层调用到的。


# Compaction 过程
## 关于 level 中的 sstable
剔除level 0不论，对于任何一个层级来说，层级的内的任意一个文件本身是有序的，而位于同一层级的内部的多个文件，他们也是有序的，而且key是不交叉的。但是很不幸的是，level n 和level n＋1的文件，key的范围可能交叉，这种交叉，就可能带来 seek miss，即数据有可能位于level n的某个文件中（根据该文件的最小key和最大key和用户要查找的key来推算），但是实际情况是并不在level n的该文件中，不得不去level n＋1的文件查找。这种seek miss不解决，就会造成查询效率的下降。而且每个 level 中的数据会有重叠的情况，即 key user 在 level n+1 里有，在 level n 里也有。

如何解决？

出现这种 seek miss 的原因是level n和level n ＋ 1，在某些key的范围内，是有交叉的。解决的方法即：根据 level n 的某个(些)文件，找到 level n＋1 中 key 范围有交叉的文件，然后合并 level n 和 level n+1 找出的文件，归入level n＋1，而删除 level n 的该部分文件，从而消除该key范围内，level n 和 level n+1 的重叠。而且进行 key 范围有交叉的文件进行合并，也可以消除 key 重叠的问题。

todo 那为什么LevelDB非要搞出来level 0 ～level 6 这么多的层级呢？就level 0和level 1不好吗？


## 如何选取哪些文件进行 Compaction
Compaction 的过程就是`选取 level n 和 level n+1 的 sstable`，然后进行合并。但 Compaction 根据触发条件不一样，在 level n 进行 Compaction 的文件也不一样。下面细说一下，主要分成以下部分进行说明：
- level n 层文件选取
- level n+1 层文件选取
- 进行合并

### level n 层文件选取
上面说了，在进行 level n 层文件选取时，根据触发条件不一样，选取的文件也不一样。

#### seek 触发
seek miss 触发时，选取比较简单。就选取达到 seek miss 阈值的那个文件，做为 level n 层进行 compaction 的文件。

#### 文件大小或个数触发
文件大小或个数触发的情况，要经过两个步骤选取文件：
- 需要考虑上一次`该 level 的 compaction`做到了哪个 key，然后`大于该key`的第一个文件即为 compaction 文件。如果第一次做，或者上次已经是最大的 key，那么回到第一个文件开始 compaction。
- 对于n >0的情况，初选情况下（即调用SetOtherInput之前）level n的参战文件只会有1个，如果n＝0，因为level 0的文件之间，key可能交叉重叠，因此，根据选定的level 0的该文件，得到该文件负责的最小key和最大key，找到所有和这个key 区间有交叠的level 0文件，都加入到参战文件。

简单概括，即当n>0的时候，初选 level n的参战文件只会有1个，而n ＝ 0的时候，初选的情看下，level n的文件也可能有多个。

### level n+1 层文件选取
确定了 level n 的文件，根据 level n 文件，就可以确定出 level n 文件的`最小 key`和`最大key`，有了`最小 key`和`最大 key`，就可以进一步确定 level n＋1 需要的文件。

选择level n＋1的步骤如下：
1. 根据level n的文件，计算得到`最小 key`和`最大 key`
2. 根据`最小 key`和`最大 key`圈定的 key 的范围，从level n＋1中选择和该范围有交叠的所有文件，计入c->inputs_[1]，作为 level n＋1 的文件
3. 根据第一步和第二步的到的所有的 level n 的文件和 level n＋1 的文件，计算`选取的` level n 和 level n+1 文件中`最小 key`和`最大 key`
4. 根据`选取的` level n 和 level n+1 文件中`最小 key`和`最大 key`，来看看 level n 中是否还有文件可加入 compaction。如果有就加入 level n 选取的文件中。

1 和 2 好理解，3 和 4 又是为了做什么呢？为什么还在重新看一下 level n 中有没有进行可以 compaction 的文件呢？我们看一下下面的说明：

看下图，根据 1 和 2 的处理，level n 中的文件 A 计算出了 level n＋1 中的 B C D 需要一起 compaction。但是由于B C D的加入，key的范围扩大了，又一个问题是 level n 层的 E 需不需要一起 compaction？从上图看，还是一起比较好的，因为 A＋E 确定的范围，完全笼罩在计算出来的B C D的范围之内，不会因为E的加入，而扩大level n＋1的文件。
![calc_level_n_1](media/calc_level_n_1.png)


我们再举一个例子，根据下图所示的情况，很明显E是不能加入战局的原因是 leven n＋1 层的B C D无法笼罩 A＋E 确定的范围，如果不管不顾，（A＋E）和（B＋C＋D） 一起Compaction造成的恶果就是，很可能和level n＋1 的某个已有文件(F)发生重叠，这就破坏了 level 1～level 6 同一层的文件之间不许重叠的约定（合并后的文件和 F 发生 key 小范围重叠）。因此，下图的情况，是不允许 level n 的 E 参与的。
![level_n_1_not](media/level_n_1_not.png)

说笼罩，其实也不确切了，因为 level n 新加入了文件，很大key 可能造成key的范围扩大，只要扩大后的key的范围，不会 involve 新的 level n＋1 的文件就行。如下图所示，虽然已经超出了 level n+1 文件圈定的范围，但是并没有和 level n＋1 其他的文件重叠交叉，这样也是可以的。
![level_n_ok](media/level_n_ok.png)

如果满足上述条件是不是 level n的文件 E 是不是一定可以 compaction 呢？ 也不一定，看参战文件是否已经超出了上限。注意，单次 compaction 的文件，我们不希望太多，造成瞬时的读写压力。因此，到底扩不扩容，看扩容之后参战文件的总大小。如果加入了level n的扩容文件，leven n和level n＋1的文件总长度超过了上限，那么就放弃扩容 level n中的文件。

上限是多少？25倍的TargetFileSize，以典型的2M大小为例，如果level n和level n＋1的总大小超过了50MB，就不再主动扩容，将level n的某些符合条件的文件involve 进来了。

最后的最后是记录下本次 compaction 文件的最大 key，对于size compaction, 下一次要靠该值来选择 level n 的文件。


## Compaction 具体执行过程
todo 这部分内容还不少，等以后再介绍。

todo 在合并 sstable 时，哪些 key 可以删除掉是有条件的，并不是有 Deletion 标志的 Key 都可以删除掉。
- 对于一个 Deletion 状态的 Key 来说，如果当前 key 所在 level 的上层 level 中，没有 key 范围重叠的 sstable 的话，才会删除这个 key。如果有重叠，并且删除掉的话，原来已经是删除状态的 key，可能又复活了，又可以查到 value 了。



# 细节
## 1，当 level 0 文件数过多时
level 0 的 Major compaction 条件是文件数超过 4 个。但因为 write 操作量非常大，造成 level 0 文件超过 4 个的情况也是有的。当遇到这种情况，levelDB 是这么处理的：
- 在 write 操作时，如果发现 level 0 文件数超过 8 个，每次写操作前都会 sleep 1ms。这么做是为了能把一部分 CPU 给 compaction 操作。
- 在 write 操作时，如果发现 level 0 文件数超过 12 个，暂停 write 操作，等待 compaction 完成（minor 或 major）。

**为什么对 level 0 的文件数进行确认呢？**
因为如果 level 0 上文件越多的话，read 越慢。因为 level 0 上的 sstable 里的 key 是有可能重叠的，所以在查找时需要一个个 sstable 查找。其它 level 的 sstable 都是不重叠的，所以在查找 key 时，可以使用二分法查找哪个 sstable 上，不用一个个查找。


## 2，Compaction 的优先级
在 compaction 有 minor 和 major 两种，而 major 里触发条件还有`文件大小`或`Seek miss`等。他们之间也是有优先级关系的，优先级关系如下：
> Minor > Manual > Size > Seek

- minor 是优先级最高的。例如，在 BackgroundCompaction 方法，会有如果 immemtable 不为空，就进行 minor compaction 的判断。在 major compaction 进行中(DoCompactionWork方法中)时，还有类似的判断。
- Manual 种类的 compaction 没有在文章中介绍。
- Size 和 Seek 优先级，在 VersionSet::PickCompaction 方法中可以看到。

为什么要把 minor 最优先呢？因为如果 immemtable 不可用的话，是无法进行 write 操作的。所以需要尽快把 immemtable 写成 sstable。

## 3，每个 level 的 sstable 大小
- level 0：4M
- leve 1~6：2M

level 0 的 sstable 是根据 memtable 大小限制生成的。
level 1~6 是在 compaction 时，看是否超过指定大小来生成的。

## 4，还什么时候会启动 compaction
以下的 db_impl.cc 里的操作都会启动 compaction。但应该只有 Write 会启动 minor compaction。

Operation | file-number | explanation
---|---|---
Get | db/db_impl.cc-1147 | after read an Key-Value pair
Write | db/db_impl.cc-1374 | after created a new MemTable
RecordReadSample | db/db_impl.cc-1170 | after walked througth several KeyValue pairs
BackgroundCall | db/db_impl.cc-681 | after the background thread has complete the previous compaction task.
Open | db/db_impl.cc-1521 | when open and recover LevelDB.

## 5，Compaction 启动次数
总的说来，Compaction 启动次数还是少一些为好。对于 100M 的数据，`进行 10 次，每次 10M`的 Compaction 所花的时间，很可能比`进行 1 次，1 次 100M`的 Compaction 所花的时间要多。



# 参考：
- [leveldb之Compaction (1) --从MemTable到SSTable文件](http://bean-li.github.io/leveldb-compaction/)：本文绝大部分的内容都是参考这里，讲的非常好。这个部分的文章是连续的，还有下面的 2 篇。
- [leveldb之Compaction (2)--何时需要Compaction](http://bean-li.github.io/leveldb-compaction-2/)
- [leveldb之Compaction（3）－－选择参战文件](http://bean-li.github.io/leveldb-compaction-3/)
- [leveldb源码分析--SSTable之Compaction - tgates - 博客园](https://www.cnblogs.com/KevinT/p/3819134.html)：关于源码的解读。因为每篇文章都是针对某一部分讲的很好，所以每篇文章都有一些可看之处。
- [leveldb源码剖析----compaction - Swartz2015的专栏 - CSDN博客](https://blog.csdn.net/Swartz2015/article/details/67633724)：此文章有关于具体的 compaciton 执行过程的源码讲解。
- [leveldb之Compaction操作下之具体实现 - qinm的专栏 - CSDN博客](https://blog.csdn.net/u012658346/article/details/45788939)：讲了一部分 version 操作，需要时候可以看看。
- [LevelDB-Compaction](http://openinx.github.io/2014/08/17/leveldb-compaction/)：讲了一些 compaction 的想法。还整理了一些 compaction 时的条件和约束，挺清晰的。
- [SSTable之Compaction上篇-leveldb源码剖析(9) | PandaDemo](http://www.pandademo.com/2016/04/compaction-of-sstable-leveldb-part-1-source-dissect-9/)：根据对代码的讲解。如果不知道某一块代码意思时，可以去看看。还讲到了 version 操作。
- [Leveldb_RTFSC | GRAKRA 三十年众生牛马 六十年诸佛龙象](http://www.grakra.com/2017/06/17/Leveldb-RTFSC/)：国人用英文写的，里面有 major compaction 之前和之后的例子。



# Todo
## 1，看一下具体的 compaction 操作，里面有许多和 version 相关的操作。


## 2，CompactMemTable 中，保存成 sstable 和 删除 log 的操作，如何做成同步操作。就是不会在创建完 sstable 后，因为停电，造成 log 没有补删除。


# 源码理解记录
## 1，Minor Compaction 的触发条件
### 运行时触发
- memtable 的使用空间是否超过 4M
- level 0 的 sstable 数量超过 4 个

第一个条件是在 MakeRoomForWrite 方法中的以下 if 条件中判断的，条件的意思是：如果 memtable 大小没有超过 4M，就返回（不做压缩）。（这个方法只会在 Write 方法中被调用）
```
} else if (!force &&
           (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {
  // There is room in current memtable
  break;
} 
```
`mem_->ApproximateMemoryUsage()`是用来返回当前 memtable 的大小，`options_.write_buffer_size`是用来预先定义好的 memtable 的大小阈值。

`options_.write_buffer_size`是在 options.cc 文件中初始化的`write_buffer_size(4<<20) = 4M`。

第二个条件是在 MaybeScheduleCompaction 方法中，有下面这个条件。
```
} else if (imm_ == nullptr &&
         manual_compaction_ == nullptr &&
         !versions_->NeedsCompaction()) {
// No work to be done
```

其中 NeedsCompaction 返回的 score 和 level 值。这两个值是在 VersionSet::Finalize 方法里计算的，过程如下：
- 如果 level 0 层的话，score = level 0 层 sstable 文件个数 / 4
- 如果 level 1~6 层的话，score = level i 层 sstable 文件总大小 / i 层 compaction 阈值大小

MaybeScheduleCompaction 方法会被下面的方法调用
- Get
- Write
- BackgroundCall
- Open


## 关于 NeedsCompaction
14，MaybeScheduleCompaction 中的`versions_->NeedsCompaction`条件应该只是为 Major compaction 准备的，因为 minor compaction 条件中，immemtable 必须不为空。
```
  } else if (imm_ == nullptr &&
             manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
```


