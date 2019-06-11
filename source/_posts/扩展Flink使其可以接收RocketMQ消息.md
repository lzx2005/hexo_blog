---
title: 扩展Flink使其可以接收RocketMQ消息
date: 2019-06-11 16:58:22
tags:    
    - Java
    - 中间件
    - 大数据
    - Flink
    - RocketMQ
categories:
    - Java
    - 中间件
---

从官方文档可以看到，Flink支持的数据源有如下几个：

- [Apache Kafka](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/connectors/kafka.html) (source/sink)
- [Apache Cassandra](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/connectors/cassandra.html) (sink)
- [Amazon Kinesis Streams](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/connectors/kinesis.html) (source/sink)
- [Elasticsearch](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/connectors/elasticsearch.html) (sink)
- [Hadoop FileSystem](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/connectors/filesystem_sink.html) (sink)
- [RabbitMQ](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/connectors/rabbitmq.html) (source/sink)
- [Apache NiFi](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/connectors/nifi.html) (source/sink)
- [Twitter Streaming API](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/connectors/twitter.html) (source)

对于其他的源，它也提供了接口给我们实现，扩展性非常好，今天我们就实现一个从RocketMQ取数据的实现。

打开项目，转到Flink处理类，可以看到：

```scala
object FlinkKafkaConsumerDemo {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val properties = new Properties()
    properties.setProperty("bootstrap.servers", "localhost:9092")
    properties.setProperty("group.id", "test")
    val data = env.addSource(new FlinkKafkaConsumer[String]("lzxtest", new SimpleStringSchema(), properties))
    data.print()
    env.execute("FlinkKafkaConsumerDemo")
  }
}
```

我们按住Command，点击`FlinkKafkaConsumer`查看其结构：

```java
public class FlinkKafkaConsumer<T> extends FlinkKafkaConsumerBase<T>{}

public abstract class FlinkKafkaConsumerBase<T> extends RichParallelSourceFunction<T> implements CheckpointListener, ResultTypeQueryable<T>, CheckpointedFunction {}

public abstract class RichParallelSourceFunction<OUT> extends AbstractRichFunction implements ParallelSourceFunction<OUT> {}

public interface ParallelSourceFunction<OUT> extends SourceFunction<OUT> {}

public interface SourceFunction<T> extends Function, Serializable {
    void run(SourceFunction.SourceContext<T> var1) throws Exception;

    void cancel();

    @Public
    public interface SourceContext<T> {
        void collect(T var1);

        @PublicEvolving
        void collectWithTimestamp(T var1, long var2);

        @PublicEvolving
        void emitWatermark(Watermark var1);

        @PublicEvolving
        void markAsTemporarilyIdle();

        Object getCheckpointLock();

        void close();
    }
}
```

可以看到，要实现自定义的Source，只需要实现接口`SourceFunction`即可，如果需要并行消费，可以实现`ParallelSourceFunction`。

创建类`RocketMQSourceFunction`，继承`SourceFunction`：

```java
public class RocketMQSourceFunction implements SourceFunction<String> {
    @Override
    public void run(SourceContext<String> sourceContext) throws Exception {
        // todo 
    }

    @Override
    public void cancel() {
        // todo 
    }
}
```

首先我们需要准备一个RocketMQ的消费者客户端，打开`pom.xml`，添加如下依赖：

```xml
<dependency>
  <groupId>org.apache.rocketmq</groupId>
  <artifactId>rocketmq-client</artifactId>
  <version>4.3.0</version>  <!-- 版本号注意修改 -->
</dependency>
```

对于`RocketMQSourceFunction`来说，我们需要初始化一个Consumer，所以添加代码如下：

```java
public class RocketMQSourceFunction implements SourceFunction<String> {
  private static final DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumerGroupTest");
  ...
}
```

这样，当类在加载的时候，系统会创建一个consumer。对于一个Consumer来说，还需要知道我们要消费的nameSrvAddr和Topic是什么，所以我们添加字段：

```java
public class RocketMQSourceFunction implements SourceFunction<String> {
  public RocketMQSourceFunction(String nameSrvAddr, String topic) {
    this.nameSrvAddr = nameSrvAddr;
    this.topic = topic;
  }
  private String nameSrvAddr;
  private String topic;
  ...
}
```

重写Run方法：

```java
@Override
public void run(SourceContext<String> sourceContext) throws Exception {
  consumer.setNamesrvAddr(nameSrvAddr);
  consumer.subscribe(topic, "*");
  consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
    msgs.forEach(msg -> {
      sourceContext.collect(new String(msg.getBody(), Charset.forName("UTF-8")));
    });
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
  });
  consumer.start();
  System.out.println("consumer started");
}
```

consumer会在接收到消息时，发送消息到sourceContext中，这样Flink的流就可以接收到消息了。同时不要忘了重写cancal方法：

```java
@Override
public void cancel() {
  consumer.shutdown();
}
```

这样，一个完整的RocketMQ的数据源接收器我们已经实现好了，在需要用到的Flink代码中加入：

```scala
object FlinkRocketMQConsumerDemo {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val rmq = env.addSource(new RocketMQSourceFunctionJava("localhost:9876", "lzxtest")).setParallelism(1)
    rmq.print().setParallelism(1)
    env.execute("FlinkRocketMQConsumerDemo")
  }
}
```

这样，Flink即可接收到RocketMQ的消息了。