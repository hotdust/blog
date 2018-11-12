# 问题
## 1，向 kafka 发消息，在发送时卡住了，出现错误。
发送客户端是使用的 python 的 kafka-producer。错误如下：
```
ERROR:kafka.conn:DNS lookup failed for bogon:9092 (AddressFamily.AF_UNSPEC)
```

**问题原因**
不能解析bogon。kafka 连接原理，首先连接 192.168.0.141:9092，再连接返回的host.name = bogon,最后继续连接advertised.host.name=bogon。

**解决方案**
解决办法1:
hosts 文件增加：`127.0.0.1 bogon`。`ping bogon`试试如果可以ping通即可。

解决办法2:
在 server.properties 文件里加入下面的配置：
> listeners=PLAINTEXT://127.0.0.1:9092

有的文章说是使用 `advertised.listeners`，它和 `listners` 有什么关系呢？
>
listeners 用于 server 真正 bind。advertisedListeners， 用于开发给用户，如果没有设定，直接使用listeners。

关于 `listners` 和 `advertised.listeners` 请参考：
- [kafka - advertised.listeners and listeners](https://yq.aliyun.com/articles/73215)
- [Kafka Broker Advertised.Listeners属性的设置](https://www.jianshu.com/p/71b295e1df4f)

关于 kafka 的配置，请参考：
- [Kafka学习整理六(server.properties配置实践)](https://blog.csdn.net/LOUISLIAOXH/article/details/51567515)

参考：
- [java kafka 连接错误](http://blog.sina.com.cn/s/blog_998c49430102x49o.html)