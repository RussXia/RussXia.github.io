---
layout: blog
title: "Java集成Kafka简单步骤"
catalog: true
tag: [Java,Kafka,2018]
---
# Java集成Kafka简单步骤

## 准备

kafka下载地址:[http://mirror.bit.edu.cn/apache/kafka/1.0.1/kafka_2.11-1.0.1.tgz](http://mirror.bit.edu.cn/apache/kafka/1.0.1/kafka_2.11-1.0.1.tgz)

项目代码地址:[https://github.com/RussXia/kafka_demo](https://github.com/RussXia/kafka_demo)

### 1.启动zookeeper

`./bin/zookeeper-server-start.sh ./config/zookeeper.properties`

zookeeper.properties 

```properties
dataDir=/tmp/zookeeper
clientPort=2181
maxClientCnxns=0
```

### 2.启动kafka-server

 `./bin/kafka-server-start.sh ./config/server-1.properties`
 `./bin/kafka-server-start.sh ./config/server-2.properties`

server-1.properties

```properties
broker.id=1
listeners=PLAINTEXT://:9091
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/tmp/kafka-logs-1
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=localhost:2181
zookeeper.connection.timeout.ms=6000
group.initial.rebalance.delay.ms=0
```

server-2.properties

```properties
broker.id=2
listeners=PLAINTEXT://:9092
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/tmp/kafka-logs-2
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=localhost:2181
zookeeper.connection.timeout.ms=6000
group.initial.rebalance.delay.ms=0
```

### 3.创建topic

`./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 2 --partitions 1 --topic test_topic`

response:

```text
WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Created topic "test_topic".
```

### 4.查询topic状态

`./bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test_topic`

response:

```text
Topic:test_topic    PartitionCount:1    ReplicationFactor:2 Configs:
Topic:test_topic   Partition: 0 Leader: 1   Replicas: 1,2   Isr: 1,2
```

## Java集成Kafka

### 1.添加依赖

```xml
<dependencies>
    <!-- https://mvnrepository.com/artifact/org.apache.kafka/kafka-clients -->
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>
```

### 2.Producer发送消息

关于Producer相关参数的含义，请参考下面的两个Class

`org.apache.kafka.clients.CommonClientConfigs`
`org.apache.kafka.clients.producer.ProducerConfig`
下面是Producer的示例代码

```java
@Slf4j
public class ProducerDemo {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, "localhost:9091,localhost:9092");
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, 0);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        //发送异步消息
        Producer<String, String> producer = new KafkaProducer<>(props);
        for (int i = 0; i < 100; i++) {
            producer.send(new ProducerRecord<>("test_topic", Integer.toString(i), "HelloWorld" + i));
        }
        //发送带回调的异步消息
        producer.send(new ProducerRecord<>("test_topic", "Test", "Hello123"), new Callback() {
            @Override
            public void onCompletion(RecordMetadata metadata, Exception exception) {
                if (exception != null) {
                    log.error(exception.getMessage());
                }
                System.out.print("The offset is : ");
                System.out.println(metadata.hasOffset() ? metadata.offset() : "0");
            }
        });
        producer.close();
    }
}
```

查看消息是否发送成功

`./bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test_topic --from-beginning`

### 3.Consumer消费消息

关于Producer相关参数的含义，请参考下面的两个Class

`org.apache.kafka.clients.CommonClientConfigs`

`org.apache.kafka.clients.consumer.ConsumerConfig`

下面是Consumer的示例代码

```java
@Slf4j
public class ConsumerDemo {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, "localhost:9091,localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "consumer_test_1");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        //订阅topic
        consumer.subscribe(Collections.singletonList("test_topic"));
        while (true) {
            //拉数据
            ConsumerRecords<String, String> records = consumer.poll(10000);
            for (ConsumerRecord<String, String> record : records)
                System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
        }
    }
}
```

## Spring boot与Kafka的集成

这一段可以参见：[https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-kafka](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-kafka)

本人搭的一个小的demo:
[https://github.com/RussXia/Kafka_SpringBoot_Integration](https://github.com/RussXia/Kafka_SpringBoot_Integration)

pom依赖：

```xml
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.2.RELEASE</version>
        <relativePath/>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- spring-kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
            <version>${spring-kafka.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <version>${spring-kafka.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

yaml配置:

```yml
server:
  port: 28001

spring:
  application:
      name: spring-boot-demo
  kafka:
    consumer:
      auto-offset-reset: earliest
      group-id: kafka_integration_1
      bootstrap-servers: localhost:9091,localhost:9092
kafka:
  topic: test_topic_1
```

Producer和Consumer代码与之前的类似，只是将其放到Spring的bean中管理。在此就不在赘述了。
