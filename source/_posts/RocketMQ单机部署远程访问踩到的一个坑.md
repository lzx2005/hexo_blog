---
title: RocketMQ单机部署远程访问踩到的一个坑
date: 2019-05-23 14:34:26
tags:    
    - Java
    - 中间件
categories:
    - Java
    - 中间件
---

之前根据`rocketmq.apache.org`的`get started`步骤部署的RocketMQ单节点，在远程访问的时候会出现如下问题：

```java
Startorg.apache.rocketmq.remoting.exception.RemotingTooMuchRequestException: sendDefaultImpl call timeout
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(DefaultMQProducerImpl.java:612)
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1253)
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1203)
	at org.apache.rocketmq.client.producer.DefaultMQProducer.send(DefaultMQProducer.java:214)
	at com.cmit.fabric.java.rocketmq.RocketMQTest.producer.ProducerTest.producerStart(ProducerTest.java:39)
	at com.cmit.fabric.java.rocketmq.RocketMQTest.producer.ProducerTest.main(ProducerTest.java:27)
```

在查询了相关资料以后，得到解决方案，假设我们的IP是：`172.16.10.53`，修改配置文件`broker.conf`，加上：

```properties
brokerIP1=172.16.10.53
```

且需要在启动namesrv和broker的时候加上本机IP：

```shell
sh bin/mqnamesrv -n 172.16.10.53:9876
sh bin/mqbroker -n 172.16.10.53:9876 -c conf/broker.conf autoCreateTopicEnable=true
```

启动后远程访问调用成功。