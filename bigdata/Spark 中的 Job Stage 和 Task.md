# 前提
>正在做 Spark 程序，但对 Job、Stage、Task 等还不算了解，上网找了一些文章，自己把这些文章中的东西总结一下，方便自己记忆。


什么是 Job，Stage 和 Task 呢？单单进行说明，不是很好理解，下面根据程序进行一下说明。程序如下：

**1，程序功能**
输入的字符串，按空格进行切割，然后统计每个单词的个数。也就是经典的`WordCounting`。

**2，程序流程**
- 1，输出字符串
- 2，进行按空格切割，生成 pair，最后进行统计每个单词数量
- 3，收集然后进行输出

**3，程序代码**
```

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import scala.Tuple2;

import java.util.Arrays;

public class HelloWorld {
    public static void main(String[] args) {

        SparkConf conf = new SparkConf().setAppName("HelloWorld");
        JavaSparkContext sc = new JavaSparkContext(conf);
        // 1，输出字符串
        JavaRDD<String> lines = sc.parallelize(Arrays.asList("pandas", "i like pandas"));
        // 2，进行按空格切割，生成 pair，最后进行统计每个单词数量
        JavaPairRDD<String, Integer> wordsPairs = lines
                .flatMap(
                        line -> Arrays.asList(line.split(" ")).iterator())
                .mapToPair(
                        word -> new Tuple2<String, Integer>(word, 1))
                .reduceByKey(
                        (x, y) -> x + y);
        // 3，收集然后进行输出
        wordsPairs.foreach(wordPair -> System.out.println(wordPair));

    }
}
```

# 一、关于 Job？
## 1，Job 是什么？
可以认为是 Spark RDD 里面的 action，每个 action 会生成一个 job 。在上面的代码中只有一个 action：
> wordsPairs.foreach();

如果我们在加一个`wordsPairs.foreach()`，那么就会有两个 action，也就是会有两个 job。

> 注意：foreachRDD 也是 action。 

## 2，Job 在哪里可以看到
在 Spark WebUI 的 jobs 栏里可以看到，每一个 Job 都是一行。如下图：
![在这里插入图片描述](https://img-blog.csdn.net/20181022173726213?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



## 二、关于 Stage
### 1，什么是 stage？
stage 是从 job 划分出来的。每个 job 可能会划分成多个 stage，这些 stage 按一定顺序执行。例子中，会划分成两个 stage：
- stage 1：`1，输出字符串`和`2，进行按空格切割，生成 pair，最后进行统计每个单词数量`，会被划分成 stage 1。
- stage 2：`3，收集然后进行输出`会被划分成 stage 2。

为什么会被划分成 2 个 stage 呢？在这前，我们需要了解一下关于 RDD 的一些知识。

### 2，关于 RDD 的依赖
RDD 之间是有依赖关系的，每个 RDD 可能是通过其它 RDD 生成的。例如：代码中的 wordPairs RDD 就是通过 lines 生成的。
 
RDD 依赖的分类主要分为两类：
- 窄依赖（也叫narrow依赖）
- 宽依赖（也叫shuffle依赖/wide依赖）

**窄依赖（也叫narrow依赖）**
窄依赖指父RDD的每一个分区最多被一个子 RDD 的分区所用，表现为：（分别对应下图左部的上、下两个图例）
- 一个父 RDD 的分区对应于一个子 RDD 的分区
- 两个父 RDD 的分区对应于一个子 RDD 的分区。

map、filter、union 都属于窄依赖。

**宽依赖（也叫shuffle依赖/wide依赖）**
从父RDD角度看：一个父 RDD 被多个子 RDD 分区使用。父RDD的每个分区可以被多个子 RDD分区依赖。

从子RDD角度看：依赖上级RDD的所有分区，无法精确定位依赖的父 RDD 分区，相当于依赖所有父分区（例如reduceByKey）。如下图右边部分。

groupByKey 等属于宽依赖。
> 宽依赖都需要“混洗”（shuffle），所以也叫 shuffle 依赖

![在这里插入图片描述](https://img-blog.csdn.net/20181022174742389?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 3，为什么要分成`窄依赖`和`宽依赖`
Spark之所以将依赖分为`窄依赖`和`宽依赖`：

(1) `窄依赖`可以支持在同一个集群Executor上，以pipeline管道形式顺序执行多条命令，例如在执行了map后，紧接着执行filter。分区内的计算收敛，不需要依赖所有分区的数据，可以并行地在不同节点进行计算。所以它的失败恢复也更有效，因为它只需要重新计算丢失的parent partition即可，

(2) `宽依赖` 则需要所有的父分区都是可用的，必须等RDD的parent partition数据全部ready之后才能开始计算，可能还需要调用类似MapReduce之类的操作进行跨节点传递。从失败恢复的角度看，`宽依赖` 牵涉RDD各级的多个parent partition。


### 4，如何划分 stage
#### （1）划分 stage 的方法
先根据` transformation 操作` 还是 `action 操作`划分。对于` transformation 操作`，再根据`窄依赖`还是`宽依赖`再进行划分。
> DAGScheduler: 根据Job构建基于Stage的DAG（Directed Acyclic Graph有向无环图)，并提交Stage给TaskScheduler。

总结下：
- 对于 transformation 操作，以`宽依赖`为分隔，分为不同的 Stages。宽依赖会划分到第后面的 stage 里。
  * 窄依赖：tasks会归并在同一个stage中，（相同节点上的task运算可以像pipeline一样顺序执行，不同节点并行计算，互不影响）
  * 宽依赖：前后拆分为两个stage，前一个stage写完文件后下一个stage才能开始
- 对于 action 操作：和其他tasks会归并在同一个stage（在没有shuffle依赖的情况下，生成默认的stage，保证至少一个stage）。

#### （2）划分结果
所以上面的程序中的过程，可以分成两个 stage ：
- stage 0：parallelize、flatMap 和 mapToPair 。这些都是`窄依赖`
- stage 1：reduceByKey 和 foreach。reduceByKey 是`宽依赖`所以分到第后面的 stage 里，因为 foreach 前有 reduceByKey 的 stage，所以放到这个 stage 里。

首先 Stage0 进行了shuffle write。如果设置了 partition，那么就会根据 parition 的数量生成 task，task 互相独立，并不需要依赖彼此做完或者怎样，所以他们在一个stage里面并发执行。
然后 stage1 是依赖之前的 stage0 完成 shuffle 的，reduceByKey开始需要 ShuffleRead stage0的计算结果。

具体例子参考：[Spark基础入门（二）--------DAG与RDD依赖](https://blog.csdn.net/silviakafka/article/details/54574653)。这个文章中的说明和小例子都很好。本文很多都是 copy 这个文章。


#### （3）其它细节
**1）依赖划分**
实际应用提交的Job中RDD依赖关系是十分复杂的，依据这些依赖关系来划分stage自然是十分困难的，Spark此时就利用了前文提到的依赖关系，调度器从DAG图末端出发，逆向遍历整个依赖关系链，遇到`宽依赖`就断开，遇到`窄依赖`就将其加入到当前stage。stage中task数目由stage末端的RDD分区个数来决定，RDD转换是基于分区的一种粗粒度计算，一个stage执行的结果就是这几个分区构成的RDD。

**2）依赖数据存储**
另外，由于`宽依赖`必须等 RDD 的 `父 RDD partition 数据`全部准备好之后才能开始计算，因此 spark 的设计是让`父 RDD`将结果写在本地，完全写完之后，通知后面的RDD。后面的RDD则首先去读之前的本地数据作为input，然后进行运算。

由于上述特性，将shuffle依赖就必须分为两个阶段(stage)去做：
- 第一个阶段(stage)需要把结果shuffle到本地，例如reduceByKey，首先要聚合某个key的所有记录，才能进行下一步的reduce计算，这个汇聚的过程就是shuffle
- 第二个阶段(stage)则读入数据进行处理。

同一个 stage 里面的 task 是可以并发执行的，下一个 stage 要等前一个 stage 都做完。（和mapreduce的reduce需要等map过程是一样的）

**为什么要写在本地**
后面的RDD多个partition都要去读这个信息，如果放到内存，如果出现数据丢失，后面的所有步骤全部不能进行，违背了之前所说的需要parent RDD partition数据全部ready的原则。为什么要保证parent RDD要ready，如下例，如果有一个partition未生成或者在内存中丢失，那么直接导致计算结果是完全错误的：

写到文件中更加可靠。Shuffle会生成大量临时文件，以免错误时重新计算，其使用的本地磁盘目录由spark.local.dir指定，缓存到磁盘的RDD数据。最好将这个属性设定为访问速度快的本地磁盘。可以配置多个路径到多个磁盘，增加IO带宽

在Spark 1.0 以后，SPARK_LOCAL_DIRS(Standalone, Mesos) or LOCAL_DIRS (YARN)参数会覆盖这个配置。比如Spark On YARN的时候，Spark Executor的本地路径依赖于Yarn的配置，而不取决于这个参数。

## 三，关于 Task
Task 就是被送到 executor 上的工作单元。每个Job会被拆分成`多组Task`， 作为一个TaskSet， 也就是 Stage。

一般来说，一个 rdd 有多少个 partition，就会有多少个 task，因为每一个 task 只是处理一个 partition 上的数据。Task简单的说，就是在一个数据 partition 上的单个数据处理流程。 例如：

> TASKSedulter: 将TaskSET提交给worker运行，每个Executor运行什么Task就是在此处分配的。TaskScheduler维护所有TaskSet，当Executor向Driver发生心跳时，TaskScheduler会根据资源剩余情况分配相应的Task。另外TaskScheduler还维护着所有Task的运行标签，重试失败的Task。

 Spark上分为2类task。
 - ShuffleMapTask：根据`宽依赖`中的`分区算法（partition）`，把 RDD 中的数据分到不同的分区上的 Task。
 - ResultTask：把结果发回给 Driver 的 Task。


## 四、关于 Partition
在从 stage 划分 task 时，是根据 partition 的数量分成多少个 task。这些 task 的都是同类型的 task，也就是说处理逻辑是相同的。分成多个 task 的目的是处理不同的数据，不同数据的划分就是根据 partition 来划分的。

我们可以指定 `分区数量`和`具体的分区实现`来进行分区：
- 指定`分区数量`：noParLines.repartition(2)
- `具体的分区实现`：省略。

**但是，当我们使用上面的方式指定分区时，分区数量是多少呢？**
spark 是从`spark.default.parallelism`这个配置具体的分区数量。但这个值是如何设置的呢？
如果配置文件spark-default.conf中没有显示的配置，则按照如下规则取值：
- 本地模式（不会启动executor，由SparkSubmit进程生成指定数量的线程数来并发）：
  * spark-shell                              spark.default.parallelism = 1
  * spark-shell --master local[N] spark.default.parallelism = N （使用N个核）
  * spark-shell --master local      spark.default.parallelism = 1
- 伪集群模式（x为本机上启动的executor数，y为每个executor使用的core数，z为每个 executor使用的内存）
  * spark-shell --master local-cluster[x,y,z] spark.default.parallelism = x * y
- mesos 细粒度模式
  * Mesos fine grained mode  spark.default.parallelism = 8
- 其他模式（这里主要指yarn模式，当然standalone也是如此）
  * Others: total number of cores on all executor nodes or 2, whichever is larger。 也就是 spark.default.parallelism =  max（所有executor使用的core总数， 2）


**Partition数量影响及调整**
上面分析了决定Partition数量的因数，接下来就该考虑Partition数量的影响以及合适的值。
- Partition数量的影响
  * Partition数量太少。太少的影响显而易见，就是资源不能充分利用，例如local模式下，有16core，但是Partition数量仅为8的话，有一半的core没利用到。
  * Partition数量太多。太多，资源利用没什么问题，但是导致task过多，task的序列化和传输的时间开销增大。
  * 
那么多少的partition数是合适的呢，这里我们参考spark doc给出的建议，[Typically you want 2-4 partitions for each CPU in your cluster](http://spark.apache.org/docs/latest/rdd-programming-guide.html)。

Partition调整 
- repartition：reparation是coalesce(numPartitions, shuffle = true)，repartition不仅会调整Partition数，也会将Partitioner修改为hashPartitioner，产生shuffle操作。
- coalesce：coalesce函数可以控制是否shuffle，但当shuffle为false时，只能减小Partition数，无法增大。

参考：
- [Spark RDD的默认分区数：（spark 2.1.0）](https://www.jianshu.com/p/4b7d07e754fa)
- [Spark RDD之Partition](https://blog.csdn.net/u011564172/article/details/53611109)




# 参考
- [Spark基础入门（二）--------DAG与RDD依赖](https://blog.csdn.net/silviakafka/article/details/54574653)：本文很多内容都 copy 这上面的，讲的很好。
- [Spark中job、stage、task的划分+源码执行过程分析](https://blog.csdn.net/hjw199089/article/details/77938688)：这个也很不错，有具体的例子，还有代码、WebUI、Log 的对应关系的说明。
- [Spark DAG之划分Stage](https://blog.csdn.net/u011564172/article/details/70172178)：也是一个例子，没有上面两个好，但是有多个 RDD 的`宽依赖`，也有可参考的地方。
- [『 Spark 』6. 深入研究 spark 运行原理之 job, stage, task](http://litaotao.github.io/deep-into-spark-exection-model)
- [Spark作业调度中stage的划分](https://wongxingjun.github.io/2015/05/25/Spark%E4%BD%9C%E4%B8%9A%E8%B0%83%E5%BA%A6%E4%B8%ADstage%E7%9A%84%E5%88%92%E5%88%86/)

关于 partition：
- [Spark RDD的默认分区数：（spark 2.1.0）](https://www.jianshu.com/p/4b7d07e754fa)
- [Spark RDD之Partition](https://blog.csdn.net/u011564172/article/details/53611109)

