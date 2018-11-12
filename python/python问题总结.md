# 问题
## 1，如何输出 debug log
```
> python 
 
>>> from kafka import * 
>>> import logging 
>>> logging.basicConfig(level=logging.DEBUG) 
>>> consumer = KafkaConsumer(bootstrap_servers=['kafkabroker_host:9092']) 
DEBUG:kafka.metrics.metrics:Added sensor with name connections-closed 
DEBUG:kafka.metrics.metrics:Added sensor with name connections-created 
DEBUG:kafka.metrics.metrics:Added sensor with name select-time 
```

参考：
- [How to enable debug logging for Kafka-Python?](https://community.hortonworks.com/content/supportkb/150084/how-to-enable-debug-logging-for-kafka-python.html)
- [Debugging kafka client in Python](https://vevurka.github.io/dsp17/python/debug/python_kafka_issue/)