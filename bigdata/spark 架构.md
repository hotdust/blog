# Spark
## 一、Spark 与 hadoop
Hadoop有两个核心模块，分布式存储模块HDFS和分布式计算模块Mapreduce。spark本身并没有提供分布式文件系统，因此spark的分析大多依赖于Hadoop的分布式文件系统HDFS。Hadoop的Mapreduce与spark都可以进行数据计算，而相比于Mapreduce，spark的速度更快并且提供的功能更加丰富

## 二、Spark 名词
- Application：Spark中的Application和Hadoop MapReduce中的概念是相似的，指的是用户编写的Spark应用程序，内含了一个Driver功能的代码和分布在集群中多个节点上运行的Executor代码
- Driver Program：Spark中的Driver即运行上述Application的main()函数并且创建SparkContext，其中创建SparkContext的目的是为了准备Spark应用程序的运行环境。在Spark中由SparkContext负责和ClusterManager通信，进行资源的申请、任务的分配和监控等；当Executor部分运行完毕后，Driver负责将SparkContext关闭。通常用SparkContext代表Driver
- Executor：Application运行在Worker节点上的一个进程，该进程负责运行Task，并且负责将数据存在内存或者磁盘上，每个Application都有各自独立的一批Executor。在Spark
 on Yarn模式下，其进程名称为CoarseGrainedExecutorBackend，类似于Hadoop MapReduce中的YarnChild。一个CoarseGrainedExecutorBackend进程有且仅有一个executor对象，它负责将Task包装成taskRunner，并从线程池中抽取出一个空闲线程运行Task。每个CoarseGrainedExecutorBackend 能并行运行Task的数量就取决于分配给它的CPU的个数了
       
- Cluster Mananger：指的是在集群上获取资源的外部服务，目前有：
  * Standalone：Spark原生的资源管理，由Master负责资源的分配；
  * Hadoop Yarn：由YARN中的ResourceManager负责资源的分配；
       
- Worker：集群中任何可以运行Application代码的节点，类似于YARN中的NodeManager节点。在Standalone模式中指的就是通过Slave文件配置的Worker节点，在Spark
 on Yarn模式中指的就是NodeManager节点     
- Job：包含多个Task组成的并行计算，往往由Spark Action催生，一个JOB包含多个RDD及作用于相应RDD上的各种Operation
- Stage：每个Job会被拆分很多组Task，每组任务被称为Stage，也可称TaskSet，一个作业分为多个阶段     
- Task：被送到某个Executor上的工作任务

## 三、Spark 架构
spark 架构都是 Driver 通过 Cluster Manager 申请资源（Executor），然后 Driver 直接和 Executor 进行通信，Driver 分配 Task 给 Exectuor 去执行。

例如，下图就是前面说明的流程。SparkContext 就是 Driver，Driver 先去资源管理器（也就是 Cluster Manager）申请资源。Cluster Manager 分配它所管理的资源。然后资源直接和 Driver 进行通信，向 Driver 注册并申请 Task 进行执行。

![在这里插入图片描述](https://img-blog.csdn.net/20181010091402575?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

<br>
Cluster Manager 的主要作用就是分配资源，分配完资源后，资源就直接和 Driver 进行工作。这种架构的好处是，Cluster Manager 是可以替换的，可以使用 Spark 自带的 Cluster Manager、Yarn、Mesos 等 Cluster Manager。下图就是架构的概括图，根据不同的 Cluster，各个部分的名称也不一样，稍后介绍每个 Cluster 。

![在这里插入图片描述](https://img-blog.csdn.net/20181010091821325?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
这种架构的特点是：
- 每个Application获取专属的executor进程，该进程在Application期间一直驻留，并以多线程方式运行Task。这种Application隔离机制是有优势的，无论是从调度角度看（每个Driver调度他自己的任务），还是从运行角度看（来自不同Application的Task运行在不同JVM中），当然这样意味着Spark Application不能跨应用程序共享数据，除非将数据写入外部存储系统。
- Spark与资源管理器无关，只要能够获取executor进程，并能保持相互通信就可以了
提交SparkContext的Client应该靠近Worker节点（运行Executor的节点），最好是在同一个Rack里，因为Spark Application运行过程中SparkContext和Executor之间有大量的信息交换。
- Task采用了数据本地性和推测执行的优化机制

### 1，Spark standalone
Standalone模式使用Spark自带的资源调度框架，采用Master/Slaves的典型架构，选用ZooKeeper来实现Master的HA。框架结构图如下:
![在这里插入图片描述](https://img-blog.csdn.net/2018101009370364?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


独立集群管理器支持两种部署模式。在这两种模式中，应用的驱动器程序运行在不 同的地方。
- 在客户端模式中(默认情况)，驱动器程序会运行在你执行 spark-submit 的机 器上，是 spark-submit 命令的一部分。这意味着你可以直接看到驱动器程序的输出，也 可以直接输入数据进去(通过交互式 shell)，但是这要求你提交应用的机器与工作节点间有很快的网络速度，并且在程序运行的过程中始终可用。
- 在集群模式下，驱动器程序会作为某个工作节点上一个独立的进程运行在独立集群管理器内部。它也会连接主节点来申请执行器节点。在这种模式下，spark-submit 是“一劳永逸”型，你可以在应用运行时关掉你的电脑。

你还可以通过集群管理器的网页用户界面访问应用的日志。向 spark- submit 传递 --deploy-mode cluster 参数可以切换到集群模式。

这种架构的具体的执行流程如下：
![在这里插入图片描述](https://img-blog.csdn.net/20181010094652570?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


1. SparkContext连接到Master，向Master注册并申请资源（CPU Core 和Memory）
2. Master根据SparkContext的资源申请要求和Worker心跳周期内报告的信息决定在哪个Worker上分配资源，然后在该Worker上获取资源，然后启动StandaloneExecutorBackend；
3. StandaloneExecutorBackend向SparkContext注册；
4. SparkContext将Applicaiton代码发送给StandaloneExecutorBackend；并且SparkContext解析Applicaiton代码，构建DAG图，并提交给DAG Scheduler分解成Stage（当碰到Action操作时，就会催生Job；每个Job中含有1个或多个Stage，Stage一般在获取外部数据和shuffle之前产生），然后以Stage（或者称为TaskSet）提交给Task Scheduler，Task Scheduler负责将Task分配到相应的Worker，最后提交给StandaloneExecutorBackend执行；
5. StandaloneExecutorBackend会建立Executor线程池，开始执行Task，并向SparkContext报告，直至Task完成
6. 所有Task完成后，SparkContext向Master注销，释放资源

### Spark on Yarn
Spark on YARN模式根据Driver在集群中的位置分为两种模式：一种是YARN-Client模式，另一种是YARN-Cluster（或称为YARN-Standalone模式）。Yarn-Client模式中，Driver在客户端本地运行，这种模式可以使得Spark Application和客户端进行交互，因为Driver在客户端，所以可以通过webUI访问Driver的状态，默认是http://hadoop1:4040访问，而YARN通过http:// hadoop1:8088访问。

#### YARN-client
![在这里插入图片描述](https://img-blog.csdn.net/20181010094958248?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

1. Spark Yarn Client向YARN的ResourceManager申请启动Application Master。同时在SparkContent初始化中将创建DAGScheduler和TASKScheduler等，由于我们选择的是Yarn-Client模式，程序会选择YarnClientClusterScheduler和YarnClientSchedulerBackend
2. ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，与YARN-Cluster区别的是在该ApplicationMaster不运行SparkContext，只与SparkContext进行联系进行资源的分派
3. Client中的SparkContext初始化完毕后，与ApplicationMaster建立通讯，向ResourceManager注册，根据任务信息向ResourceManager申请资源（Container）
一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向Client中的SparkContext注册并申请Task
4. client中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向Driver汇报运行的状态和进度，以让Client随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务
5. 应用程序运行完成后，Client的SparkContext向ResourceManager申请注销并关闭自己

#### YARN-Cluster
在YARN-Cluster模式中，当用户向YARN中提交一个应用程序后，YARN将分两个阶段运行该应用程序：
- 第一个阶段是把Spark的Driver作为一个ApplicationMaster在YARN集群中先启动；
- 第二个阶段是由ApplicationMaster创建应用程序，然后为它向ResourceManager申请资源，并启动Executor来运行Task，同时监控它的整个运行过程，直到运行完成

![在这里插入图片描述](https://img-blog.csdn.net/20181010095845120?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hvdGR1c3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


1. Spark Yarn Client向YARN中提交应用程序，包括ApplicationMaster程序、启动ApplicationMaster的命令、需要在Executor中运行的程序等。
2. ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，其中ApplicationMaster进行SparkContext等的初始化。
3. ApplicationMaster向ResourceManager注册，这样用户可以直接通过ResourceManage查看应用程序的运行状态，然后它将采用轮询的方式通过RPC协议为各个任务申请资源，并监控它们的运行状态直到运行结束。
4. 一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向ApplicationMaster中的SparkContext注册并申请Task。这一点和Standalone模式一样，只不过SparkContext在Spark Application中初始化时，使用CoarseGrainedSchedulerBackend配合YarnClusterScheduler进行任务的调度，其中YarnClusterScheduler只是对TaskSchedulerImpl的一个简单包装，增加了对Executor的等待逻辑等
5. ApplicationMaster中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向ApplicationMaster汇报运行的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务。
6. 应用程序运行完成后，ApplicationMaster向ResourceManager申请注销并关闭自己。

#### Yarn Client 和 Yarn Cluster 区别
理解YARN-Client和YARN-Cluster深层次的区别之前先清楚一个概念：Application Master。在YARN中，每个Application实例都有一个ApplicationMaster进程，它是Application启动的第一个容器。它负责和ResourceManager打交道并请求资源，获取资源之后告诉NodeManager为其启动Container。从深层次的含义讲YARN-Cluster和YARN-Client模式的区别其实就是ApplicationMaster进程的区别。
- YARN-Cluster模式下，Driver运行在AM(Application Master)中，它负责向YARN申请资源，并监督作业的运行状况。当用户提交了作业之后，就可以关掉Client，作业会继续在YARN上运行，因而YARN-Cluster模式不适合运行交互类型的作业。
- YARN-Client模式下，Application Master仅仅向YARN请求Executor，Client会和请求的Container通信来调度他们工作，也就是说Client不能离开。

### Spark Standalone 和 Spark on Yarn 区别
相同点：
- Client 模式下，Driver 在启动命令的本机上。
- Cluster 模式下，Driver 是在 Cluster Manager 管理的结点上（Spark 的 Worker 或 Yarn 的 Node Manager）。

不同点：
- Cluster Manager 和 资源结点：
  * Spark Standalone 架构上，Cluster Manager 叫做 `Master`，资源结点叫做`Worker`。
  * Spark on Yarn 架构上，Cluster Manager 叫做 Resource Manager，资源结点叫做`NodeManager`。



参考：
[Spark(一): 基本架构及原理](https://www.cnblogs.com/tgzhu/p/5818374.html)：这个最全，把 spark 架构、standalone、spark on yarn 架构结构都说了。
[Spark的运行架构分析（一）之架构概述](https://blog.csdn.net/Gamer_gyt/article/details/51822765)：第一篇文章拿出来一部分，加上自己的理解和讲解。
[深入理解spark之架构与原理](https://my.oschina.net/sunzy/blog/1617071)：第一篇文章拿出来一部分，加上自己的理解和讲解。

