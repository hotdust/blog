# 前言
其本质是一种内部数据重合整合的机制，同样也是一种平衡读写速率的有效手段。
Immutable MemTable -> immtable

# Compaction 分类
Compaction 主要分为两种：
- Minor Compaction：把 Immutable MemTable 变成 level 0 的 sstable。
- Major compaction：合并 level 0 ~ 6 之间 sstable。


# Minor Compaction
一次 Minor compaction 非常简单，其本质就是将一个内存数据库中的所有数据持久化到一个磁盘文件中。

## 触发条件
触发条件这里有点复杂，触发条件分为两种场景：
- 恢复时触发
- 运行中触发

### 恢复时触发
恢复时触发的场景，只会在`DB 打开`时进行一次。因为在`打开 DB`时，会进行 log 恢复，如果有 log 中的数据没有写到 sstable 里的话，就会恢复这些数据到 memtable 或 sstable。

如果在 log 恢复到 memtable 时，超过了 memtable 的大小限制，就会把 memtable 直接写成 memstable。而不会先把引用设置到 immemtable 上，再把 immemtable 写到 sstable 上。

### 运行中触发
运行时触发分为两种情况：
- memtable 的使用空间是否超过 4M
- level 0 的 sstable 数量超过 4 个

触发步骤发为两步，走这两步才可能触发：
1. 在 Write 方法中，判断 memtable 的使用空间是否超过 4M。如果超过，会把 memtable 的引用设置到 immemtable 上。
2. 调用 MaybeScheduleCompaction 方法。此方法中会判断 immemtable 是否为 null，如果不为 null 可能会执行 minor compaction。以下是调用 MaybeScheduleCompaction 方法的地方。
  * Get
  * Write
  * BackgroundCall
  * Open

触发步骤发为两步，走这两步才可能触发：
1. 在 Write 方法中，判断 memtable 的使用空间是否超过 4M。如果超过，会把 memtable 的引用设置到 immemtable 上。
2. 调用 MaybeScheduleCompaction 方法。此方法中会判断 immemtable 是否为 null，如果不为 null 可能会执行 minor compaction。以下是调用 MaybeScheduleCompaction 方法的地方。
  * Get
  * Write
  * BackgroundCall
  * Open

在上面的两个条件同时满足的情况下，会阻塞写线程，把memtable移到immtable。然后新起一个memtable，让写操作写到这个memtable里。最后将imm放到后台线程去做compaction.


todo 在 level0 sstable num > 4 时，也进行 minor.

# Major compaction
MaybeScheduleCompaction 中的`versions_->NeedsCompaction`条件应该只是为 Major compaction 准备的，因为 minor compaction 条件中，immemtable 必须不为空。
```
  } else if (imm_ == nullptr &&
             manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
```

# 源码解读
## 1，Minor Compaction 的触发条件
### 运行时触发
- memtable 的使用空间是否超过 4M
- level 0 的 sstable 数量超过 4 个

第一个条件是在 MakeRoomForWrite 方法中的以下 if 条件中判断的，条件的意思是：如果 memtable 大小没有超过 4M，就返回（不做压缩）。
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




# 问题
1，为什么？如果查找了多次，某个文件不得不查找，却总也找不到，总是去高一级的level，才能找到。这说明该层级的文件和上一级的文件，key的范围重叠的很严重？

2，level 一共几层？

3，说一下下面参数的作用。
static const int kNumLevels = 7;

// Level-0 compaction is started when we hit this many files.
static const int kL0_CompactionTrigger = 4;

// Soft limit on number of level-0 files.  We slow down writes at this point.
static const int kL0_SlowdownWritesTrigger = 8;

// Maximum number of level-0 files.  We stop writes at this point.
static const int kL0_StopWritesTrigger = 12;

// Maximum level to which a new compacted memtable is pushed if it
// does not create overlap.  We try to push to level 2 to avoid the
// relatively expensive level 0=>1 compactions and to avoid some
// expensive manifest file operations.  We do not push all the way to
// the largest level since that can generate a lot of wasted disk
// space if the same key space is being repeatedly overwritten.
static const int kMaxMemCompactLevel = 2;

// Approximate gap in bytes between samples of data read during iteration.
static const int kReadBytesPeriod = 1048576;


===================
下面这段的意义
      // We are getting close to hitting a hard limit on the number of
      // L0 files.  Rather than delaying a single write by several
      // seconds when we hit the hard limit, start delaying each
      // individual write by 1ms to reduce latency variance.  Also,
      // this delay hands over some CPU to the compaction thread in
      // case it is sharing the same core as the writer.
      mutex_.Unlock();

4，看一下 doc/impl.html 中的说明。

5，看一下 void VersionSet::Finalize(Version* v) 里的那段英文说明。