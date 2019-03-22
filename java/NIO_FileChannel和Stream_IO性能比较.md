> 在 NIO 的 FileChannel 和 IO 的 Stream 性能进行比较时，都说 FileChannel 速度快，是不是真的这么快呢？为什么快呢？

#一、NIO 快的说法
##说法1
原因有的说 NIO 操作是面向 Block（一块Buffer），而 Stream 是面向字节的，一次只能处理一个或多个字节。其实这么说太粗犷了，具体的说， Stream 形式可以一次处理一个字节，也可以一次处理很多字节，全看接口的实现和使用的使用方式。举例说明一下：

- DataOutputStream#writeChar：这个方法是用来写 Char 类型的，具体方法内部是如何写的呢？内部使用了 2 次 write(byte) 方法，把 Char 的高位和低位分别写了进去。而 writeLong 方法，调用了 8 次 write(byte) 方法。想一想，每次调用 write 方法时，都会做执行`系统调用`和`上下文切换`，会花费很多时间。
- DataOutputStream#write(byte b[], int off, int len)：这个方法是写一个 byte 数据，只调用一次 write 方法。如果我们把要写的 Long 型转换成 byte 数据，然后调用这个方法进行一次写的话，那么只执行一次的`系统调用`和`上下文切换`，速度会快很多。

上面的例子就说明了 Stream 一次处理一个字节和多个字节的接口。那 NIO 如何呢？如果 FileChannel 每次只写一个字节的话（ByteBuffer），也是一样的，也会花很长时间（MMAP 例外）；一次写多个 byte 数据也会很快。而且在 linux 2.6 内核（Redhat），Stream 要比 FileChannel 更快（包括 MMAP）。
(这里的写，IO 和 NIO 都是指使用 write 方法。)
<br>

##说法2
有的文章说，磁盘是以扇区( 512 byte )为最小单位，Stream 一次处理一个或多个字节，有些浪费。但这应该是说每次写都进行 flush 刷盘的情况吧。不管 IO 还是 NIO 每次写数据时，都只是写到系统的内存中，也就是 page cache 这块缓存上。只有调用 flush( IO ) 或 force( NIO )，才会把 page cache 中的数据写到磁盘上。所以只要不调用这些方法，是不涉及扇区的。而且，就算是每次调用刷盘方法，IO 和 NIO 速度基本也差不多，因为 NIO 也可以每次只写一个字节。
<br>



#二、MMAP 快吗？
简单地说，要是像上面一样，如果复制一个大 buffer 的内容的话，IO 和 MMAP 也差不多。但在把“大量数据”进行“小批量”写/读时，MMAP 性能好非常多。有意思的是，使用 DirectBuffer 进行对 MMAP 写时，反而比 Heap Byte 慢。实际上，在 DirectBuffer+Channel 写时，如果只是写（不是文件拷贝）的话，速度也是不快的，还是一个很大的 DirectBuffer 快（这里还没有完全理解为什么一个很大的 DirectBuffer 就快）。 

| 方法 | 机器 | 文件大小 | 每次写大小 | 总时间(s)（avg） | user time（avg） | sys time（avg） | 
|:-------|------|------|------|------|------|------|
| writeFilePartlyByFileStream | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 100B | 13+ | 3+ | 10+ |
| writeFilePartylyByChannel(HeapBuffer) | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 100B | 8+ | 3.5+ | 5+ |
| writeFilePartlyWithHeapByMMAP(DirectBuffer) | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 100B | 8+ | 2.7+ | 5.4+ |
| writeFilePartlyWithHeapByMMAP | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 100B | 0.5+ | 0.38+ | 0.15+ |
| writeFilePartlyWithBufferByMMAP | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 100B | 1+ | 0.8+ | 0.2+ |
| writeFilePartlyByFileStream | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 1K | 2.3+ | 0.5+ | 0.7+ |
| writeFilePartylyByChannel(HeapBuffer) | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 1K | 2.1+ | 1.3+ | 0.7+ |
| writeFilePartlyWithHeapByMMAP(DirectBuffer) | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 1K | 1.7+ | 0.8+ | 0.7+ |
| writeFilePartlyWithHeapByMMAP | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 1K | 0.5+ | 0.3+ | 0.2+ |
| writeFilePartlyWithBufferByMMAP | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD | 500M | 1K | 0.8+ | 0.6+ | 0.2+ |

**1，为什么 MMAP 快？**
有很多文章说 MMAP 快，例举几个：

 - 有一篇文章 [It's all about buffers: zero-copy, mmap and Java NIO](http://xcorpion.tech/2016/09/10/It-s-all-about-buffers-zero-copy-mmap-and-Java-NIO/) 上说，是因为 MMAP 在操作映射区域时没有 system call。
 - 还有一些文章 [Zero Copy I: User-Mode Perspective](https://www.linuxjournal.com/article/6345?page=0,0) 说，mmap 减少了 user space 的内存复制。但是无法联想到 Java 底层是如何做的，也没有找到相关文章。

但从方法调用方面也许可以看出点端倪，两个 IO 的方法签名如下：

- FileOutputStream#writeBytes：`private native void writeBytes(byte b[], int off, int len, boolean append)`。
- DirectByteBuffer(MappedByteBuffer的实现类)的 put 方法最终调用的是`unsafe.putByte`：`public native void putByte(long address, byte x);`

由此可见，unsafe#putByte 方法的第一个参数是个地址，应该是直接向地址写数据。而 FileOutputStream#writeBytes 方法应该是把数据先拷贝到内核，再写到具体的地址上。这两个方法感觉就像 Java 方法中的传值和传址调用一样。
想通过查看内存变化，看是否 write/read 会比 MMAP 会不会多使用内存，通过看“运行前、运行中、运行后”的 meminfo，使用的内存基本上都是一样的。

**2，为什么要用 mmap？**

- 减少一次内存拷贝。而且在进行读写时，更少的 system call。
- 如果用 Stream IO 需要把整个文件都读进来，进行缓存。MMAP 可以对整个文件进行操作，而不必不关心系统是如何做的。
- MMAP 可以反复使用文件中的某一块，而 Stream IO 则不可以，如果想这么做需要自己缓存一份数据。这样需要内存比较大。
- 可以共享内存。这样就不用每个进程都打开一份数据和文件。

有一些场景不适合使用 MMAP

- 对于大多数操作系统来说，MMAP 一个文件的 cost 要比“通过 read/write 读写“几十K字节”的 cost 多的多，所以 MMAP 还是适合处理读写大文件操作（这里是指文件大，而且读写的量也很大）。

- 人们喜欢 MMAP 是因为可以减少一次 copy 操作。但是 virtual memory mapping 操作也是一个很昂贵的操作。而且他有一些实际的缺点，人们也容易忽略这些缺点，因为多的那次内存拷贝看起来很慢，优化它会带来很大的提升。

- MMAP 的创建和销毁的 cost 非常显著。page table 要非常干净地 unmap everything。还要维护一个 mappings 的 list。TLB 在 ummap 后也要进行 flush。（cost 有多高呢？有没有测试或统计呢？）

- page fault 也是非常昂贵的，而且非常慢。(有多慢呢？有没有测试或统计呢？)

还有一些其它问题，请参看下面的文章，文章中经典两句，由此可见符合使用场景的测试非常重要：

- Zero Copy 不等于速度快。（Zero-copy" does not equate to "fast）
- `只拷贝数据一次`这种 Test Case，对于 MMAP 的测试来说，可能是非常差的一种 Case（But your test-suite (just copying the data once) is probably pessimal for mmap().）

[Re: mmap/mlock performance versus read](https://marc.info/?l=linux-kernel&m=95496636207616&w=2)
[Re: Integration of SCST in the mainstream Linux kernel](http://lkml.iu.edu/hypermail/linux/kernel/0802.0/1496.html)
<br>

#三，`DirectBuffer + FileChannel`性能如何？
在上面做 Stream IO 和 MMAP 对于小批量写时，也对`DirectBuffer + FileChannel`形式的写进行了测试。总的说来，比 Stream IO 快很多，但比 MMAP 还是要慢很多。而且还有一点和 MMAP 类似，DirectBuffer 并不一定快，上面的测试结果都是 HeapBuffer 更快，具体到底哪个更快，还要看业务实际场景。从上面的测试结果来看，当写 100B 数据时，HeapBuffer 更快，而 1K 数据时，DirectBuffer 更快。还有一点要注意，变成 1K 时，Stream IO 的速度也上来了。

**那 `DirectBuffer + FileChannel` 比 Stream IO 快的原因是什么呢？**
很多文章都说因为 DirectBuffer 在 JVM 堆外，直接在系统内核申请的空间原因。如果是这样的话，为什么 HeapBuffer 时也很快呢？而且 DirectBuffer 快的话，感觉应该是两个 channel 在进行交换时，使用 DirectBuffer 更快，因为减少数据拷贝和上下文切换。
从上面的数据来看，使用 Channel 时，系统调用的时间变少了。和 Stream IO 对比，少了 10S（但当写数据变成 1K 时，Stream IO 的系统调用时间也下来了）。对于内存的使用，可以使用 meminfo 观察一下，没有进行观察，但感觉使用的大小应该是一样的，可能使用区域是不一样的。最后，至于系统调用是如何变少的，还待研究。

从上面的数据来看，FileChannel 的形式和 MMAP 相比还是有差距的，但为什么 RocketMQ 不光使用 MMAP，还使用 FileChannel 呢？
有可能是使用场景不一样，结果也不一样。还有可能是其它原因，要再看看代码。
<br>

#四、NIO 如何使用性能最高？
看了 sendfile 和 mmap 的一些原理性知识后，总结起来，还是 channel 和 channel 之间传数据时，NIO 性能最好，因为这样减少`数据拷贝`和`上下文切换`。比如：文件拷贝(sendfile)、NIO socket Channel + FileChannel。
但是，也需要注意到 sendfile 使用方式也是有使用的局限性的。sendfile() 系统调用不需要将数据拷贝或者映射到应用程序地址空间中去，所以 sendfile() 只是适用于`应用程序地址空间不需要对所访问数据进行处理的情况`。相对于 mmap() 方法来说，因为 sendfile 传输的数据没有越过用户应用程序 / 操作系统内核的边界线，所以 sendfile () 也极大地减少了存储管理的开销。sendfile () 其它局限性，如下所列：

- sendfile() 局限于基于文件服务的网络应用程序，比如 web 服务器。据说，在 Linux 内核中实现 sendfile() 只是为了在其他平台上使用 sendfile() 的 Apache 程序。
- 由于网络传输具有异步性，很难在 sendfile () 系统调用的接收端进行配对的实现方式，所以数据传输的接收端一般没有用到这种技术。
- 基于性能的考虑来说，sendfile () 仍然需要有一次从文件到 socket 缓冲区的 CPU 拷贝操作，这就导致页缓存有可能会被传输的数据所污染。


下面就是一个文件拷贝的例子：

> 前提：
> - 拷贝文件大小：1G

```
// 拷贝文件时，byte[] 数组大小为 1M
private  static  void fileCopyByIO() throws Exception{
    long start = System.currentTimeMillis();

    File source = new File(FILE_PATH);
    File dest = new File(COPY_FILE_PATH);
    if(!dest.exists()) {
        dest.createNewFile();
    }

    FileInputStream fis = new FileInputStream(source);
    FileOutputStream fos = new FileOutputStream(dest);
    byte [] buf = new byte[WRITE_SIZE]; // WRITE_SIZE 为 1000 * 1000
//        byte [] buf = new byte[512];
    int len = 0;
    while((len = fis.read(buf)) != -1) {
        fos.write(buf, 0, len);
    }

    fis.close();
    fos.close();
    System.out.println("fileCopyByIO:" + (System.currentTimeMillis() - start));

}



private  static  void fileCopyByNIOWithTransfer() throws Exception{
    long start = System.currentTimeMillis();

    File source = new File(FILE_PATH);
    File dest = new File(COPY_FILE_PATH);

    if(!dest.exists()) {
        dest.createNewFile();
    }

    FileInputStream fis = new FileInputStream(source);
    FileOutputStream fos = new FileOutputStream(dest);
    FileChannel sourceCh = fis.getChannel();
    FileChannel destCh = fos.getChannel();

    destCh.transferFrom(sourceCh, 0, sourceCh.size());

    sourceCh.close();
    destCh.close();
    fos.close();
    fis.close();
    System.out.println("fileCopyByNIOWithTransfer:" + (System.currentTimeMillis() - start));

}

private  static  void fileCopyByNIOWithMMAP() throws Exception {
    long start = System.currentTimeMillis();

    File source = new File(FILE_PATH);
    File dest = new File(COPY_FILE_PATH);

    if(!dest.exists()) {
        dest.createNewFile();
    }

    FileInputStream fis = new FileInputStream(source);
    FileOutputStream fos = new FileOutputStream(dest);
    FileChannel sourceCh = fis.getChannel();
    FileChannel destCh = fos.getChannel();

    MappedByteBuffer mbb = sourceCh.map(FileChannel.MapMode.READ_ONLY, 0, sourceCh.size());
    destCh.write(mbb);


    sourceCh.close();
    destCh.close();
    fos.close();
    fis.close();
    System.out.println("fileCopyByNIOWithMMAP:" + (System.currentTimeMillis() - start));

}
```

时间对比如下：

| 方法 | 机器 | 总时间(s)（avg） | user time（avg） | sys time（avg） | 
|:-------|------|------|------|------|
| fileCopyByIO | Red Hat 4.4.7-16（2.6内核）+ 网络磁盘 + 8G内存 | 17+ | 0.7+ | 2.5 |
| fileCopyByNIOWithTransfer | Red Hat 4.4.7-16（2.6内核）+ 网络磁盘 + 8G内存 | 16+ | 0.15 | 2 |
| fileCopyByNIOWithMMAP | Red Hat 4.4.7-16（2.6内核）+ 网络磁盘 + 8G内存 | 16+ | 0.12 | 2 |
| fileCopyByIO | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD + 1G内存 | 3 | 0.27 | 1.5 |
| fileCopyByNIOWithTransfer | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD + 1G内存 | 2.2 | 0.11 | 1.3 |
| fileCopyByNIOWithMMAP | RedHat 4.8.5-11 (4.10.4-1.el7内核) + SSD + 1G内存 | 2.4 | 0.8 | 1.3 |

如果拷贝时的 buffer 设置的大一些的话，Stream IO 还是很快的。要注意几点：

- 如果 Stream IO 的 buffer 设置的很大的话，速度可能会变慢。上面的测试中，把 IO 拷贝时 buffer 设置成 10M 的话，比设置成 1M 时慢。
- 上面的测试中，IO 拷贝的速度和 NIO 速度一样的时候也是有的，其实 IO 速度也是很快的。
- 测试过程中，没有每次都对 page cache 进行清理。因为试了一下，page cache 对写数据基本没有什么影响。
- 测试的文件比较小，如果很大的话，NIO 优势可能会更明显。


参考文章：
[java 四种io实现速度对比](https://www.jianshu.com/p/e52db372d986)：文件拷贝的对比文章。在 Linux 2.6、4.0 (RedHat) 和 MacOS(10.13) 上进行测试，都是 MMAP 最快。但这个比较中，Stream IO 的写 buffer size 为 512 字节，所以这么比较感觉有点不太合适。
[Java NIO FileChannel versus FileOutputstream performance / usefulness](https://stackoverflow.com/questions/1605332/java-nio-filechannel-versus-fileoutputstream-performance-usefulness)：回答 1 中，说 NIO 快，但没有给出代码。
[Linux 中的零拷贝技术，第 2 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy2/index.html)
<br>


#五，为什么要用 DirectBuffer？什么时候用 DirectBuffer。
从现在了解的来看，在使用 Channel 写/读数据时，要使用 DirectBuffer。因为写数据时，如果不是 DirectBuffer 的话，Channel 会创建临时的 DirectBuffer，把 HeapBuffer 放到临时的 DirectBuffer 后，从 DirectBuffer 中读取数据写到目的地。为什么要创建临时 DirectBuffer，可能是因为防止 GC 干扰。可以参考：[Java NIO中，关于DirectBuffer，HeapBuffer的疑问？](https://www.zhihu.com/question/57374068)。


MMAP 写时候用不用 DirectBuffer 呢？
不一定，从上面的测试结果来看，使用 DirectBuffer 进行写小块数据时，反而没有 HeapBuffer 快。原因还没深入调查。
<br>

#总结
1，上面的测试场景大部分只是针对一个大文件的`顺序写`和`连续写`，没有读场景。应用需求不一样，场景也不一样，结果可能也会有所不同，当进行开发时，还需要针对具体场景进行测试。

2，是不是 NIO 就一定快呢？不一定，操作系统不一样，实现不一样，快慢也不一样。

3，不管是 NIO 还 IO，buffer 大小很关键，直接影响速度。

4，如果要达到写最高速度，可以进行调整 Linux IO 调度算法。可以参考下面的文章：

- [Linux I/O调度](http://www.cnblogs.com/sopc-mc/archive/2011/10/09/2204858.html)：讲了调试的几种类型。在 linux 2.6 内核测试，mmap 方法，deadline 1700ms 左右，CFQ 1900ms 左右，快 200ms 左右，100M 的数据。
- [调整 Linux I/O 调度器优化系统性能](https://www.ibm.com/developerworks/cn/linux/l-lo-io-scheduler-optimize-performance/index.html)：讲 IO 调试器的

同时你可能也想了解一下磁盘的问题：

-[聊聊Linux IO](http://0xffffff.org/2017/05/01/41-linux-io/)：介绍不同的磁盘在单线程/多线程的读写方面的不同。
- [Analyzing I/O performance in Linux](https://cmdln.org/2010/04/22/analyzing-io-performance-in-linux/)
- [Understanding Disk I/O - when should you be worried?](http://blog.scoutapp.com/articles/2011/02/10/understanding-disk-i-o-when-should-you-be-worried)
- [linux I/O优化 磁盘读写参数设置](http://www.cnblogs.com/276815076/p/5687814.html)

可能你还想了解一下算法方面：

- LSM-Tree 的设计便是合理的利用了存储介质的特性，做到了最大化的性能利用（磁盘换成SSD也依旧能有很好的运行效率）。

#参考：
[通过零拷贝实现有效数据传输](https://www.ibm.com/developerworks/cn/java/j-zerocopy/)
[Java文件映射[mmap]全接触](https://site.douban.com/161134/widget/articles/8506170/article/18487141/)
[认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)：写了 linux mmap 和 read/write 的内核处理过程。对理解 mmap 挺好的。
[java中的mmap实现](http://xiaoz5919.iteye.com/blog/2093323)：讲 java 实现，讲了一些 jdk 源码和一些 linux 工具
[If you use an input/output stream with a buffered output stream can greatly improve performance.](https://orangepalantir.org/topicspace/show/83)：主要讲 stream 写时候，写的块的大小，对速度的影响。如果写的块很大的话，性能是非常好的。在 linux 4.0+ 内核 和 SSD 硬盘上测试，整个写一块大数据，和 NIO 基本一样。
[Linux 内核的文件 Cache 管理机制介绍](https://www.ibm.com/developerworks/cn/linux/l-cache/index.html)：讲了 Linux 系统是如何使用 cache 的、和 cache 的原理，Linux 底层原理性的东西。
[Linux 中的零拷贝技术，第 1 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy1/index.html)：讲了`零拷贝`技术的分类
[Linux 中的零拷贝技术，第 2 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy2/index.html)：讲了`零拷贝`技术的具体的使用场景和局限性，非常不错。