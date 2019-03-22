https://lishoubo.github.io/2017/09/27/MappedByteBuffer%E7%9A%84%E4%B8%80%E7%82%B9%E4%BC%98%E5%8C%96/
https://www.bbsmax.com/A/pRdBW6p9zn/
https://my.oschina.net/feichexia/blog/212318



#基础知识
##一，block、page、page cache 和 buffer cache 

- block：是操作系统用来“读/写”文件的最小管理单元。就像我们在给一个磁盘建立文件系统时候，我们可以指定`-block_size`。
- page：是针对内存管理，例如从磁盘读出的数据就缓存在内存页中
- page cache：专门用来保存“磁盘文件缓存”的页面叫“缓冲区页（page cache）”，Linux 完成所有的“文件IO”都是通过 page cache。page cache 的大小等于：Cached + SwapCached（Cached 和SwapCached 参考 /proc/meminfo）。page cache 包含 buffer cache (在 page cache 里面被称为 buffer head)，每个 buffer cache 指向一个 block。
- buffer cache：原来是一个单独的缓存，在`内核2.4`版本后，就和 page cache 合并了。page cache 里面是多个 buffer head（可以看成是 buffer cache），每个 buffer head 有指向 block 的 buffer 的指针（b_data），也有指向实际具体的 block 指针（b_bdev）。

![这里写图片描述](https://img-blog.csdn.net/20180508123135191?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

在 /proc/meminfo中存储了当前系统的内存使用情况，比如下面这个例子, 
 - Buffers表示buffer cache的容量
 - Cached表示位于物理内存中的页缓存page cache
 - SwapCached表示位于磁盘交换区的页缓存page cache

所以实际页缓存page cache的容量 =  Cached + SwapCached
（有的时候为了防止 swap，禁止了 swap 功能，这样就 swap 相关的都是 0 了）

参考：
[linux内核tmpfs/shmem浅析](https://blog.csdn.net/ctthuangcheng/article/details/8916065)：讲了内存共享和 page、page cache 、swap 的关系。
[漫谈linux文件IO](http://elf8848.iteye.com/blog/1944226)：讲了写数据所经历的从内核到磁盘的整个过程，还讲了 raw io 和 direct io 的区别。挺不错的。
[What is the difference between Buffers and Cached columns in /proc/meminfo output?](https://www.quora.com/What-is-the-difference-between-Buffers-and-Cached-columns-in-proc-meminfo-output)

学习：
[Page Cache, the Affair Between Memory and Files](https://manybutfinite.com/post/page-cache-the-affair-between-memory-and-files/)
[What is some existing documentation on Linux memory management?](https://landley.net/writing/memory-faq.txt)：总结了"What every programmer should know about memory"中的一些概念，都很精简，不错。

下面进行具体介绍一下。


###1，关于 page
page 从两个维度来看：

- PRIVATE/SHARED：用来表示某块内存是否可以被其它 process 看到。
- ANONYMOUS/FILE-BACKED：用来表示这块内存是不是根文件相关的。ANONYMOUS 代表不是和文件相关的，就像我们平时声明的变量。FILE-BACKED 代表这块内存是和文件相关的，内容是从文件中加载的。

| - | PRIVATE | SHARED | 
|:-------|------|------|
| ANONYMOUS | stack<br> malloc()<br> mmap(ANON, PRIVATE)<br> brk()/sbrk() | mmap(ANON, SHARED) |
| FILE-BACKED | mmap(fd, PRIVATE)<br> binary/shared libraries | mmap(fd, SHARED) |

**PRIVATE：**
意思是只属于某个 process 的内存，对其它 process 不可见。大部分的内存都是 private 的。
既然 private 类型的内存的修改对其他 process 不可见，服从于 copy-on-write 机制。另一方面，虽然内存是 private 的，但多个 process 可能会共享相同的 physical memory。大家对 KDE 的错误认识就是，使用了很我 RAM，因为每个 process 都要加载 Qt 和 KDElibs。但是，由于 copy-on-write 机制，所有的 process 对于那些 lib 的“只读”部分，都使用“完全相同的” physical memory。

file-backed private memory 的修改不会被写回到文件。however changes made to the file may or may not be made available to the process.

**SHARED：**
Shared Memory 是被设计用来进行 process 间交互的。可以使用 mmap() 和 shm* 这样的系统调用来创建。当一个 process 修改了 Shared Memory，这些修改对其它 process 是可见的。

对于 file-backed 类型的 Shared Memory 来说，修改对其它 process 也是可见的。

**Anonymous：**
Anonymous memory 是和文件无关的，就像是我们平时声明的变量等（和文件有关的内存是下面的 File-backed 类型的内存）。
Anonymous memory 在被写入之前，kernel 不会这块内存映射到 physical address。也就是说 Anonymous memory 在使用之前，不会对 kernel 增加任何压力。这样使 process 可以保留一块内存在它的 vma 中，在真正使用它之前。

**File-backed and Swap：**
当一块 memory map 是 file-backed 类型的话，意味着数据是从磁盘上加载的。很多时间根据需要进行加载，但也可以预先加载。

当 physical memory 紧张时，kernel 会试着把一部分数据从 RAM 移动回磁盘。如果是 file-backed and shared memory 的话，把这块内存直接清空就可以了，因为数据都是从磁盘上来的。如果是 anonymous/private memory 的话，kerenl 会把这部分内存中的数据写到磁盘上某个地方，被称作“swapped out”。


###2，内存类型和 cached、buffers、AnonPages 的关系
在 /proc/meminfo 中有很多字段，我们先看以下 3 个字段：

- Cached
- SwapCached
- Buffers
- AnonPages

这 3 个字段和内存类型有什么联系呢？

- Cached：表示的是 File-backed 类型内存的大小。这些缓存都放在 page cache 里。所有page cache里的页面(Cached)都是file-backed pages，不是Anonymous Pages。”Cached”与”AnonPages”之间没有重叠。
- SwapCached：Swap意思是交换分区，通常我们说的虚拟内存，是从硬盘中划分出的一个分区。当物理内存不够用的时候，内核就会释放缓存区（buffers/cache）里一些长时间不用的程序，然后将这些程序临时放到Swap中，也就是说如果物理内存和缓存区内存不够用的时候，才会用到Swap。而 SwapCached 就是被交换到磁盘上的内存的大小。“Cached”和”SwapCached”两个统计值是互不重叠的。更进一步了解请参考：[linux内核tmpfs/shmem浅析](https://blog.csdn.net/ctthuangcheng/article/details/8916065)，[FREE命令显示的BUFFERS与CACHED的区别](http://linuxperf.com/?p=32)，[/PROC/MEMINFO之谜](http://linuxperf.com/?p=142)
- AnonPages：表示的是 Anonymous 类型内存的大小。但如果是基于 tmpfs 的 shared Anonymous 类型的内存的话，是存到 page cache 中的。参考：[/PROC/MEMINFO之谜](http://linuxperf.com/?p=142)，[Tmpfs](https://www.kernel.org/doc/Documentation/filesystems/tmpfs.txt)
- buffers：表示是和 block 有关的、非 File-backed 类型的内存。

**对于 buffers，“和 block 有关的、非 File-backed 类型的内存”是什么意思呢？**
page cache 是保存和磁盘有关缓存。如果缓存的数据是“从文件加载的” 并且 “文件是从磁盘加载的”（ a file and a block representation）（大部分情况都是这样），那么这样的缓存都会放到 page cache 里。
但还有一部分缓存是“磁盘相关的”，但不是“和文件相关的”，这样的缓存放到 buffer cache 里。例如：直接读写块设备(raw block IO)、以及文件系统元数据(metadata)。

参考：
[linux内核tmpfs/shmem浅析](https://blog.csdn.net/ctthuangcheng/article/details/8916065)
[FREE命令显示的BUFFERS与CACHED的区别](http://linuxperf.com/?p=32)
[/PROC/MEMINFO之谜](http://linuxperf.com/?p=142)


##二、关于 Linux 共享内存
共享内存有 3 种方式：

- System V 共享内存：System V shared memory(shmget/shmat/shmdt)
- POSIX 共享内存：POSIX shared memory(shm_open/shm_unlink)
- MMAP 共享内存：Shared mappings – mmap(2) 

**关于 MMAP：**
MMAP 本来的是存储映射功能。它可以将一个文件映射到内存中（page cache 中），在程序里就可以直接使用内存地址对文件内容进行访问，这可以让程序对文件访问更方便。由于这个系统调用的特性可以用在很多场合，所以Linux系统用它实现了很多功能，并不仅局限于存储映射。

**关于 System V 和 POSIX 共享内存：**
这两者的共享内存都是使用 tmpfs 实现的，只不过 POSIX 共享内存是通过用户空间挂在的tmpfs文件系统实现的（如果用户不开启 tmpfs 功能，实现不了），而system V的共享内存是由内核本身的tmpfs实现的。
而 POSIX 共享内存内部，其实是利用 MMAP 实现的，只不过映射的是tmpfs文件系统上的文件。


什么是tmpfs？Linux提供一种“临时”文件系统叫做tmpfs，它可以将内存的一部分空间拿来当做文件系统使用，使内存空间可以当做目录文件来用。现在绝大多数Linux系统都有一个叫做/dev/shm的tmpfs目录，就是这样一种存在。

参考：
关于共享内存：
[linux内核tmpfs/shmem浅析](https://blog.csdn.net/ctthuangcheng/article/details/8916065)：讲了内存共享和 page、page cache 、swap 的关系。
[浅析 Linux 的共享内存与 tmpfs 文件系统](http://blog.jobbole.com/111665/)
[Linux系统中的tmpfs、/dev/shm、tmp以及共享内存机制](https://www.lengyuewusheng.com/2017/08/12/00016_Linux%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98%E6%9C%BA%E5%88%B6/)
[linux共享内存的设计](https://blog.csdn.net/dog250/article/details/5303709)
[Linux 进程间通信 : 共享内存（上）](https://cloud.tencent.com/developer/article/1005542)
[Linux进程间通信：共享内存 （下）](https://cloud.tencent.com/developer/article/1005543)


关于IO：
[漫谈linux文件IO](http://elf8848.iteye.com/blog/1944226)：讲了写数据所经历的从内核到磁盘的整个过程，还讲了 raw io 和 direct io 的区别。挺不错的。
[System-based I/O vs. Raw I/O](https://www.itworld.com/article/2785965/open-source-tools/raw-disk-i-o.html)：讲了系统IO调用和RawIO的区别。

关于 buffer 和 cache 的区别：
[What is the major difference between the buffer cache and the page cache? Why were they separate entities in older kernels? Why were they merged later on?](https://www.quora.com/What-is-the-major-difference-between-the-buffer-cache-and-the-page-cache-Why-were-they-separate-entities-in-older-kernels-Why-were-they-merged-later-on)
[What is the difference between Buffers and Cached columns in /proc/meminfo output?](https://www.quora.com/What-is-the-difference-between-Buffers-and-Cached-columns-in-proc-meminfo-output)


##3，Debug Linux
[Exploring the Linux Storage Path - Tracing block I/O kernel events](https://www.ibm.com/developerworks/community/blogs/58e72888-6340-46ac-b488-d31aa4058e9c/entry/exploring_the_linux_storage_path_tracing_block_i_o_kernel_events?lang=en)：如何跟踪 block IO 事件。

如何使用 ftrace 进行 debug linux：
[Debugging the kernel using Ftrace - part 1](https://lwn.net/Articles/365835/)
[Debugging the kernel using Ftrace - part 2](https://lwn.net/Articles/366796/)
[A look at ftrace](https://lwn.net/Articles/322666/)

跟踪 page cache：
[Linux Page Cache Hit Ratio](http://www.brendangregg.com/blog/2014-12-31/linux-page-cache-hit-ratio.html)：这个是讲一个叫“cachestat”的工具的，用来观察 page cache 的 hit、miss 等指标。这里面还讲了一些观察 page cache 的其它方法和工具。
[FIO (Flexible I/O Tester) Part5 – Direct I/O or buffered (page cache) or raw performance?](http://tfindelkind.com/2015/08/10/fio-flexible-io-tester-part5-direct-io-or-buffered-page-cache-or-raw-performance/)：如何使用 FIO 工具测试 DirectIO、page cache、raw 的性能。里面讲了很多

工具：
[perf-tools](https://github.com/brendangregg/perf-tools)：测试和观察 linux 性能的工具集，有很多工具可以用。
[Linux Page Cache Hit Ratio](http://www.brendangregg.com/blog/2014-12-31/linux-page-cache-hit-ratio.html)：这个是讲一个叫“cachestat”的工具的，用来观察 page cache 的 hit、miss 等指标。这里面还讲了一些观察 page cache 的其它方法和工具。
[page cache monitoring kernel patch](https://lkml.org/lkml/2011/7/18/326)：一个观察 page cache 用的 kernel patch，这个工具有两个非常棒的特点：进程级别的使用比率 和 文件级别的使用比率。
[pcstat](https://github.com/tobert/pcstat)：这个工具使用 mincore 来显示“文件有多少大小内容在 page cache 里”，非常不错。
(fincore是用来查看文件在内存中的状态，有关fincore的实现可以在linux下man mincore，mincore是根据缓存buffer指针来其指向的缓冲区判断在cache中的状态，fincore就是在mincore的基础上直接操作文件，就是通过对文件mmap获得指针，再调用mincore来判断。)


关于 page cache：
[Linux文件读写机制及优化方式](http://os.51cto.com/art/201609/517642.htm)：讲了 linux 读写数据时的流程，和一些名词解释。还讲了普通的 read/write 是如何运行的，都读了哪些缓存。
[Linux Cache 机制探究](http://www.penglixun.com/tech/system/linux_cache_discovery.html)
[block(块),page(页),buffer cache（块缓冲）区别与联系](https://blog.csdn.net/menogen/article/details/32701457)
[Linux 内核的文件 Cache 管理机制介绍](https://www.ibm.com/developerworks/cn/linux/l-cache/index.html)：介绍了 cache 的机制，可以看明白用户调用的整个过程。非常好。
[Linux Page Cache Basics](https://www.thomas-krenn.com/en/wiki/Linux_Page_Cache_Basics)：外国的一个文章，讲了什么是“脏页(Dirty Page)”，如何测试。入门级的文章，非常好。
（从这个文章上看出，mmap 也是用了 page cache）
[Linux 内核的文件 Cache 管理机制介绍](https://www.ibm.com/developerworks/cn/linux/l-cache/index.html)：讲了 Linux 系统是如何使用 cache 的、和 cache 的原理，Linux 底层原理性的东西。
[Linux 中的零拷贝技术，第 1 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy1/index.html)：讲了`零拷贝`技术的分类
[Linux 中的零拷贝技术，第 2 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy2/index.html)：讲了`零拷贝`技术的具体的使用场景和局限性，非常不错。
[计算机底层知识拾遗](https://blog.csdn.net/column/details/computer-os-network.html)：一个讲 linux 的专栏，可以看看，很好的总结。



1，FileChannel写DirectBuffer之后，数据就到了PageCache里面，就到了MapperByteBuffer里面，因为MapperByteBuffer被用作了文件的cache

应该不是写到 具体文件里，是写到了 page cache 里。MapperByteBuffer 读时，也是从 page cache 里读。


2，
先写mappedByteBuffer，然后将数据刷新到磁盘，这个时候通过 pmap查看进程的内存:
00007fd4e8600000 10240 4 0 rw-s- /tmp/1
刚开始，进程所占的文件的cache里面，dirty为0，然后，fileChannel写入后，dirty发生了变化:
00007fd4e8600000 10240 4 4 rw-s- /tmp/1
验证了我们的流程：directBuffer被写入到了MapperByteBuffer（本质是，用户空间的数据，被写到了系统的page cache里面）。

上面这个如何测试？
1，一个进程中，一个 channel 和 mapped，一个写另一个读？
2，两个进程，一个 channel 和 mapped，一个写另一个读？
3，再测试 10 批数据，分批写入 buffer，然后一次通过 channel 写入文件，然后 force，和“分10次写入 mapped，并且每次都 force”。看看哪个速度快。

4，如果确认读了 page cache 呢？

5，关于查看 page cache 的工具，smaps，maps，pmap，sar
一个非常好的文章，讲了很多方法来监视 page cache：
[Linux Page Cache Hit Ratio](http://www.brendangregg.com/blog/2014-12-31/linux-page-cache-hit-ratio.html)
[perf tools: pagecache monitoring](https://lwn.net/Articles/452050/)：监控工具 perf tools

smaps:
[linux 内存查看方法：meminfo\maps\smaps\status 文件解析](https://www.cnblogs.com/jiayy/p/3458076.html)
[Linux smaps接口文件结构](https://blog.csdn.net/linuxchyu/article/details/26154865)

pmap:
[Linux pmap](https://jaminzhang.github.io/linux/Linux-pmap/)
[Linux Pmap 命令 - 查看进程用了多少内存](https://linux.cn/article-2217-1.html)

sar:
[ sar 找出系统瓶颈的利器](http://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/sar.html)
[linuxr下sar调优工具的深入分析](https://blog.csdn.net/macky0668/article/details/6839525)


6，Activemq 写文件是如何做的呢？用没用 NIO？

7，查看 page cache 使用的命令

- cat /proc/meminfo ：看 Cached 
- top -b -n1 | head：看 Swap 后面的 Cached
- /usr/bin/time -v firefox：可以看某个应用的 Major page fault 和 minor page fault 问题。

清除 page cache :[Linux下清除缓存 drop_caches，sysctl（备忘）](https://blog.csdn.net/Sky_qing/article/details/8988461)
如果没有被消除掉，可以看 [什么手工DROP_CACHES之后CACHE值并未减少？](http://linuxperf.com/?p=201)

8，RocketMQ 的写文件，是不是没有 [MappedByteBuffer的一点优化](https://lishoubo.github.io/2017/09/27/MappedByteBuffer%E7%9A%84%E4%B8%80%E7%82%B9%E4%BC%98%E5%8C%96/) 最后的那块说的那么复杂？因为写到了 page cache 里后，读的话就可以读到了？RocketMQ是如何读的 呢？

9，关于零拷贝
[Zero Copy I: User-Mode Perspective](https://www.linuxjournal.com/article/6345)
[linux零拷贝原理（一）](https://leokongwq.github.io/2017/01/12/linux-zero-copy.html)：上面文章的翻译

10，总结minor and major page fault in Linux?
minor page fault 有没有影响？感觉不是问题。
https://www.quora.com/What-is-the-difference-between-minor-and-major-page-fault-in-Linux
http://blog.scoutapp.com/articles/2015/04/10/understanding-page-faults-and-memory-swap-in-outs-when-should-you-worry

11，关于 sendfile 看下面的文件就知道 sendfile 对应的 java 方法。但 rocketmq 文档只说只是 bio 传输，不能使用 nio，这个事怎么确认呢？因为 FileChannel 就是 nio 下面的包。是不是说 sendfile 方法是阻塞的呢？
https://www.ibm.com/developerworks/cn/java/j-zerocopy/：文章上还有 transfer 的测试代码。transfer 和 mmap 比哪个快呢？

sendfiile 也是使用 page cache 的。


12，kafka 为什么这么快。
https://mp.weixin.qq.com/s?__biz=MzIxMjAzMDA1MQ==&mid=2648945468&idx=1&sn=b622788361b384e152080b60e5ea69a7#rd

13，如果响应慢，应该如果解决？
最近页面打开比较慢，经查看，web server 的cpu、load以及数据库服务器的cpu、load、memory都很正常。但是web server 8g的内存即将消耗尽。将memcached迁移走，页面响应明显变快。
另外，sar -B 看到，page faults达到每秒15000左右，但是page in 和page out都不高。说明大多数是从内存cache里读入到可用区域的中断

排查 cpu，内存，硬盘，网络。
vmstat 可以展现给定时间间隔的服务器的状态值,包括服务器的CPU使用率，内存使用，虚拟内存交换情况,IO读写情况。

14，RocketMQ 的零拷贝的选择方式。比较一下

15，阿里的几个 kafka 和 rocketmq 的对比：
https://yq.aliyun.com/articles/62832
https://yq.aliyun.com/articles/62833?spm=a2c4e.11153940.blogcont62832.11.5aec4735E8T6Md