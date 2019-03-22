# 综合：
[大数据学习笔记](https://chu888chu888.gitbooks.io/hadoopstudy/content/Content/3/chapter0301.html)：综合的入门级的，容易明白
[有态度的HBase/Spark/BigData](http://hbasefly.com/)：主讲 HBase、Spark、时序数据库，讲的非常容易明白，非常原理性非常强。


## 入门
### 1，Hadoop 架构
NameNode、DataNode、 Secondary NameNode、JobTracker、TaskTracker
其中NameNode、Secondary、NameNode、JobTracker运行在Master节点上，而在每个Slave节点上，部署一个DataNode和TaskTracker，以便 这个Slave服务器运行的数据处理程序能尽可能直接处理本机的数据。

2，YARN 和 Hadoop 的关系
YARN 是 Hadoop 2 开始使用的资源管理框架，Hadoop 1 的资源管理功能集成在 MapReduce 逻辑内部（资源管理只能为 MapReduce 作业服务），通常称第一代 MapReduce 为 “MRv1”，可以说，YARN 是 MRv1 的一部分功能 —— 主要职责是取代 MRv1 中 JobTracker 实现资源调度和作业管理分离，更加科学和高效地利用计算资源，有点像 Mesos 分布式资源管理系统。其中，作业管理由 ApplicationMaster （AM）实现，而 YARN 的资源管理方面，提供了计算资源管理和分配功能，对内存和磁盘等资源可以有效地管理。 它并不是下一代 MapReduce（MRv2），YARN 姑且可以简单地认为是 MRv1 中一部分功能的代替者，但这样表述显然让 YARN 使命大大缩水，实际上，YARN 是独立存在的，具有通用性，不仅可以调度 MapReduce 作业，还可以作为其他计算框架的资源管理框架，如 Spark、Storm 等可以跑在 YARN 上，它们通过 YARN 来管理计算资源，计算任务有了稳定的平台支撑，可以保证性能和稳定性，同时，这些计算框架可以方便地读取 HDFS 上的数据；

3，YARN 工作流程和组件
[简述hadoop 2.x Yarn组件协作过程](https://segmentfault.com/a/1190000012820966)
[初步掌握Yarn的架构及原理](http://www.cnblogs.com/codeOfLife/p/5492740.html)


4，YARN 配置
[YARN架构设计详解](http://www.cnblogs.com/wcwen1990/p/6737985.html)




#InfluxDB:
基础：

- [InfluxDB简明手册 ](https://www.bookstack.cn/read/influxdb-handbook/README.md)：挺好的简单入门
- [InfluxDB使用](https://kiswo.com/article/1020)
- [官方英文文档](https://docs.influxdata.com/influxdb/v1.6/introduction/getting-started/)
- [官方文档中文翻译](https://jasper-zhang1.gitbooks.io/influxdb/content/)：翻译的怎么样还没有看。
- [InfluxDB Line Protocol tutorial](https://docs.influxdata.com/influxdb/v1.6/write_protocols/line_protocol_tutorial/)
- [InfluxDB frequently asked questions](https://docs.influxdata.com/influxdb/v1.6/troubleshooting/frequently-asked-questions)

原理：

- [时序数据库技术体系 – 初识InfluxDB](http://hbasefly.com/2017/12/08/influxdb-1/)
- [InfluxDB详解之TSM存储引擎解析（一）](http://blog.fatedier.com/2016/08/05/detailed-in-influxdb-tsm-storage-engine-one/)
- [InfluxDB详解之TSM存储引擎解析（二）](http://blog.fatedier.com/2016/08/15/detailed-in-influxdb-tsm-storage-engine-two/)
- [LSM Tree 学习笔记](http://blog.fatedier.com/2016/06/15/learn-lsm-tree/)：InfluxDB 的结构和 LSM Tree 结构很像
- [时序数据库](http://hbasefly.com/category/%E6%97%B6%E5%BA%8F%E6%95%B0%E6%8D%AE%E5%BA%93/)：关于时序数据库的很多文章，还有原理性的文章，可以和上面的文章结合起来看。

经验：
1，关于 field 的类型
field 是有类型的，如果类型是 int 的话，插入 float 就会报错，反之亦然。在插入数据时，插入一个数值字段，例如：cnt=2，默认这个这个 cnt 类型为 float。如果想指定这个 field 的类型为 int 的话，就要在后面加一个小写字母`i`，例如：cnt=2i。
官方解释：https://docs.influxdata.com/influxdb/v1.6/troubleshooting/frequently-asked-questions/#how-do-i-write-integer-field-values


2，关于插入时间格式的问题
在插入数据时，可以使用两种方式插入数据：json 和 line。使用 json 格式的话，timestamp 指定为`2009-11-10T23:00:00.560`这种格式没有问题。但使用 line 方式插入数据的话，使用这种格式就会报格式错误。可能在 influxDB 内部对 json 格式的时间有自动转换吧，但 json 格式插入方式已经`不建议使用`了。



3，关于数据的修改
InfluxDB 是无法做删除操作的，但还是可以做修改操作的。修改方法就是通过`time` 和 `Tags`，如果插入的数据的`time和Tags 的值`在数据库中已经存在，就会替换已存在的数据。
官方解释：https://docs.influxdata.com/influxdb/v1.6/troubleshooting/frequently-asked-questions/#how-does-influxdb-handle-duplicate-points



# CDH
## 资料

- [CDH 5.8.x 中文文档](http://cwiki.apachecn.org/pages/viewpage.action?pageId=2162815)
- [CDH 5.5.x 英文文档](https://www.cloudera.com/documentation/enterprise/5-5-x/topics/cm_intro_primer.html)
- 
## 经验

### 1，关于目录结构
- 组件的 bin 文件都在`/opt/cloudera/parcels/CDH/lib/spark/bin/` 目录下。目录中的 CDH 目录，是同级带版本号的 CDH_XXX 目录的软连接。
- 组件的配置文件都在`/etc/xxx/conf` 目录下。
- XXX_HOME：使用 parcel 方式安装 SPARK_HOME 默认在/opt/cloudera/parcels/CDH/lib/spark

### 2，安装 CDH 的要求。
https://cloud.tencent.com/developer/article/1078212

### 3，CDH 配置
[CDH安装前置准备](https://cloud.tencent.com/developer/article/1078212)

### 4，关于 CDH 的 gateway
gateway 有一种功能是，在 CM 修改集群信息后，同步到有 gateway 的机器上。gateway 不需要启动，只要机上上有这个 role 就可以。例如：有两台机器，一台机器上有 spark history server，另一台机器有 spark gateway，在上面的“集群”菜单选 spark，然后点“配置”进行修改的话，两台机器的配置都会以相同的方式修改。但如果另一台机器上没有 spark gateway，那修改的配置只会反映到有 spark history server 的机器上。
 
再有，一般来说只有一台机器上装 spark history server，如果点“集群”，在“状态摘要”中点击“History Server”，再点“配置”的话，会提示“此服务中的此角色只有一个实例。我们建议您在服务配置页上更改配置。”。如果点“取消”，就会继续在“History Server”的配置画面进行配置；
如果点“服务配置”，就会跳转到“点击集群后，再点配置的画面”，这个画面是 spark 服务整体的配置。如果你点击“取消”的进行修改配置的话，这个配置不会反应到任何一个机器上。本以为应该
反映到“History Server”的那台机器上，结果那台机器上也没有。所以，如果要 Spark 所有相同机器配置都被修改，就要到 spark 服务整体配置画面去修改配置。
> 上面的测试方法是，在 CM 画面中修改配置中“spark-env.sh”文件内容，然后重启看看是否 spark gateway 和 spark history server 中“spark-env.sh”文件内容被修改了。

### 5，在 CM 画面修改完组件配置后，不生效
在 CM 画面修改完组件配置后，并不是立刻生效的。因为修改是修改的数据库里的数据，如果进行重启后，才会把数据库里的数据反映到 client 端。所以，在修改配置并重启后，在重启画面中，可以看到 HDFS、YARN、HIVE 等相关的 Client 配置被发布到 Client 端。

### 6，如果安装时，log 说 jdk 版本不对，要修改主机的 jdk 配置。
原因是，Cloudera Manager 和 CDH processes 和 CM service role 使用的 JVM 是不一样的。Cloudera Manager 是机器启动时的环境变量，而 CDH processes 和 CM service role 使用的是各种组件 env 里的。

修改方法：
在 cloudera-manager 管理画面，选择`主机->所有主机`，再选择右边的`配置`按钮。在搜索里搜索 java，再`java 主目录`的输入框里输入 JDK 的路径，例如：`/usr/java/jdk1.8.0_181-amd64`


参考：
- [cloudera manager 升级到jdk1.8](https://blog.csdn.net/chenguangchun1993/article/details/78903463)

- [This change affects all CDH processes and Cloudera Management Service roles in the cluster.](
https://www.cloudera.com/documentation/enterprise/5-14-x/topics/cdh_ig_jdk_installation.html)

error log
```
PATH_TO_JAVA=/usr/java/jdk1.7.0_79/bin/java
[[ 1.7 != \1\.\8 ]]
```

## 7，使用 kafka_10 的话，要在 spark 配置里修改 kafka 版本
进入CDH的spark2配置界面，在搜索框中输入SPARK_KAFKA_VERSION，出现如下图，然后选择对应版本，这里我应该选择的是0.10，然后保存配置，重启生效。重新跑sparkstreaming任务，问题解决。
参考：https://blog.csdn.net/u010936936/article/details/77247075

## 8，CDH 安装 spark2
1，按下面的步骤进行安装。
注意：要把下载文件后缀`sha1`改成`sha`，这点下面的文章上都没有。

- [CDH5.12.1安装spark2.2](http://blog.xumingxiang.com/292.html)
- [CDH 5.13安装spark2](https://www.jianshu.com/p/6acd6419f697)：和上面的文章很像，有一些问题的处理。

2，修改主机 JDK 版本。
修改方法：
在 cloudera-manager 管理画面，选择`主机->所有主机`，再选择右边的`配置`按钮。在搜索里搜索 java，再`java 主目录`的输入框里输入 JDK 的路径，例如：`/usr/java/jdk1.8.0_181-amd64`

- [cloudera manager 升级到jdk1.8](https://blog.csdn.net/chenguangchun1993/article/details/78903463)
- [This change affects all CDH processes and Cloudera Management Service roles in the cluster.](
https://www.cloudera.com/documentation/enterprise/5-14-x/topics/cdh_ig_jdk_installation.html)

3，使用 spark2-shell 测试能否正常启动。
如果不能正常启动，根据错误信息，可能需要设置`yarn.scheduler.maximum-allocation-mb`和`yarn.nodemanager.resource.memory-mb`。

4，如果使用 kafka 10 的话，需要在 WebUI 的 spark 的 configuration 画面，查找 kafka 然后修改成 10。


## 9，集群中各个组件配置

### 1，yarn
#### 集群搭建
1，ResourceManager 和 NodeManager 分开。
2，ResourceManager 添加 2 个，做成高可用集群。[官方高可用设置](https://www.cloudera.com/documentation/enterprise/5-3-x/topics/cdh_hag_rm_ha_config.html#concept_nyf_vcx_5m)
#### 参考配置
（1）RM的内存资源配置, 配置的是资源调度相关
RM1：yarn.scheduler.minimum-allocation-mb 分配给AM单个容器可申请的最小内存
RM2：yarn.scheduler.maximum-allocation-mb 分配给AM单个容器可申请的最大内存

注：
- 最小值可以计算一个节点最大Container数量
- 一旦设置，不可动态改变

（2）NM的内存资源配置，配置的是硬件资源相关
yarn.nodemanager.resource.memory-mb 节点最大可用内存
yarn.nodemanager.vmem-pmem-ratio 虚拟内存率，默认2.1
yarn.scheduler.maximum-allocation-vcores：单个任务可申请的最多虚拟CPU个数，默认是32。这个配置影响启动时`--executor-cores`后面的数量。
注：
- RM1、RM2的值均不能大于NM1的值
- NM1可以计算节点最大最大Container数量，max(Container)=NM1/RM2

#### Executor 的 CPU 和 内存如何设置
- [Spark On YARN内存和CPU分配](https://blog.csdn.net/fansy1990/article/details/54314249)
- 设置 CPU 核数（yarn.nodemanager.resource.cpu-vcores）时，大数据组的方式是设置成：`( CPU core size - 1) * 2`，让程序竞争使用。

### 2，Hadoop 
#### 集群搭建
- JournalNode：运行的JournalNode进程非常轻量，可以部署在其他的服务器上。注意：必须允许至少3个节点。当然可以运行更多，但是必须是奇数个，如3、5、7、9个等等。当运行N个节点时，系统可以容忍至少(N-1)/2(N至少为3)个节点失败而不影响正常运行。 
- Failover Controller：至少 2 个。
- NameNode：2 个，需要开启高可用

参考：
- [CDH 高可用列表](https://www.cloudera.com/documentation/enterprise/5-13-x/topics/admin_ha.html)


## 10，CDH内存调拨过度警告分析
在 CDH 可能会看到这样的警告：
> Memory on host xxxxx is overcommitted. The total memory allocation is 107.8 GiB bytes but there are only 62.7 GiB bytes of RAM (12.5 GiB bytes of which are reserved for the system). Visit the Resources tab on the Host page for allocation details. Reconfigure the roles on the host to lower the overall memory allocation. Note: Java maximum heap sizes are multiplied by 1.3 to approximate JVM overhead

[CDH内存调拨过度警告分析](CDH内存调拨过度警告分析)

# HIVE
## 资料
- [hive types](https://cwiki-test.apache.org/confluence/display/Hive/LanguageManual+Types#LanguageManualTypes-DecimalsdecimalDecimals)
## 经验
### 1，命令行输出 debug log
在使用命令行执行语句时，有时候会出现错误。但错误太简单看不出是什么错误，所以有时候需要输出具体的 debug log。方法如下：
```
# 使用 hive.root.logger=DEBUG,console 参数，进入 hive
hive -hiveconf hive.root.logger=DEBUG,console
# 然后执行命令
show database;
```

例子：

第一次执行后，错误如下
```
hive> show databases;
FAILED: SemanticException org.apache.hadoop.hive.ql.metadata.HiveException: java.lang.RuntimeException: 
Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
```
使用上面的命令，进入 hive。再次执行，可以看出连接不上 ops-test-005:9083，去 ops-test-005看了一下，端口没有，服务可以挂了。
```
hive> show databases;
18/09/30 15:21:13 [main]: INFO log.PerfLogger: <PERFLOG method=Driver.run from=org.apache.hadoop.hive.ql.Driver>
18/09/30 15:21:13 [main]: DEBUG conf.VariableSubstitution: Substitution is on: show databases
18/09/30 15:21:13 [main]: INFO ql.Driver: Compiling command(queryId=shijiapeng_20180930152121_bef91299-dbaf-4f42-9a31-1d6f1acb53c5): show databases
...省略...
18/09/30 15:21:14 [main]: INFO hive.metastore: Trying to connect to metastore with URI thrift://ops-test-005:9083
18/09/30 15:21:14 [main]: WARN hive.metastore: Failed to connect to the MetaStore Server...
org.apache.thrift.transport.TTransportException: java.net.ConnectException: Connection refused (Connection refused)
	at org.apache.thrift.transport.TSocket.open(TSocket.java:226)
	at org.apache.hadoop.hive.metastore.HiveMetaStoreClient.open(HiveMetaStoreClient.java:472)
	at org.apache.hadoop.hive.metastore.HiveMetaStoreClient.<init>(HiveMetaStoreClient.java:252)
```

### 2，partition 太多删除表或删除 parition 超时
因为创建的 paritition 太多，在删除表或 partition 时超时。一超时，就发生 MetaStore 无法使用，必须重启的问题。这种情况下，如何删除表呢？

> 从 mysql 删除 partition，然后从 hive 从删除表。

1，首先进入 mysql，选择 metastore 数据库。这个数据库是 hive 使用的数据库。
2，删除 parition 相关表的信息。PARTITIONS、PARTITION_KEYS、PARTITION_KEY_VALS、PARTITION_PARAMS 这几个表是和 partition 相关的表。
3，首先查看 TBLS 表，看看删除表的 ID。`select * from TBLS;`
4，根据`表ID`删除 PARTITIONS、PARTITION_KEYS、PARTITION_KEY_VALS、PARTITION_PARAMS 表的信息。在删除 PARTITION_PARAMS 表信息时，因为有外键的关系，无法删除。所以需要先把外键删除掉，再加回来。语句如下：
> #删除外键
alter table PARTITION_PARAMS drop foreign key PARTITION_PARAMS_FK1;

>#恢复外键
>alter table PARTITION_PARAMS add CONSTRAINT `PARTITION_PARAMS_FK1` FOREIGN KEY (`PART_ID`) REFERENCES `PARTITIONS` (`PART_ID`);

### 3，如何向 hive 插入分区数据
向 hive 插入分区数据的本质是：
- 向 hive 的 hdfs 目录下创建分区目录，然后把数据拷贝分区目录下。
- 然后告诉 hive 创建了分区，可以去读分区里的内容了。（hive 可能去修改的 mysql）

插入分区数据方式有两种，都是基本上面的本质来做的，只是实现方式有所不同。

1，load 命令
一个 load 命令就可以实现本质中的两步操作。例如：
> load data local inpath '/home/hadoop/Desktop/data' overwrite into table t1 partition ( pt_d = '201701');


2，alter table 命令
alter table 命令完成的操作是`本质`中第 2 步的操作。做完这个操作，hive 就可以去分区目录里读取数据了。但这个命令不像 load 可以把数据拷贝到分区目录下，需要我们自己把数据拷贝到分区目录下。例如：
> alter table cloudeye.service_log add partition (date_ymd='2018-09-01', date_hour='00') partition (date_ymd='2018-09-01', date_hour='01')


两个命令的区别：
- load：当数据已经存在时，使用 load 命令好一些，可以直接完成我们想要的结果。
- alter table：当数据不存在，随时写时，用 alter table 好一些。例如：spark streaming 时间处理数据，处理完的数据直接就放到 hive 目录下。这时，我们可以使用 alter table 命令先创建好分区，这样写进去的数据直接就可以被搜索到，实时性高一些。