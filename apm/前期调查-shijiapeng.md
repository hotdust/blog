# 一、 我们要监控什么？
1，调用链
- WebAPI 到 RPC 调用链
- RPC 调用时间
- 是否是灰度调用

2，数据库
- SQL 内容
- SQL 执行时间
- 是否走了 DAO 缓存

3，中间件
- MQ 内容
- Redis

4，Golang 分布式调用

5，APP
- todo


# 二、现有开源 APM？
## 1，现有开源 APM 都有什么？
Java 版本
Golang 版本

## 2，他们的特点是什么？



# 三、需要不需要自研？
## 1，开源 APM 和我们需求有什么不同？
## 2，开发人员的痛点？
### （1）现在看调用链是如何用 ES 看的，有什么问题。
## 3，阿里的 APM 都能监控什么？
## 4，自研的好处
- 和`云眼监控`打通



# 四、APM 注意的问题
1，调用链是否全都保存
2，以字节码增强，还是埋点的方式
3，



todo:
1，看看 New Relic 能做什么？
2，log, metrics, tracing 都是什么，都如何使用，我们都有什么了？
3，amp 不光需要 tracing，还需要 metrics。tracing 可以通过 elk 的日志可以取得，还是上传服务？metrics 就像是 cat 给我们提供的响应时间等图表。
4，为什么要使用 open tracing，有什么好处。
5，指标使用 Prometheus 试试。




---------------------
temp:
1，关于 metrics，对不同的东西，记录的指标也不一样。
- 对于 resource，比如 queue，记录`使用率（utilization）`、`饱和度（saturation）` 和 `错误数（error count）`
- 对于 endpoint（服务），记录`request count`、`error count` 和 `duration（时长）`。

2，几种`指标展示`图表：
- Counter（记数器）：记录发生的事件的个数。例如：所有请求数、出错请求数。
- Guage（仪表盘）：记录会根据时间进行波动的指标。例如：线程池里线程的个数。
- histogram（柱状图）：recording observations of scalar quantities of events。例如：请求时长。

2.5 metric 还可能记录`非 request 相关`的指标。例如，springboot 内部状态。

3，关于 logging，要记录具体的 event。例如：debug，error 。

4，关于 tracing，tracing 是 request-scoped。例如：RPC 调用的时间；实际执行的 SQL 文本；或者 http request 的关联 ID。

5，tracing 和 metrics 交集是指` Request 相关的 metrics`，例如，request duration，reuqest counter。tracing 和 logging 交集是指` Reqeust 相关的 event`，这个地方是指什么呢 todo？logging 和 metrics 的交集是指`对于 Event 的 Aggregation`，例如，就像我们的云眼，统计每种 event 的个数。

6，metrics 可以使用 Prometheus 存储。logging 可以使用 elk 存储。

7，为什么要使用 opentracing?

8，什么是 APM。

9，下一步再介绍 google 论文。

10，apdex 是什么？

11，为什么要代替 cat？
- cat 无法取得每一次的调用链。
- cat 数据很难提取，并进行二次分析使用
- 不符合 open tracing 标准
- 当知道有一个接口调用时间长，但不知道具体哪里调用时间长，还需要继续埋点。

问题：
- 1，cat 能根据某个 Traceid 查询具体的调用链吗？
- 2，cat 能取得 sql 文本吗？

12，运维人员，想从 APM 里取得什么信息呢？或者发生问题，想如何快速定位问题呢？

[Peter Bourgon · Logging v. instrumentation](https://peter.bourgon.org/blog/2016/02/07/logging-v-instrumentation.html)：
[Peter Bourgon · Metrics, tracing, and logging](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html)
[AIOps实践思考：AIOps如何与APM结合？](https://mp.weixin.qq.com/s/AcQkUiBKNOpFcUN4n3aPXQ)



===================== 下一阶段调查 =============
[应用性能监控（APM） - 掘金](https://juejin.im/post/5ba06f96f265da0ac84929e9)
[携程无线APM平台 - 如何实现全球端到端性能监控 - Ctrip无线技术 | 十条](http://www.10tiao.com/html/417/201801/2647810495/1.html)
Application_Performance_Management_For_Dummies 电子书
[产品介绍 · AliAPM 用户手册](https://apm.aliyun.com/doc/introduction.html)
[王东：微服务下的APM全链路监控-云栖社区-阿里云](https://yq.aliyun.com/articles/272142)
[Distributed Tracing in 10 Minutes – OpenTracing – Medium](https://medium.com/opentracing/distributed-tracing-in-10-minutes-51b378ee40f1)


==========================================
1，使用什么样的 APM 组件？
- 对代码侵入小的。
使用字节码增强的组件，对业务代码侵入小，可以实现 mysql 等拦截。
- 支持语言多的。
至少支持 java、go、python.
- 对除 Java 语言外，侵入少的组件。
例如 go 语言，看哪种 lib 对语言侵入少，方便。
- 支持 open tracing 的。
- UI 尽量好的。
- 存储支持多的，并且支持主流的存储的。
例如：ES 等。
- 支持中间件多的。
例如：dubbo、mysql、redis、memcache、MQ 等。
- 方便二次开发的。


skywalking 文档：
- [鲍捷的博客 | Bao Jie's Blog](http://jiebaojie.com/notes/opensource/apacheIncubatorSkywalking/#%E4%B8%AD%E9%97%B4%E4%BB%B6%E6%A1%86%E6%9E%B6%E4%B8%8E%E7%B1%BB%E5%BA%93%E6%94%AF%E6%8C%81%E5%88%97%E8%A1%A8)


比较：
- [全链路监控（一）：方案概述与比较 - 掘金](https://juejin.im/post/5a7a9e0af265da4e914b46f1)
- [分布式跟踪系统——产品对比 - Go中国技术社区 - golang](https://gocn.vip/article/852)
- [Jaeger vs Apache Skywalking](https://blog.getantler.io/jaeger-vs-apache-skywalking/)
- [Zipkin vs Jaeger: Getting Started With Tracing | Logz.io](https://logz.io/blog/zipkin-vs-jaeger/)
- [调用链选型之Zipkin，Pinpoint，SkyWalking，CAT - lissownpro的个人空间 - 开源中国](https://my.oschina.net/lissown/blog/3002548)：做了支持组件的对比





Jaeger：
基础：
- [开放分布式追踪（OpenTracing）入门与 Jaeger 实现](https://yq.aliyun.com/articles/514488?utm_content=m_43347)
- [Jaeger-分布式调用链跟踪系统理论与实战](https://cloud.tencent.com/developer/article/1160850)：里面还讲了 go 语言使用方式，代码侵入性低。
- [从Zipkin到Jaeger，Uber的分布式追踪之道tchannel - fei33423的专栏 - CSDN博客](https://blog.csdn.net/fei33423/article/details/79452948)
- [Uber分布式追踪系统Jaeger使用介绍和案例【PHP Hprose Go】](https://segmentfault.com/a/1190000011636957)
- [优步分布式追踪技术再度精进](https://www.infoq.cn/article/evolving-distributed-tracing-at-uber-engineering)

进阶：
- [jaeger的简单部署及JAVA接入 | Zepon Lin's skirt](https://www.linzepeng.com/2018/07/23/jaeger/)：Java 接入
- [Uber工程团队的开源分布式追踪系统Jaeger（java实现） - super超记的个人空间 - 开源中国](https://my.oschina.net/u/1789379/blog/1551421)：Java 接入
- [SpringCloud + Opentracing + jaeger调用链解决方案 - 小姐姐味道](http://sayhiai.com/index.php/archives/40/#comment-24)：Spring Cloud 接入
- [Uber jaeger--一个基于Go的分布式追踪系统 - 北极之北的个人空间 - 开源中国](https://my.oschina.net/u/2548090/blog/1821359)：讲了 Spring 如何使用 Jaeger，引用 Jar 包等。
- [Take OpenTracing for a HotROD ride – OpenTracing – Medium](https://medium.com/opentracing/take-opentracing-for-a-hotrod-ride-f6e3141f7941)：讲 Jaeger 的 Hotrod 例子如何使用。
- [使用elasticsearch作为存储引擎部署jaeger - 北极之北的个人空间 - 开源中国](https://my.oschina.net/u/2548090/blog/1821372)
- [opentracing-contrib/java-agent: Agent-based OpenTracing instrumentation in Java](https://github.com/opentracing-contrib/java-agent)：jaeger 的字节码增强的实现。
- [kubernetes - How istio send tracing spans to jaeger? - Stack Overflow](https://stackoverflow.com/questions/53459759/how-istio-send-tracing-spans-to-jaeger)：istio 内部如何支持 jaeger 的。
- [实例 | 当Istio遇见Jaeger，如何解决端到端分布式追踪问题？](https://zhuanlan.zhihu.com/p/34122358)：如何在 istio 上使用 jaeger。
- [Jaeger on Kubernetes 部署总结](https://mathspanda.github.io/2018/09/19/jaeger-deploy/)：讲了Jaeger on Kubernetes 部署总结，还讲了多租户和单租户的概念。






深入:
- [The life of a span – JaegerTracing – Medium](https://medium.com/jaegertracing/the-life-of-a-span-ee508410200b)：讲了从 client 到 store 之间，数据传输的格式都是什么样的。







open tracing:
- [opentracing 使用介绍](http://zenlife.tk/use-opentracing.md)
- [OpenTracing语义标准规范及实现](https://www.jianshu.com/p/a963ad0bbe3e)
- [Towards Turnkey Distributed Tracing](https://medium.com/opentracing/towards-turnkey-distributed-tracing-5f4297d1736)
- [Distributed Tracing in 10 Minutes](https://medium.com/opentracing/distributed-tracing-in-10-minutes-51b378ee40f1)
- [OpenTracing Tutorials](https://github.com/yurishkuro/opentracing-tutorial)：opentracing github
- [opentracing文档中文版 ( 翻译 ) 吴晟](https://wu-sheng.gitbooks.io/opentracing-io/content/)
- [The OpenTracing project](https://opentracing.io/)：官方网站



其它：
- [这么多监控组件，总有一款适合你](https://segmentfault.com/a/1190000017175363)
- [业务实时监控服务 ARMS](https://www.aliyun.com/product/arms?spm=5176.10695662.784055.1.11c9384cyDr1Ma)
- [浅述APM采样与端到端](https://segmentfault.com/a/1190000007370525):1.何为APM, 2.何为端到端, 3.何为Apdex(采样的做法与弊端)


1，开源组件只支持3层调用链？
2，





========================== 关于 Jaeger 的调查 ========
1，读一下 Jaeger 的文档。
2，Jaeger 提供每秒或每分钟接口调用次数、平均时间等信息吗？
OK 3，Jaeger Spark dependencies 是做什么的？
   生成服务依赖关系的任务。
OK 4，Jaeger 有没有 Dubbo 的包？
   没有，需要自己实现。
5，Jaeger 如何加入到 Spring Cloud 中？
6，opentracing-java 和 jaeger-client-java 有什么不同。
7，Jaeger 能不能利用 OpenTracing contribute 里的东西？
8，如何取得 mysql 内容。
9，如何取得 redis 内容。
10，做 Java 项目，去测试 tracing。



todo：
1，看一下 open-tracing 体系。
2，Jaeger 能不能利用 OpenTracing contribute 里的东西？
3，看搭建项目的文章，搭建项目，看每个组件如何使用。
4，




1，看 Opentracing
2，看 Trace Context Specification：Specification for distributed tracing context propagation format:



要自己做的东西：
1，Dubbo 的包
2，OpenTracing 的工具类
3，对 DAO 的修改。