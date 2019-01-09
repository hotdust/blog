> 此文章对应 rocketmq 版本为：4.0.0


#一，修改文件
####1，namesrv 模块 -> NamesrvStartup.java
因为每台机器目录结构可能不一样，这里使用通用方式设置 rocketmq home
```
final NamesrvConfig namesrvConfig = new NamesrvConfig();// 这段代码是原来的代码
// add 设置 rocketmq home
String rocketMQHome = Thread.currentThread().getContextClassLoader().getResource("").getFile().replaceAll("/namesrv/target/classes/", "");
namesrvConfig.setRocketmqHome(rocketMQHome);
```


####2，broker 模块 -> BrokerStartup.java
因为每台机器目录结构可能不一样，这里使用通用方式设置 rocketmq home
```
final BrokerConfig brokerConfig = new BrokerConfig();// 这段代码是原来的代码
// add 设置 rocketmq home。因为每台机器目录结构可能不一样，这里使用通用方式
String rocketMQHome = Thread.currentThread().getContextClassLoader().getResource("").getFile().replaceAll("/broker/target/classes/", "");
brokerConfig.setRocketmqHome(rocketMQHome);
```

####3，example 模块 -> Producer.java
设置 name server。在使用命令行发消息时，也是需要设置一下环境变量的，所以在IDE里启动命令行的类时，也需要设置一下。
```
// add 设置 name server
producer.setNamesrvAddr("localhost:9876");

producer.start();// 这段代码是原来的代码
```

####4，example 模块 -> Consumer.java
设置 name server。在使用命令行发消息时，也是需要设置一下环境变量的，所以在IDE里启动命令行的类时，也需要设置一下。
```
// add 设置 name server
consumer.setNamesrvAddr("localhost:9876");

consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);// 这段代码是原来的
```

所有文件都修改完后，可以使用下面的命令来看一下 cluster 是否启动成功。
> sh mqadmin clusterList -n localhost:9876


<br>
#二，启动

启动顺序如下：

- NamesrvStartup
- BrokerStartup
- Producer
- Consumer


#参考：

- [RocketMQ源码阅读——debug环境构建](RocketMQ源码阅读——debug环境构建)
- [ROCKETMQ 源码debug 问题集锦](https://www.jianshu.com/p/8388278dd05a)