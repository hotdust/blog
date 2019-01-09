
#一，编译过程
个人编译的是 4.0 版本（branch:release-4.0.0-incubating），请切换到 4.0 分支后再进行编译。
切换方法如下：(4.0.0.cp 是你自己起的分支名字)
> git checkout -b 4.0.0.cp remotes/origin/release-4.0.0-incubating

编译请参考：[Mac编译RocketMQ 4.1.0](https://www.cnblogs.com/jyris/p/6889663.html)

<br>
#二，启动过程
##1，启动 name server
先启动 name server。默认端口为：9876
> sh mqnamesrv

确认方法：
1，确认端口
>lsof -i:9876
>输出：
>java    14932  abc   39u  IPv6 xxxxxxxxxxxxxxxx      0t0  TCP *:sd (LISTEN)

2，确认启动后的 log。

>Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=320m; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: Using the DefNew young collector with the CMS collector is deprecated and will likely be removed in a future release
Java HotSpot(TM) 64-Bit Server VM warning: UseCMSCompactAtFullCollection is deprecated and will likely be removed in a future release.
Java HotSpot(TM) 64-Bit Server VM warning: Cannot open file /dev/shm/rmq_srv_gc.log due to No such file or directory

The Name Server boot success. serializeType=JSON



##2，启动 broker
启动 broker，默认端口为：10911。下面的 `localhost:9876` 是上面启动的 name server 的地址。
> sh mqbroker -n "localhost:9876"

确认方法：
1，确认端口
>lsof -i:10911
>输出：
>java    15007  sjp   74u  IPv6 0xbe5fc462f362c7a5      0t0  TCP *:10911 (LISTEN)

2，确认 log。
>Java HotSpot(TM) 64-Bit Server VM warning: Cannot open file /dev/shm/mq_gc_pid15007.log due to No such file or directory

##3，查看集群状态
> sh mqadmin clusterList -n localhost:9876
> 输出：
```
#Cluster Name     #Broker Name            #BID  #Addr                  #Version                #InTPS(LOAD)       #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE
DefaultCluster    bogon                   0     172.23.1.236:10911     V4_0_0_SNAPSHOT          0.00(0,0ms)         0.00(0,0ms)          0 423299.08 -1.0000
```

#三，收发消息
1，首先设置环境变量，下面的 localhost:9876 是 name server 的地址。注意，要使用“发送/接收”命令的 shell 都要设置。
>export NAMESRV_ADDR=localhost:9876

2，发送消息。下面的命令会自动发 1000 条消息到`TopicTest`里（这个 topic 是系统自动创建的）。
> sh tools.sh org.apache.rocketmq.example.quickstart.Producer

消息内容(部分)：
> SendResult [sendStatus=SEND_OK, msgId=AC1701EC3F1C3D4EAC69543121E303E7, offsetMsgId=AC1701EC00002A9F000000000005B20C, messageQueue=MessageQueue [topic=TopicTest, brokerName=bogon, queueId=0], queueOffset=499]

发消息之前、之后，可以使用下面的命令看 max offset 的变化。
> sh mqadmin topicStatus -n localhost:9876 -t TopicTest

3，接收消息。下面的命令会自动从`TopicTest`里接收消息。
> sh tools.sh org.apache.rocketmq.example.quickstart.Consumer

消息内容(部分)：
>ConsumeMessageThread_9 Receive New Messages: [MessageExt [queueId=1, storeSize=180, queueOffset=229, sysFlag=0, bornTimestamp=1523923911030, bornHost=/172.23.1.236:62028, storeTimestamp=1523923911031, storeHost=/172.23.1.236:10911, msgId=AC1701EC00002A9F000000000002B9B2, commitLogOffset=178610, bodyCRC=157816852, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=250, CONSUME_START_TIME=1523923946095, UNIQ_KEY=AC1701EC3EDA3D4EAC6954280D760395, WAIT=true, TAGS=TagA}, body=18]]]


#四，使用 web 控制台
搭建方法请参看：[官方搭建方法](https://github.com/apache/rocketmq-externals/tree/master/rocketmq-console)

启动方法：
> java -jar rocketmq-console-ng-1.0.0.jar --server.port=12581 --rocketmq.config.namesrvAddr=localhost:9876

- server.port：是 web 应用的端口
- rocketmq.config.namesrvAddr：是 name server 的地址

还有其它参数可以设置，请看`src/main/resources/application.properties`文件中的项目。


#五，集群搭建
请参考：
[ RocketMQ（二）集群配置](https://blog.csdn.net/lovesomnus/article/details/51769977)
[官方 Deployment](https://rocketmq.apache.org/docs/rmq-deployment/)
[RocketMQ-集群方式及环境搭建](https://www.jianshu.com/p/9d4e0ff358c6)
