# Prometheus
为什么选用 Prometheus?
- 可扩展性，高可用
- 报警灵活
- 支持度高。Prometheus 支持的 Agent 多，很多中间件输出 metrics 默认就是 Prometheus 格式的。


## 关于 Alert 配置
对于是否发送 Alert 给 AlertManager，是由 Prometheus Server 的 Alerting rules 来控制。还有几个配置也和 Alert 有关系，下面说明一下。

**Global 配置：**
- scrape_interval：多长时间抓取一次数据。
- evaluation_interval：多长时间去执行一下 Alerting rules 中的规则。

**Alerting rules 配置：**
这里只想说明一下 expr 和 for 关键字。
- expr：根据 Prometheus expression 来定义 alert condition，例如：up == 0。
- for：表示 expr 定义的规则，一直持续多长时间后，才报警。（一直的意思是，每次 evaluation_interval 时间后，执行 expr 时都符合条件。）

从 Alert 从`Prometheus Server`到`AlertManager` 的过程，及一些参数的作用，可参考文章：
- [Prometheus: understanding the delays on alerting](https://pracucci.com/prometheus-understanding-the-delays-on-alerting.html)
- [Prometheus智能化报警流程避免邮件轰炸-xujpxm-51CTO博客](https://blog.51cto.com/xujpxm/2055970)



# AlertManager
## Route 配置
配置：
```
route:
 group_by: ['alertname']
 group_wait: 5s
 group_interval: 10s
 repeat_interval: 1h
 receiver: 'web.hook'
```

报警内容：
```
[
  {
    "labels": {
      "alertname": "Tesl",
      "random": "1",
      "severity": "critical"
    }
  }
]
```

### group_by
是指按哪些 label 进行分组。括号里的内容是 label 名称。报警内容中有 3 个 label，配置中指定按 alertname 这个 label 进行分组。分组后的例子如下：
```
{
  "receiver": "web\\.hook",
  "status": "firing",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "TestAlert1",
        "env": "local",
        "random": "1"
      },
      "annotations": {},
      "startsAt": "2019-03-06T15:40:04.174101+08:00",
      "endsAt": "0001-01-01T00:00:00Z",
      "generatorURL": ""
    }
  ],
  "groupLabels": {
    "alertname": "TestAlert1"
  },
  "commonLabels": {
    "alertname": "TestAlert1",
    "env": "local",
    "random": "1"
  },
  "commonAnnotations": {},
  "externalURL": "http://shijiapeng.local:9093",
  "version": "4",
  "groupKey": "{}:{alertname=\"TestAlert1\"}"
}
```

### group_wait
当一个 alert 创建了一个新的 group 的时候，会等待这个时间，将相同的 group 的报警合并后，再发出去。

例如，配置中的值是 5s。也就是说，当报警进来时，创建了一个新的 group 的话（注意：这里指 group 之前不存在，后面会说到 group 存在的情况。），不会立刻发送出去，会等 5s 后再发送。在等待这 5s 时，如果有新的报警进来，会合并到 group 中一起发送出去。发送的例子可以参考上面的例子。

### group_interval
当一个 group 存在，并且发送发报警（也就 group_wait 时间过了，执行了报警）。在这之后有新的报警来的话，不会立刻发送出去，会等待  group_interval 时间后，再发送出去。等待的原因也是为了合并。
例如，在例子中，group_wait：5s，group_interval：10s。

| 报警 | 发出时间 | 报警时间 |
|---|---|---|
| 1 | 10:00 | 10:05（group_wait：5s） |
| 2 | 10:01 | 10:05（group_wait：5s） |
| 3 | 10:07 | 10:15（group_wait：5s + group_interval：10s）|
| 4 | 10:24 | 10:25（group_wait：5s + group_interval：10s * 2） |

注意：
group_interval 不是建立在`报警`发出的时间基础上，而是建立在 group_wait 的基础上。例如：第一次 group_interval 执行时间，是 group_wait + group_interval = 15s 后，第二次 group_interval 时间是， group_wait + group_interval * 2 = 25s 后。


为什么有了 group_wait 还要 group_interval。个人感觉是因为，有时间有问题时，不想等待合并，想立刻报出来一条，让大家知道现在可能会有问题。之后的大量报警需要合并，这时  group_interval 就有用了。

### repeat_interval
这个和 resolve_timeout 有关系。resolve_timeout 是等待多长时间后，没有此类报警后，就算恢复。repeat_interval 的意思，如果在 repeat_interval 这么长时间后，还没有恢复的话，就把问题再发一次，让大家知道问题还没有解决。


## inhibit_rules 配置
这个配置的目的是，消除冗余的告警。举例来说：同一台server-A的告警，如果有如下两条告警，并且配置了抑制规则。
- mysql_uptime
- server_uptime

最后只会收到一条server_uptime的告警。A机器挂了，势必导致A服务器上的mysql也挂了；如配置了抑制规则，通过服务器down来抑制这台服务器上的其他告警；这样就能消除冗余的告警，帮助运维第一时间掌握最核心的告警信息。

配置例子
```
global:
  resolve_timeout: 5m
 
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname']
```

source_match、target_match 和 equal 里面都是 label。

注意：如果有 warning 先过来并且报警了。当 critical 来时，在报警时会把之前的 warning 去掉，在之后的报警中就不包含 warning 了。



## API
可以使用 API 向发送报警：https://prometheus.io/docs/alerting/clients/


### 向 AlertManager 发送 Alert
一般都是 Prometheus 向 AlertManager 发送 Alert 报警。我们也可以不使用 Prometheus，而使用自己的应用向 AlertManager 发送 Alert。下面就是一个自己向 AlertManager 发送的 Alert 例子：
```
{
  "labels": {
    "alertname": "TestAlert2",
    "env": "local",
    "random": "156",
    "severity": "critical",
    "code": "1000"
  }
}
```
**命令（命令前面的 date; 可以不要，只是为了查看发送时间方便）：**
```
date;curl -H "Content-Type: application/json" -d '[{"labels":{"alertname":"TestAlert2","env":"local","random":"156","severity":"critical", "code":"1000"}}]' http://localhost:9093/api/v1/alerts
```

对于 AlertManager，如何识别和使用 Alert 内容，请参看：[Clients | Prometheus](https://prometheus.io/docs/alerting/clients/)

对于 Prometheus Server，如何产生 Alert 内容，请参看：[Alerting rules | Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)

注意：AlertManager 是会对 Alert 进去重的，，去重规则是以 labels 做为`数据唯一性`的定义。annotations 不做为`数据唯一性`的定义。具体看：https://prometheus.io/docs/alerting/clients/




### 向 AlertManager 发送`恢复 Alert`
AlertManager 如果接收不到 Alert 的`恢复 Alert`的话，当 resolve_timeout 设置的时间到了，会自动发恢复相应的 Alert。当然我们也可以向 AlertManager 手动发送`恢复 Alert`，发送的条件有两个：
- 手动发送的`恢复 Alert`和之前的 Alert 的 Lables 里的内容必须一致。
- 增加一个 endsAt 属性，这个属性和 Lables 属性同级。格式为：0000-01-01T00:00:00Z。（只要有这个属性就就代表恢复，不在乎值为 0000-01-01T00:00:00Z，还是 2019-01-01T00:00:00Z）

例如：
```
{
  "labels": {
    "alertname": "TestAlert2",
    "env": "local",
    "random": "156",
    "severity": "critical",
    "code": "1000"
  },
  "endsAt": "0000-01-01T00:00:00Z"
}
```

## 报警积累
AlertManager 在接收到报警后，对于`示恢复的报警`是进行`积累发送`的，下面解释一下是什么是`积累发送`。

1. 在 group_wait 时间内，接收到了 A、B 两个 Alert，AlertManager 发一个通知，告诉有 2 个 Alert。
2. 然后在 group_interval 时间内，又接到 C、D、E 三个 Alert，并且 A、B Alert 并没有恢复。这时 AlertManager 发出的通知，是包括这一共 5 个 Alert 的。

我们常用的方式可能是，只想报告`某段间隔`内发生的 Alert。这两种方式的区别，有的文章说，是属于 level triggered，而不是 edge triggered。下面是文章 [Chris's Wiki :: blog/sysadmin/PrometheusAlertsProblem](https://utcc.utoronto.ca/~cks/space/blog/sysadmin/PrometheusAlertsProblem?showcomments#comments) 中的一段：
> One way to put this is to say that Alertmanager is sort of level triggered instead of edge triggered

什么是 Edge triggered 和 Level triggered。这两个名词是 IO 相关的名词，解释如下：
- Edge-Triggered：字面上理解就是指“边缘触发”，说的是当状态变化的时候触发。以后如果状态一直没有变化或没有重新要求系统给出通知，将不再通知应用程序。
- Level-Triggered：是指“条件触发”，说的是在某种状态下触发，如果一直在这种状态下就一直触发。

举个读socket的例子，假定经过长时间的沉默后，现在来了100个字节。这时无论`边缘触发`和`条件触发`都会产生一个`read ready notification`通知应用程序可读。应用程序读了 50 个字节，然后重新调用api等待io事件。
- 这时`条件触发`的api会因为还有 50 个字节可读（处于`还有数据可读`的条件下），从而立即返回用户一个 read ready notification，告诉用户有数据可以读取。如果用户不读取，就会一起发通知。
- 而边缘触发的 api 会因为`可读`这个状态没有发生变化（因为没有新数据进来，所以状态没有变化。没有读完的那些数据是`老数据`，不会改变状态），而陷入长期等待。因此在使用边缘触发的api时，要注意每次都要把所有的数据都读完。

从 AlertManager 来看，如果 Alert 没有恢复，就像上面的`条件触发`的数据没有读完一样，会一直通知用户还有 Alert 没有被恢复。

下面的文章是关于 AlertManager 的一些感觉不太合理的地方的看法：
- [PrometheusAlertsClearingTime](https://utcc.utoronto.ca/~cks/space/blog/sysadmin/PrometheusAlertsClearingTime)：这个是关于 AlertManager 当有 Alert 被恢复时，为什么要发送一个通知，里面带有 Resolved 和 Unresolved 的 Alert 的。
- [PrometheusAlertsProblem](https://utcc.utoronto.ca/~cks/space/blog/sysadmin/PrometheusAlertsProblem)：这个是说 Alert 积累问题的。





## 参考
- [第十章 alertmanager 报警规则详解 · 如何以优雅的姿势监控kubernetes · 看云](https://www.kancloud.cn/huyipow/prometheus/527563)：讲了 AlertManager 的基本作用和规则。
- [AlertManager 告警组件 | TiDB-OPS](https://www.tidb.cc/Monitor/170607-AlertManager.html)：有 templdate 例子 和 测试 AlertManager 方法。
- [报警神器 AlertManager 的使用_ - jishuwen(技术文)](https://www.jishuwen.com/d/2xia)：和 k8s 的结合，还有 python 写的 webhook。
- [Firing -> Resolved -> Firing issue with prometheus + alertmanager · Issue #952 · prometheus/alertmanager](https://github.com/prometheus/alertmanager/issues/952)：从这个 issue 上找到的 Prometheus 向 AlertManager 发送的 Json 的例子。







# 参考
- [第1章 天降奇兵 - prometheus-book](https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/quickstart)：非常好的 Prometheus 的学习手册，非常全，还容易读。
- [《Prometheus Book》阅读笔记 | Stdio's Blog](https://blog.stdioa.com/2018/11/prometheus-book-note/)：Prometheus 的一个快速了解的文章。
- [实战 Prometheus 搭建监控系统 - aneasystone's blog](https://www.aneasystone.com/archives/2018/11/prometheus-in-action.html)：另一个 Prometheus 的一个快速了解的文章。
- [Prometheus一条告警是怎么触发的 - 知乎](https://zhuanlan.zhihu.com/p/43637757)：Prometheus报警的一个了解用的好文章。
- [Prometheus 实战](https://songjiayang.gitbooks.io/prometheus/content/)：一个非常全的，有不明白可以查查。
- [Prometheus、Alertmanager 高可用部署 - 简书](https://www.jianshu.com/p/657e637c6e75)：高可用部署文章。

Good：
- [Prometheus: understanding the delays on alerting](https://pracucci.com/prometheus-understanding-the-delays-on-alerting.html)：非常好的讲解，Alert 从`Prometheus Server`到`AlertManager` 的过程。
- [Prometheus智能化报警流程避免邮件轰炸-xujpxm-51CTO博客](https://blog.51cto.com/xujpxm/2055970)：讲了 Alert 的运转流程。



# 深入阅读
- [My Philosophy on Alerting](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/edit)：关于`报警`的一些哲学
- [Google - Site Reliability Engineering](https://landing.google.com/sre/sre-book/chapters/monitoring-distributed-systems/)
- [Avoiding Alerts Overload from Microservices - YouTube](https://www.youtube.com/watch?v=lC5SfTMFK3M)
- [Recording rules | Prometheus](https://prometheus.io/docs/practices/rules/)：Prometheus BEST PRACTICES
- [Prometheus Blog Series (Part 1): Metrics and Labels](https://blog.pvincent.io/2017/12/prometheus-blog-series-part-1-metrics-and-labels/)：这是一个介绍 Prometheus 的系列的 Blog。感觉挺不错的，还没有细看。


