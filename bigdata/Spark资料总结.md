
# 一、学习
## 1，安装Spark，并了解基础操作
首先安装上Spark，再执行一下基础操作，就可以了。这里的目的是通过Spark的Shell，了解一下Spark的基础操作。接下来看看文章下面的一些概念和作用什么的就可以，不用看的太细。

- [Spark快速入门指南 - Spark安装与基础使用](http://dblab.xmu.edu.cn/blog/spark-quick-start-guide/)


## 2，了解如何使用Java编写Spark程序
（1）先看一下官方的文档。如果对于不了解Spark的人来说，直接看官方文档可能很难理解，所以在官方文档下面有一个中文版的官方文档。

- [Spark编程指南(官方英文版)](http://spark.apache.org/docs/latest/programming-guide.html#initializing-spark)
- [Spark编程指南(中文版)](https://strongyoung.gitbooks.io/spark-programming-guide/content/)

（2）在看官方文档时，会看到关于RDD中使用Closure的问题，对于这个问题可以看一下下面3个文档来了解一一下。

- [理解Spark中的闭包(closure)](http://blog.csdn.net/u014729236/article/details/46662181)
- [Spark——共享变量](http://www.solinx.co/archives/570)
- [Spark 3. RDD 操作一 基础 ，放入方法，闭包，输出元素, 使用 K-V 工作](http://www.jianshu.com/p/ea41a6d02f34)

（3）在看官方文档时，可能还会看到一些Driver、Node、Partition等词汇。想了解关于Spark的一些基础概念的话，可以看下面的文章。

- [Spark里几个重要的概念及术语](http://blog.csdn.net/oopsoom/article/details/23857949)：这个文章对基本概念作了简单说明。
- [『 Spark 』2. spark 基本概念解析](http://litaotao.github.io/spark-questions-concepts)：这个文章对基本介绍的比较多，而且这个博客的其它博文，对基础概念大部都做了很详细的介绍。可以看看其它的文章。
- [Distributed Systems Architecture](https://0x0fff.com/spark-architecture-shuffle/)：这个文章里有很多关于Spark结构的图，介绍的看起来挺详细的。而文章里还有其它文章的链接，想详细知道的可以看看。
还有一个关于Spark架构的PPT和它的讲演视频：
- [Spark Architecture Video](https://0x0fff.com/spark-architecture-video/)
- [Spark Architecture PPT](https://0x0fff.com/spark-architecture-talk/)

（4）在看官方文档时，会看到一些Map和Reduce的API，下面的文章，让你能快速知道这个API的用法。

- [Spark RDD API详解(一) Map和Reduce](https://www.zybuluo.com/jewes/note/35032)

（5）进阶
- [jaceklaskowski/spark-streaming-notebook: Notes about Spark Streaming in Apache Spark](https://github.com/jaceklaskowski/spark-streaming-notebook)：关于 spark-streaming 的总结。讲的挺简单，而且讲到了一些别地方看不到的，例如：某块是怎么个流程，如何设置 debug 来看这些流程。读 Kafka 时的工具如何使用，都有什么功能等等。它的 github 上还有其它的 关于 spark 的书。
 - [spark 自己的分布式存储系统 - BlockManager](https://mp.weixin.qq.com/s?__biz=MzI3MjY2MTYzMA==&mid=2247483665&idx=1&sn=ba078a1b87036b199dd68a4988df788a&scene=21#wechat_redirect)




## 3，启动 spark 方法。
### （1）学习文章
- [运行您的第一个 Spark 应用](http://cwiki.apachecn.org/pages/viewpage.action?pageId=2886790)：装完环境后，测试 spark 环境是否可用
- [Spark集群安装和使用](http://blog.javachen.com/2014/07/01/spark-install-and-usage.html)：spark on yarn
- [YARN 上运行 Spark 应用](http://cwiki.apachecn.org/pages/viewpage.action?pageId=2886869)： spark on yarn 原理

### （2）常用命令
1，上传 jar 到 hdfs
hdfs dfs -put spark-test-1.0-SNAPSHOT.jar hdfs://nameservice1/test/sjp/monitor/spark-test.jar
2，启动 spark
spark2-submit --class test.SparkStreamingServiceDemo5 --deploy-mode cluster --master yarn hdf//nameservice1/test/sjp/monitor/spark-test.jar cloud_aggregation cm_input cm_output 10.31.187.4:9092


# 二、经验
## 1，安装完 YARN 后，安装 Spark 是否还需要 worker？
个人感觉不用。因为原来 Work 的工作，都由 NodeManager 来处理了。
- [Spark(一): 基本架构及原理](https://www.cnblogs.com/tgzhu/p/5818374.html)
- [Spark Standalone架构设计要点分析](http://shiyanjun.cn/archives/1545.html)：对 standalone 模式和原理讲的挺好的。

## 2，停止正在运行在 yarn 上的 spark 程序。
1，使用下面的命令找出对应的 application。`yarn application -list`
2，杀死 application。`yarn application -kill application_1537252157889_00023`
参考：[如何使用 Curl 停止 Spark 中某个正在运行的 Yarn 作业](https://docs.azure.cn/zh-cn/articles/azure-operations-guide/hdinsight/aog-hdinsight-apache-spark-howto-kill-yarn-job-via-curl)

## 3，关于 Spark 程序的打包
15，在打包时，使用 maven-shade-plugin 打包，org.apache.spark group 的依赖设置成 `<scope>provided</scope>`，这些包不用打包，在环境上都有。
jar 大小可以减少很多，包含 spark-core_2.11、spark-streaming_2.11、spark-streaming-kafka-0-10_2.11 的包有 100多M，不包含只有 500K。其它的 jar 还是要包含的。
参考：[利用maven-shade-plugin打包包含所有依赖jar包](https://blog.csdn.net/kezhong_wxl/article/details/77622097)

为了避免每次spark上传很大的依赖包，还可以把大部分依赖放到hdfs上,然后在spark_default中指明spark.yarn.jars到hdfs目录

## 4，本地程序调试，不使用 setMaster
VM options中输入“-Dspark.master=local”

## 5，如何在Windows中使用Intellij idea搭建远程Hadoop开发环境？
https://www.zhihu.com/question/50820730/answer/124755135
https://blog.csdn.net/qq_31806205/article/details/80451743

## 6，spark 设置 hadoop 等配置方法

**1，在程序内部设置（测试过）**
> sparkConf.set("spark.hadoop.dfs.replication", "1");

好像只能 hadoop 能这么设置的（"spark.hadoop." + hdfs-site.xml 具体项目），别可能不行。原因：[How can I change HDFS replication factor for my Spark program?](https://stackoverflow.com/questions/46098118/how-can-i-change-hdfs-replication-factor-for-my-spark-program) 文章最下面的答案。

**2，命令行设置（没有测试过）**
spark-submit --conf spark.hadoop.fs.s3a.access.key=value

**3，在程序内，读 hdfs-site.xml 等配置文件（没有测试过）**
基本方法就是把这些文件放到 classpath 下，方法有 2 个。
- 方法1:
把 hdfs-site.xml 文件放到 resources 目录下。
- 方法2:
[Custom Hadoop Configuration for Spark from Python (PySpark)?](https://stackoverflow.com/questions/29571644/custom-hadoop-configuration-for-spark-from-python-pyspark)

更多参考：[Spark Configuration](https://spark.apache.org/docs/latest/configuration.html)

## 7，使用 saveAsHadoopFile 时生产 _SUCCESS 文件问题
使用 saveAsHadoopFile 保存文件时，有时候会产生 _SUCCESS 文件。使用下面的设置就可以不产生这个文件：
> sparkConf.set("spark.hadoop.mapreduce.fileoutputcommitter.marksuccessfuljobs", "false");

参考：[How to avoid _success file in Mapreduce Output Folder.](http://bigdatafindings.blogspot.com/2015/07/how-to-avoid-success-and-log-files-in.html)
我产生 _SUCCESS 文件的原因。


## 8，把 JavaPairDStream 输出到多个文件
（1）如果要输出到多个文件到指定目录，并且指定文件名的话，需要自己实现输出类。这个输出类要继承 MultipleOutputFormat 类。MultipleOutputFormat 是输出到多个文件的类，但如果要输出成我们指定的文件名的话，需要自己实现一些方法。

（2）在输出时，可以使用两个方式：
- Dstream.saveAsHadoopFiles：这种方式加上面的 MultipleOutputFormat 实现类，可以完成我们的需要。但必须还要输出以 "prefix-TIME_IN_MS[.suffix]" 为名字的文件夹，而且是必须输出。
- JavaRDD.saveAsHadoopFiles：可以不必须输出临时文件夹。但要输出一个 _SUCCESS 文件。设置`sparkConf.set("spark.hadoop.mapreduce.fileoutputcommitter.marksuccessfuljobs", "false")` 可以避免输出这个文件。

（3）在输出时，如果要输出到同一个文件中的话，可使用 rdd.coalesce(1) 把数据重新分区。如果 rdd 已经是 reduceByKey 这种已经分区完成的 rdd 的话，并且要按 key 相同的输出到一个文件的话，应该可以不使用  rdd.coalesce(1) 。要注意：如果不同的分区的数据，要交差输出到不同的文件的话，可能会有问题。

coalesce 功能是重新分区，相当于 repartition(x, true)。
```
pair.foreachRDD(rdd->{
    if (rdd.count() != 0) {
        rdd.saveAsHadoopFile("hdfs://localhost:9000/test/", String.class, String.class, RDDMultipleAppendTextOutputFormat.class);
//                rdd.coalesce(1).saveAsHadoopFile("hdfs://localhost:9000/test/", String.class, String.class, RDDMultipleAppendTextOutputFormat.class);

    }

});
```

参考：
- [how to make saveAsTextFile NOT split output into multiple file?](https://stackoverflow.com/questions/24371259/how-to-make-saveastextfile-not-split-output-into-multiple-file)：讲了为什么要使用 rdd.coalesce(1) 
- [spark-streaming多目录追加写](https://blog.csdn.net/ukakasu/article/details/80067068)：参考这个写的 writer 代码。
- [spark streaming 实现根据文件内容自定义文件名，并实现文件内容追加](https://blog.csdn.net/qq_19917081/article/details/56841299)：这个是 java 版本的一个实现。修改的类太多。不像上面那个符合设计要求。

## 9，Idea 开发时常配置的参数
在进行 Idea 开发时，需要配置一些本地开发用的参数。如果这些参数用代码写到程序中，在到生产环境使用时，忘记需要注释掉会很麻烦。所以，把一些参数在 Idea 的`Run/Debug Configurations`中进行设置，不会影响代码。常用配置如下：

### (1)在 VM Option：
- -Dspark.master=local ：设置本地启动
- -Dspark.hadoop.dfs.replication=1 ：设置 hadoop 写的数据的副本为 1 份。一般在自己机器上安装 hadoop 时，都是单 dataNode 的。如果不设置这个，Append 数据时可能会产生一个错误。
- -Dspark.hadoop.fs.default.name=hdfs://localhost:9000 ：设置 Hadoop 的 默认服务名。一般在自己机器上安装 hadoop 时，如果在这里设置些项目的话，使用`hdfs:///xxx`方式的话，会报错。如果写命名`hdfs://nameserver/xxx`的话不会报错，但这样程序的可移植性就低了。

## 10，注意进行 cache
请看下面的代码，下面的代码逻辑很简单：1，生成 pair  2，保存 pair。如果像下面这样做的话，`生成 pair` 代码里面的`System.out.println("pair called");`会输出 4 遍。但如果在`生成 pair`和`保存pair`代码之间加 `pair.cache` 的话，就会只输出 2 遍。这说明，不进行 cache 的话，保存时使用 pair 时候，又进行了一次和`生成 pair`一样的过程。所以要注意使用 cache 功能。

```
// 生成 pair
JavaPairDStream<String, String> pair = messages.map(ConsumerRecord::value)
        .mapToPair(log -> {
            System.out.println("pair called");
                return new Tuple2<>("", "");
            }
        });
// 保存 pair
pair.foreachRDD(rdd->{
    long rddCnt = rdd.count();
    if (rddCnt != 0) {
        rdd.saveAsHadoopFile("", String.class, String.class, CustomClass.class);
    }
});

```

可能又会想，如果把`生成`和`保存`操作，像下面这样连上写，会不会只生成 2 遍呢？经过测试，也会生成 4 遍。所以，要注意进行 cache。
```
// 生成并保存 pair
	messages.map(ConsumerRecord::value)
        .mapToPair(log -> {
            System.out.println("pair called");
                return new Tuple2<>("", "");
            }
        }).foreachRDD(rdd->{
			    long rddCnt = rdd.count();
			    if (rddCnt != 0) {
			        rdd.saveAsHadoopFile("", String.class, String.class, CustomClass.class);
			    }
		});
```


## 11，在 transform 中 print
spark 的设计是在 transform 时，我们想打印 transform 过程中的值。在本地测试时是可以看到的，但在 yarn cluster 模式下，看不到 print 的内容。例如：
```
JavaPairDStream<String, String> servicePair = messages.map(ConsumerRecord::value)
        .filter(log -> log.contains("\"logType\":\"service\""))
        .mapToPair(log -> {
            try {
                // 下面的输出，不会输出到 driver 的 stdout log 中，会输出到 Exectutor Log 中。
                System.out.println("service log:" + log);
                String key = ...省略...;
                return new Tuple2<>(key, ...省略...);
            } catch (Exception e) {
                e.printStackTrace();
                return new Tuple2<>("", "");
            }
        });
```

原因应该是，yarn cluster 收集的 stdout log 是 driver 的输出，不是 Executor 的输出。transform 中的 print 输出，输出到了 Exectutor log 中。想看 Executor log 的话，可以从 spark 的 WebUI 中看到。进入 WebUI 后，选择上面菜单栏中的`Executors`，然后可以看到 Executors 列表。在列表中，最后一列是 log 列，可以选择看 stdout 或 stderr log。

这篇文章 [Spark losing println() on stdout](https://stackoverflow.com/questions/33225994/spark-losing-println-on-stdout) 是 print 看不到相关的一篇文章，点赞最多的回答可能不是正确的，但每个回答下面的评论非常好，可以看看。

## 12，关于 Spark Streaming 中 LocationStrategies 的设置
- LocationStrategies.PreferBrokers()：仅仅在你 spark 的 executor 在相同的节点上，优先分配到存在  kafka broker 的机器上；
- LocationStrategies.PreferConsistent()：大多数情况下使用，一致性的方式分配分区所有 executor 上。（主要是为了分布均匀）
- LocationStrategies.PreferFixed()：如果你的负载不均衡，可以通过这两种方式来手动指定分配方式，其他没有在 map 中指定的，均采用 preferConsistent() 的方式分配；

## 13，什么样的类需要声明 接口？
- 在 Driver 里声明的`类实例`，在 Excecutor 里使用时，这个`类`需要声明 Serializable 接口。
- 不需要实例化的`工具类`，不需要声明 Serializable 接口。
- 每个实例都是在 Executor 上合建的类，不需要声明 Serializable 接口。

参考：
- [Spark 中的序列化陷阱](https://segmentfault.com/a/1190000012353884)
- [Spark中的序列化机制](https://blog.csdn.net/u011491148/article/details/46910803)

## 14，流式统计的几个难点
[流式统计的几个难点](https://segmentfault.com/a/1190000003048757)

## 15，Spark Streaming 数据进行排序
使用 transform 或 transformToPair 进行排序。transform 可以把 rdd 转化成我们想要的类型，所以 transform 方法的 Function 需要返回一个 rdd。注意：这个 Function 是 spark 类包中的接口。
```
JavaPairDStream<Integer,String> sortedStream = swappedPair.transformToPair(
    new Function<JavaPairRDD<Integer,String>, JavaPairRDD<Integer,String>>() { 
     @Override 
     public JavaPairRDD<Integer,String> call(JavaPairRDD<Integer,String> jPairRDD) throws Exception { 
        return jPairRDD.sortByKey(false); 
        } 
}); 
```

## 16，聚合时保存数据里，最小那个时间字段
在聚合时，我们可能会想聚合一类数据，这类数据都有时间字段，聚合后我们想保留此类数据中，最小那个时间。做法如下：

1. 我们把数据转成 pair，把时间字段放到 pair 的 value 里，做为 value 的一部分。例如：Tuple2<Integer, String>，Integer 是要统计的数值。
2. 使用 reduceByKey 聚合数据。
```
.reduceByKey((v1, v2) -> {
    // 保存小的时间
    String smallDate = v1._2.compareTo(v2._2) < 0 ? v1._2 : v2._2;
    return new Tuple2<Integer, String>(
            v1._1 + v2._1, // reduce 个数相加
            smallDate);
});
```

## 17，如果解决数据倾斜
- [Spark性能优化指南——高级篇](http://lxw1234.com/archives/2016/05/663.htm)
- [Spark性能调优](https://www.jianshu.com/p/4c584a3bac7d)
- [超实用的Spark数据倾斜解决姿势，学起来！](https://dbaplus.cn/news-73-1460-1.html)

## 18，Spark Streaming WebUI 使用说明
https://github.com/jaceklaskowski/spark-streaming-notebook/blob/master/spark-streaming-webui.adoc

## 19，aggregate 的理解
[Spark的fold()和aggregate()函数](https://www.jianshu.com/p/15739e95a46e)

## 20，Yarn 的 Executor 的 CPU 和 内存如何设置
- [Spark On YARN内存和CPU分配](https://blog.csdn.net/fansy1990/article/details/54314249)
- 设置 CPU 核数（yarn.nodemanager.resource.cpu-vcores）时，大数据组的方式是设置成：`( CPU core size - 1) * 2`，让程序竞争使用。

## 21，User class threw exception: org.apache.spark.SparkException: Job aborted due to stage failure:  java.io.FileNotFoundException: Too many open files

- 去掉过多的 print。
- 加大内存。
- ~~把 spark.shuffle.manager 设置成 SORT~~ 。 Spark 2.0 Hash Based Shuffle退出历史舞台，只有 SortShuffleManager。
- 修改 partition 数量。用`rdd.getNumPartitions()`查看有多少个 paritition。用`someBigSDF.repartition()`设置 partition 数量。
- 看系统的配置。`/etc/security/limits.conf`中配置了 fd 数量，以 “*” 开头的部分是对所有用户的。再看看其它应用或框架在启动时，有没有设置这些，或者在`/etc/security/limits.d`下加一些配置。
- 注意 CDH 中各个 role 的 `Maximum Process File Descriptors` 是否配置了。这里配置了的话，应该是在系统之上影响 fd 数量。
- [Fixing a “Too many open files” exception](https://www.codacy.com/blog/fixing-a-too-many-open-files-exception/)：这里有些建议，还有看 fd 数量的方法，没有测试这个方法。

自己的修改：
- 内存从 2000 加大到 2500
- 去掉 rdd.print() 等输出

**自己代码产生问题的原因**
自己代码产生的问题并不是上面的原因造成的，而 kafka producer 使用完没有 close 的原因。使用完 close 掉就不再发生问题了，如果要写的更好一些，可以使用一个 producer pool，减少 producer 的生成和销毁。
- [fsalem/Spark-Kafka-Writer](https://github.com/fsalem/Spark-Kafka-Writer)
- [Spark Streaming Connection Pooling – Puneet Singh – Medium](https://medium.com/@bigdataengineer/spark-streaming-connection-pooling-3fa183ca6ded)
- [Spark Streaming Connection Pooling – Puneet Singh – Medium](https://medium.com/@bigdataengineer/spark-streaming-connection-pooling-3fa183ca6ded)



## 22，Spark Streaming job 延迟
### 什么是延迟呢？
假如我们设置成`每 1 分钟执行一次`的话，如果数据在 1 分钟中处理完成，就那算没有延迟。如果 1 分钟内没有完成，那下 1 分钟的数据的处理无法进行（如果发生延迟，上一个时间点的数据处理没有完成，spark 会继续接收下一个时间点的数据，但不会处理数据）。

### 如果判断是否发延迟呢？
是否发生延迟，可以通过 WebUI 的 Streaming 页面中的 `active batches`可以看出。如果`Scheduling Delay`栏里时间超过设置的执行批次的时间，并且有很多 job 的状态是`queued`状态的话，应该是延迟了。
![spark_streaming_delay](media/spark_streaming_delay.png)


### 如果判断发生延迟的原因？
在 Completed Batches 或 Active Batches 里面，看 Processing Time 多的 batch。然后点`具体的 Batch Time`可以查看执行 job 的时间。
![spark_streaming_delay_batch](media/spark_streaming_delay_batch.png)

从下面的第一行就可以看出花费了 40s，点击`+details`还可以查看使用是哪部分代码。
![spark_streaming_delay_job](media/spark_streaming_delay_job-1.png)

这样就可以定位代码了，然后在本地对代码进行修改和测试，是看看是为代码为什么花费时间长，能不能减少。

另外，上图中有的 job 状态是 skipped，这是什么意思呢？这个代表有的 RDD 处理已经使用了 cache，不需要重新计算了。看 RDD 是否使用了 cache，也可以在 transform 中加入 print 代码，然后在本机启动，看看输出多少次 print 代码。如果 3 个 rdd action 处理时，都出现了 transform 中的 print 代码，说明 transform 被执行了多次。

### 解决的问题
#### 1，json 转对象使用大量 cpu，增加 cpu cores
通过上面的方法，判断出是 json 对象转换时花的时间特别多，但每个 container 只给了 2 cores。把 cores 增加到 5 个后，问题没有了。也可以修改 json 对象转换的类，但感觉效率增加不会太多。

#### 2，有时候耗时多，有时候耗时少
是因为读取 kafka 的参数设置问题，下面有说明。


## 23，控制 executor 不自动增长
在 submit 作业时，设置 executor 的数量，但启动后 executor 实际数量比设置的多。这是因为 Spark On YARN 模式的 Spark Application 根据 Task 自动调整 Executor 数，要启用该功能，需做以下操作：
```
spark.dynamicAllocation.enabled true
spark.shuffle.service.enabled true
spark.dynamicAllocation.minExecutors 1 #最小Executor数 
spark.dynamicAllocation.maxExecutors 100 #最大Executor数 
```
> 有文章说还需要配置 yarn.nodemanager.aux-services.spark_shuffle.class，但在 yarn 2.6.0 上，这个配置好像被去掉了。

如果 spark 设置中，设置了可以自动增加的话，它可能就会增长。可两个办法让 executor 不自动增加。

方法1:
在命令行中增加参数：`--conf spark.dynamicAllocation.enabled=false`

方法2:
修改上面的操作，让自动调整不生效。

## 24，Job 延迟问题：平均处理时间为 1~2 秒，但偶尔有时候会增加 40 秒，变成 41~42 秒。
kafka 版本 0.10.2.1。调查了很多，最后问题是因为 kafka 参数设置问题。先说一下解决办法：
> session.timeout.ms 和 request.timeout.ms 时间设置成大于 batch 间隔时间。

```
consumerParams.put("request.timeout.ms", (batchDuration + 25) * 1000);
int sessionTimeout = (batchDuration + 20) * 1000;
consumerParams.put("session.timeout.ms", sessionTimeout);
consumerParams.put("heartbeat.interval.ms", sessionTimeout / 3);
```
个人设置的是 request.timeout.ms 比 batch 时间大 10 秒，session.timeout.ms 时间比 batch 时间大 5 秒，heartbeat.interval.ms 是 session.timeout.ms 的 1／3
> 其实主要想设置的是 session.timeout.ms，但 request.timeout.ms 一定要比 session.timeout.ms 大，所以也修改了 request.timeout.ms。heartbeat.interval.ms 这个不设置好像也行，没有进行测试。

### 调查过程
因为这个问题偶尔发生，所以不好定位问题原因。先是把业务代码都去掉，从 kafka 里读取数据后就 count，还会是发生，看来和业务没有关系。

后来在网上根据现象找答案，在 stack overflow 找到了一个问题，虽然没有人解答，但给了提示。问题 [scala - Strange delays in spark streaming - Stack Overflow](https://stackoverflow.com/questions/41720656/strange-delays-in-spark-streaming) 说到一点：
> request.timemout.ms 的默认值为 40s，修改后这个值为 10s 后，延迟也变成了 10s 左右。
> （注意，不同 kafka 版本这个值也不一样）

自己试了一下，果然是这样。看了一下 executor 的 log，在 batch 启动后，延迟了 40 秒才开始读 partition 里的数据。但解决办法不是把这个值改小，而是增大，超过 batch 时间。其实官方文档 [Spark Streaming + Kafka Integration Guide (Kafka broker version 0.10.0 or higher) - Spark 2.2.0 Documentation](https://spark.apache.org/docs/2.2.0/streaming-kafka-0-10-integration.html)
上也说过要修改这些属性的问题，但没有给出为什么要修改，修改成什么值，下面是官方文档上的一段引用：
> If your Spark batch duration is larger than the default Kafka heartbeat session timeout (30 seconds), increase heartbeat.interval.ms and session.timeout.ms appropriately. For batches larger than 5 minutes, this will require changing group.max.session.timeout.ms on the broker. 

### 问题原因
不太清楚问题的原因。查了一些资料，在 0.10.1 之后，heartbeat 就和 poll 分开了，有单独的线程去处理 heartbeat，为什么还要把 session.timeout.ms 设置比 batch 时间长呢？是不是 spark 做了什么操作呢？

这是一个参数设置的文章，以后有问题时候，可以看看：[Optimizing Spark Streaming applications reading data from Apache Kafka - Stratio Blog](https://www.stratio.com/blog/optimizing-spark-streaming-applications-apache-kafka/)

### 问题2
把 session.timeout 修改成大于 batchDuration 时间后，出现了新问题。有时候每分钟从 kafka 接收的记录数，是前一分钟的 2 倍或 3 倍的情况。而下一分钟的数据又非常非常少，例如：
> 18:19:00 400000
> 18:20:00 900000
> 18:21:00 80

原因是同时执行了两个 spark 程序，同时从不同的 topic 里消费数据，但 consumerGroup 确是一样的。结果是，第二个程序启动时，因为和第一个程序是同一个 consumerGroup，所以要重新分配 partition。第二个程序启动时发起 rebalance，它很快可以读到自己 topic 的数据，但第一个程序因为是同一个 consumerGroup，所以也需要进行 rebalance，虽然它读的 topic 和第二个程序不一样。在 rebalance 时，第一个程序的 consumer 发生了 session.timeout，在 session.timeout 时间后，又可以读取到记录。

因为时间差问题，第一个程序执行完 rebalace 之后，可能又会影响第二个程序，发生 rebalance。之后第二个程序又影响第一个程序，可能会反复进行。下面是 server.log 里的内容，会看到不断进行 rebalance。
```
# session.timout 时间是 1m50s


[2018-12-17 17:51:50,010] INFO [GroupCoordinator 1]: Assignment received from leader for group cloud_aggregation for generation 292 (kafka.coordinator.GroupCoordinator)

# 此处发生 session.timeout，从 52:00 开始，到 53:50。
# 以下都是因为两个程序互相影响，发生的 consumer rebalance。
[2018-12-17 17:52:00,003] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 292 (kafka.coordinator.GroupCoordinator)
[2018-12-17 17:53:50,004] INFO [GroupCoordinator 1]: Stabilized group cloud_aggregation generation 293 (kafka.coordinator.GroupCoordinator)
[2018-12-17 17:53:50,008] INFO [GroupCoordinator 1]: Assignment received from leader for group cloud_aggregation for generation 293 (kafka.coordinator.GroupCoordinator)

[2018-12-17 17:54:00,003] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 293 (kafka.coordinator.GroupCoordinator)
[2018-12-17 17:55:50,005] INFO [GroupCoordinator 1]: Stabilized group cloud_aggregation generation 294 (kafka.coordinator.GroupCoordinator)
[2018-12-17 17:55:50,009] INFO [GroupCoordinator 1]: Assignment received from leader for group cloud_aggregation for generation 294 (kafka.coordinator.GroupCoordinator)

[2018-12-17 17:56:00,004] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 294 (kafka.coordinator.GroupCoordinator)
[2018-12-17 17:57:50,005] INFO [GroupCoordinator 1]: Stabilized group cloud_aggregation generation 295 (kafka.coordinator.GroupCoordinator)
[2018-12-17 17:57:50,010] INFO [GroupCoordinator 1]: Assignment received from leader for group cloud_aggregation for generation 295 (kafka.coordinator.GroupCoordinator)
[2018-12-17 17:58:00,001] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 295 (kafka.coordinator.GroupCoordinator)
^@^@^@[2018-12-17 17:59:50,002] INFO [GroupCoordinator 1]: Stabilized group cloud_aggregation generation 296 (kafka.coordinator.GroupCoordinator)
[2018-12-17 17:59:50,007] INFO [GroupCoordinator 1]: Assignment received from leader for group cloud_aggregation for generation 296 (kafka.coordinator.GroupCoordinator)
^@[2018-12-17 18:00:00,004] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 296 (kafka.coordinator.GroupCoordinator)
^@^@^@[2018-12-17 18:01:50,017] INFO [GroupCoordinator 1]: Stabilized group cloud_aggregation generation 297 (kafka.coordinator.GroupCoordinator)
[2018-12-17 18:01:50,023] INFO [GroupCoordinator 1]: Assignment received from leader for group cloud_aggregation for generation 297 (kafka.coordinator.GroupCoordinator)
^@[2018-12-17 18:02:00,001] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 297 (kafka.coordinator.GroupCoordinator)




^@[2018-12-17 18:03:00,006] INFO [GroupCoordinator 1]: Stabilized group cloud_aggregation generation 298 (kafka.coordinator.GroupCoordinator)
[2018-12-17 18:03:00,011] INFO [GroupCoordinator 1]: Assignment received from leader for group cloud_aggregation for generation 298 (kafka.coordinator.GroupCoordinator)
^@^@^@^@[2018-12-17 18:05:02,697] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 298 (kafka.coordinator.GroupCoordinator)
^@^@^@[2018-12-17 18:06:52,699] INFO [GroupCoordinator 1]: Group cloud_aggregation with generation 299 is now empty (kafka.coordinator.GroupCoordinator)
^@[2018-12-17 18:07:00,003] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 299 (kafka.coordinator.GroupCoordinator)
[2018-12-17 18:07:00,004] INFO [GroupCoordinator 1]: Stabilized group cloud_aggregation generation 300 (kafka.coordinator.GroupCoordinator)
[2018-12-17 18:07:00,004] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 300 (kafka.coordinator.GroupCoordinator)
[2018-12-17 18:07:00,009] INFO [GroupCoordinator 1]: Stabilized group cloud_aggregation generation 301 (kafka.coordinator.GroupCoordinator)
[2018-12-17 18:07:00,014] INFO [GroupCoordinator 1]: Assignment received from leader for group cloud_aggregation for generation 301 (kafka.coordinator.GroupCoordinator)
^@[2018-12-17 18:07:50,002] INFO [GroupCoordinator 1]: Preparing to restabilize group cloud_aggregation with old generation 301 (kafka.coordinator.GroupCoordinator)
^@^@^@[2018-12-17 18:09:00,006] INFO [GroupCoordinator 1]: Stabilized group cloud_aggregation generation 302 (kafka.coordinator.GroupCoordinator)
[2018-12-17 18:09:00,009] INFO [GroupCoordinator 1]: Assignment received from leader for group cloud_aggregation for generation 302 (kafka.coordinator.GroupCoordinator)
```


## 25，spark streaming 从 kafka 里读取数据时，executor、cores 和 memory 应该如何设置？
### （1）partition 的读取
当从 kafka 里读取数据时，会根据 executor 和 cores 数量，来决定如何读取 partition。

executor | cores | partition | 读取方式
--- | --- | --- | ---
5 | 4 | 20 | 每个 executor 分配 4 个 partition，正好每个 core 读取一个 partition
5 | 2 | 20 | 每个 executor 分配 4 个 partition，2 个 core 先读取两个 partition。等这 2 个 partition 数据读取完成后，再读取另外 2 个 partition。 
3 | 3 | 20 | 2 个 executor 每个要读取 7 个 partition，1 个 executor 要读取 6 个 partition。读取时和上面一样，因为 core 数量和分配给 executor 的 partition 数量不是一比一对应，先读取一部分 partition，读完后再读取其它`未读取的 partition`。

所以当 executor 和 core 的数量和 partition 数量对应不上的话，会出现读取 kafka 数据不及时的问题。下面是一段 log：
```
18/11/20 18:14:00 INFO kafka010.KafkaRDD: Computing topic nginx, partition 17 offsets 12386586 -> 12411589
18/11/20 18:14:00 INFO kafka010.KafkaRDD: Computing topic nginx, partition 2 offsets 12386585 -> 12411589
18/11/20 18:14:00 INFO kafka010.KafkaRDD: Computing topic nginx, partition 11 offsets 12386585 -> 12411589
18/11/20 18:14:00 INFO kafka010.KafkaRDD: Computing topic nginx, partition 14 offsets 12386584 -> 12411588
18/11/20 18:14:00 INFO kafka010.KafkaRDD: Computing topic nginx, partition 5 offsets 12386585 -> 12411589
18/11/20 18:14:00 INFO kafka010.KafkaRDD: Computing topic nginx, partition 8 offsets 12386586 -> 12411589
18/11/20 18:14:03 INFO memory.MemoryStore: Block rdd_179_11 stored as bytes in memory (estimated size 16.1 MB, free 461.9 MB)
...............................
18/11/20 18:14:04 INFO executor.Executor: Finished task 19.0 in stage 436.0 (TID 3383). 1808 bytes result sent to driver
18/11/20 18:14:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 3385
18/11/20 18:14:06 INFO executor.Executor: Running task 14.0 in stage 436.0 (TID 3385)
18/11/20 18:14:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 3386
18/11/20 18:14:06 INFO executor.Executor: Running task 18.0 in stage 436.0 (TID 3386)
18/11/20 18:14:06 INFO kafka010.KafkaRDD: Computing topic nginx, partition 1 offsets 12373459 -> 12398462
18/11/20 18:14:06 INFO kafka010.CachedKafkaConsumer: Initial fetch for spark-executor-cloud_aggregation nginx 1 12373459
18/11/20 18:14:06 INFO kafka010.KafkaRDD: Computing topic nginx, partition 18 offsets 12386581 -> 12411584
```
从上面的 log 上可以看出，`partition 18`以外的 partition 是在 14:00 开始读取的，而`partition 18`是从 14:06 开始读取的。

所以，最好能平均分配 partition 的读取。如果不能平均分配，也最好不要出现：`一个 executor 读取 partition 数量` 比 `其它 exectuor 读取 partition 数量` 多 1 的现象，这样会让多数 executor 等待一个 executor，浪费资源。

### （2）partition 数量 和 executor、core 的数量对等
设置成上面表格中第一行的情况，正好每个线程消费一个 partition 的话也行，但有时候会出现出现`几秒`的 Scheduling delay。把 cores 加大，每个 executor 的 cores 数量比 partition 数量大 1 后，就没发生这种问题。

再有，读取时只能是一个 core，但处理时可以是多个 core，多 core 还是有好处的（前提是计算中用的 CPU 够多，如果 IO 操作多，core 多也没什么作用）。

### （3）partition 
**对于增大 partition**
会增加读取 kafka 的速度，但也会增加 core 的使用。如果增加 partition 不增加 core，等于没增长 partition。

**对于 executor、core 和 memeory**
是`executor 数量少，但 cores 和 memory 多`的方案好呢？还是`executor 数量多，但 cores 和 memory 少`的方案好呢？还在思考中。


## 26，接收器容错。从 kafka 中拉数据时，如何保证不丢失消息？如何保证不重复消费消息？
如果又想不丢、又不重复消费，基本上这个问题无解。因为这个问题涉及到一个`先保存 offset，还是先处理消息`的问题，因为先后顺序的不同，才会造成丢失和重复的问题。在找答案时找了一些文章，记录一下，以后有需要时候使用。

 - [Spark Streaming 读取 Kafka 的各种姿势解析](https://www.ctolib.com/topics-123320.html)
 - [Spark Streaming读取Kafka数据](https://www.jianshu.com/p/8603ba4be007)
 - [spark streaming读取kafka示例](https://blog.csdn.net/xzj9581/article/details/79223826)
 - [何管理Spark Streaming消费Kafka的偏移量（三）](http://qindongliang.iteye.com/blog/2401194)
 - [Spark streaming接收Kafka数据](https://blog.csdn.net/bsf5521/article/details/76635867?locationNum=9&fps=1)
 - [官方文档](https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)
 - [Spark Streaming 管理 Kafka Offsets 的方式探讨](https://juejin.im/entry/5acd7224f265da237c693f7d)
 - [为什么 Spark Streaming + Kafka 无法保证 exactly once？](https://www.jianshu.com/p/27f91de7417d)
 - [Spark Streaming重复消费,多次输出问题剖析与解决方案](https://my.oschina.net/jfld/blog/671189)：这个也解决不重复问题
 - [spark streaming 从kafka 拉数据如何保证数据不丢失](http://coolplayer.net/2016/11/30/spark-streaming-%E4%BB%8Ekafka-%E6%8B%89%E6%95%B0%E6%8D%AE%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E6%95%B0%E6%8D%AE%E4%B8%8D%E4%B8%A2%E5%A4%B1/)
 - [SparkStreaming如何优雅的停止服务](http://qindongliang.iteye.com/blog/2364713)
 - [Offset Management For Apache Kafka With Apache Spark Streaming](http://blog.cloudera.com/blog/2017/06/offset-management-for-apache-kafka-with-apache-spark-streaming/)
 - [Spark Streaming + Kafka Integration Guide](https://spark.apache.org/docs/2.2.0/streaming-kafka-0-10-integration.html#storing-offsets)：官方的方法，不错。

## 27，如何优雅关闭，不丢失消息
- [Spark Streaming优雅的关闭策略优化](http://qindongliang.iteye.com/blog/2404100)
- [Spark streaming 设计与实现剖析](https://mp.weixin.qq.com/s?__biz=MzI3MjY2MTYzMA==&mid=2247483758&idx=1&sn=acd78535a2398f7109087256f3a06b15&scene=21#wechat_redirect)


## 28，如何删除 spark application history
CDH 修改 spark-defaults.conf 文件，添加以下配置：
```
spark.history.fs.cleaner.enabled true
spark.history.fs.cleaner.maxAge  12h
spark.history.fs.cleaner.interval 1h
```

但删除的比较慢。

参考：
- [Spark History Server Automatic Cleanup – Hadoopsters](https://hadoopsters.net/2016/10/27/spark-history-server-automatic-cleanup/)
- [Cleaning up Spark history logs - Stack Overflow](https://stackoverflow.com/questions/42817924/cleaning-up-spark-history-logs)
