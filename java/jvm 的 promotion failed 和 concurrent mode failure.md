# 前言

对于采用CMS进行老年代GC的程序而言，尤其要注意GC日志中是否有promotion failed 和 concurrent mode failure 两种状况。为什么要注意这两种情况呢？
> 应用线程将会全部停止（相当于网站这段时间无法响应用户请求），进行压缩式垃圾收集（回退到 Serial Old 算法）。

# 一、关于 Promotion Failed
promotion failed 是在进行Minor GC时，eden 和 survivor space放不下、对象只能放入老年代，而此时老年代也放不下造成的。

下面是一个一个 promotion failed 的例子
>
2019-01-27T08:47:53.128+0800: 217785.642: [GC (Allocation Failure) 
2019-01-27T08:47:53.128+0800: 217785.642: [ParNew (0: promotion failure size = 4)  (1: promotion failure size = 9)  (promotion failed): 426104K->426265K(471872K), 0.1616134 secs]
2019-01-27T08:47:53.290+0800: 217785.804: [CMS: 1412191K->254596K(1572864K), 1.4969890 secs] 1838215K->254596K(2044736K), [Metaspace: 112138K->112138K(1157120K)], 1.6599627 secs] [Times: user=1.68 sys=0.00, real=1.66 secs]

## 1，产生原因
提升失败原因：
- 新生代提升太快
- 老年代碎片太多，放不下大对象提升（表现为老年代还有很多空间但是，出现了 promotion failed）

## 2，解决方案
对于`新生代提升太快`的原因，有可能是应用中生成新对象速度过快，但还无法回收。或者可能是因为设置了下面两个参数：
- MaxTenuringThreshold：设置晋升到老年代的年龄，默认 15 次 Minor GC 后晋升到老年代。
- PretenureSizeThreshold：对象大小超过一定数值的直接晋升到老年代


对于`老年代碎片太多，放不下大对象提升`问题，（1）如果频率太快或者 Full GC 后空间释放不多的话，说明空间不足，首先可以尝试调大老年代空间（2）如果内存不足，可以设置进行 n 次 CMS 后进行一次压缩式 Full GC，参数如下：
- XX:+UseCMSCompactAtFullCollection：允许在 Full GC 时，启用压缩式 GC
- XX:CMSFullGCBeforeCompaction=n     在进行 n 次，CMS 后，进行一次压缩的 Full GC，用以减少 CMS 产生的碎片

还有一个方法，增加 Survivor 空间的。因为 Survivor 空间不足，才放到 Old 空间。所以如果使用的对象，不是每次 Minor GC 都能释放，可以增加 Survivor 空间。如果 Survivor 空间很小的话，Minor GC 无法回收的`存活对象`就可能无法放到 Survivor 空间，而是直接放到 Old 空间。


# 二、关于 concurrent mode failure
concurrent mode failure是在执行CMS GC的过程中同时有对象要放入老年代，而此时老年代空间不足造成的（有时候“空间不足”是CMS GC时当前的浮动垃圾过多导致暂时性的空间不足）。

## 1，产生原因
原因可能会有两个：
- 在 CMS 启动过程中，新生代提升速度过快，老年代收集速度赶不上新生代提升速度。
- 在 CMS 启动过程中，老年代碎片化严重，无法容纳新生代提升上来的大对象

## 2，解决方案
和 `Promotion Failed` 解决方案的前两个方案一样。


# 三，在 GC 的时候其他系统活动影响

有些时候系统活动诸如内存换入换出（vmstat）、网络活动（netstat）、I/O （iostat）在 GC 过程中发生会使 GC 时间变长。

前提是你的服务器上是有 SWAP 区域（用 top、 vmstat 等命令可以看出）用于内存的换入换出，那么操作系统可能会将 JVM 中不活跃的内存页换到 SWAP 区域用以释放内存给线程使用（这也透露出内存开始不够用了）。内存换入换出是一个开销巨大的磁盘操作，比内存访问慢好几个数量级。

看一段 GC 日志：耗时 29.47 秒
```
Heap before GC invocations=132 (full 0):  
par new generation total 2696384K, used 2696384K [0xfffffffc20010000, 0xfffffffce0010000, 0xfffffffce0010000)  
eden space 2247040K, 100% used [0xfffffffc20010000, 0xfffffffca9270000, 0xfffffffca9270000)  
from space 449344K, 100% used [0xfffffffca9270000, 0xfffffffcc4940000, 0xfffffffcc4940000)  
to space 449344K, 0% used [0xfffffffcc4940000, 0xfffffffcc4940000, 0xfffffffce0010000)  
concurrent mark-sweep generation total 9437184K, used 1860619K [0xfffffffce0010000, 0xffffffff20010000, 0xffffffff20010000)  
concurrent-mark-sweep perm gen total 1310720K, used 511451K [0xffffffff20010000, 0xffffffff70010000, 0xffffffff70010000)  
2013-07-17T03:58:06.601-0700: 51522.120: [GC Before GC: : 2696384K->449344K(2696384K), 29.4779282 secs] 4557003K->2326821K(12133568K) ,29.4795222 secs] [Times: user=915.56 sys=6.35, real=29.48 secs]
```
再看看此时的 vmstat 命令中 si、so 列的数值，如果数值大说明换入换出严重，这是内存不足的表现。

解决方法：减少线程，这样可以降低内存换入换出；增加内存；如果是 JVM 内存设置过大导致线程所用内存不足，则适当调低 -Xmx 和 -Xms。

# 参考：
- [JVM 调优 —— GC 长时间停顿问题及解决方法 - ImportNew](http://www.importnew.com/22886.html)：本文的主要内容都是参考这个文章的。
- [JVM系列三:JVM参数设置、分析 - redcreen - 博客园](https://www.cnblogs.com/redcreen/archive/2011/05/04/2037057.html)

