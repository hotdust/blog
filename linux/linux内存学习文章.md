#结构
[page fault带来的性能问题](https://yq.aliyun.com/articles/55820)
[Linux文件读写机制及优化方式](http://os.51cto.com/art/201609/517642.htm)：讲了 linux 读写数据时的流程，和一些名词解释。还讲了普通的 read/write 是如何运行的，都读了哪些缓存。
- [What is some existing documentation on Linux memory management?](https://landley.net/writing/memory-faq.txt)
- [Memory – Part 1: Memory Types](https://techtalk.intersec.com/2013/07/memory-part-1-memory-types/)：这个是个系列，都看一下，哪些作用不大就跳过去。
- [Understanding the Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/)：看一下，内容挺多的，感觉看不明白就的跳过去。
- [linux内核tmpfs/shmem浅析](https://blog.csdn.net/ctthuangcheng/article/details/8916065)

#工具

- [Linux常用的性能监控工具](http://debugo.com/linux-perf-tools/)
- [深入理解iostat](http://bean-li.github.io/dive-into-iostat/)
- [容易被误读的IOSTAT](易被误读的IOSTAT)


#机器慢，要排查什么指标
参考：[Linux常用的性能监控工具](http://debugo.com/linux-perf-tools/)
##1，CPU
###Load Average
使用 uptime 或 top 命令查看 CPU 最近的负载。

###mpstat 
mpstat是一个报告处理器相关统计信息的工具。
```
$ mpstat 
Linux 2.6.39-400.17.1.el6uek.x86_64 (oipdb1)    04/22/2014      _x86_64_        (48 CPU)
10:54:14 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
10:54:14 AM  all   60.58    1.18   36.07    0.66    0.00    0.84    0.00    0.00    0.66
```

##2，IO
主要看

- read/write 字节数
- util：在统计时间内所有处理IO时间，除以总共统计时间。例如，如果统计间隔1秒，该设备有0.8秒在处理IO，而0.2秒闲置，那么该设备的%util = 0.8/1 = 80%，所以该参数暗示了设备的繁忙程度。一般地，如果该参数是100%表示设备已经接近满负荷运行了。
- awailt：每一个IO请求的处理的平均时间（单位是毫秒）

命令：

-  sar -d
- iostat

##3，内存
主要看

- 物理内存
- 缓存：page cache、page fault、swap 等。
- 其它内存情况：/proc/meminfo

命令：

- sar -r
- sar -R
- sar -B
- sar -W
- sar -S
- vmstat
- free
- /proc/slabinfo和vmstat -m

#4，进程和运行队列
主要看：

- 等待 CPU  的进程数
- 当前进程数
- 每秒创建进程数量
- 上下文切换数

命令：

- sar -q
- uptime
- sar -w