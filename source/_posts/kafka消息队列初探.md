---
title: kafka消息队列初探
tags:
  - kafka
categories:
  - big-data
abbrlink: cedf1ac1
date: 2018-03-08 11:44:54
---

### JMS

<br/>

#### java message service

java消息服务。

<br/>

#### queue

1. 只有能有一个消费者。P2P模式(点对点).
2. 发布订阅(publish-subscribe,主题模式)，

<br/>

{% asset_img k1.png %}

JMS：

两种模式：

p2p模式：只有能有一个消费者。P2P模式(点对点).

发布订阅(publish-subscribe,主题模式)，

<br/>

kafka：

用**消费者组**的方式把两种模式组合在一起，把所有的用户都放在同一个组中，此时只会有一个消费者收到消息，把所有的消费者都单独放在一个组中，此时所有的消费者都会收到消息。

<br/>



### kafka

1. 分布式流处理平台。
2. 在系统之间构建实时数据流管道。
3. 以topic分类对记录进行存储
4. 每个记录包含key-value+timestamp
5. 每秒钟百万消息吞吐量。

<br/>

producer			//消息生产者

consumer			//消息消费者

consumer group		//消费者组

kafka server			//broker,kafka服务器

topic				//主题,副本数,分区.

zookeeper			//hadoop namenoade + RM HA | hbase | kafka

<br/>



### 应用场景：缓冲

​	实时处理，准实时处理，数据的产生速度 ，远远大于storm或者spark streaming 的计算速度(也就是集群本身的吞吐量并不大)，此时如果采用推模式，storm或者spark streaming很容易爆掉，此时就要引进kafka，数据先进kafka，本身是分布式的，提供可靠性，而且基于内存的，本身的速度非常快，每秒钟百万记录的吞吐量。此时计算系统采用拉模式，有多大的计算速度，就去拉取多少数据。

​	例如通话记录要实时监控，此时就要把数据先放进kafka，然后进入storm，消费电话记录中的每句话，抓敏感词，发现即加入黑名单。



<br/>

### 安装kafka

1. 选择s202 ~ s204三台主机安装kafka
2. 准备zk：略
3. jdk：略
4. tar文件
5. 环境变量
6. 配置kafka

```shell
[kafka/config/server.properties]
...
broker.id=202
...
listeners=PLAINTEXT://:9092
...
log.dirs=/home/centos/kafka/logs
...
zookeeper.connect=mini1:2181,mini2:2181,mini3:2181
```

1. 分发server.properties，同时修改每个文件的broker.id
2. 启动kafka服务器

- a)先启动zk
- b)启动kafka

```shell
[s202 ~ s204]
$>bin/kafka-server-start.sh -daemon config/server.properties
```

- c)验证kafka服务器是否启动

```shell
$>netstat -anop | grep 9092
```

1. 创建主题 

```shell
$>bin/kafka-topics.sh --create --zookeeper mini1:2181 --replication-factor 3 --partitions 3 --topic calllog
```

1. 查看主题列表

```shell
$>bin/kafka-topics.sh --list --zookeeper mini1:2181
```

1. 启动控制台生产者

```shell
[ 注意9092是kafka的端口，要选择一个开启了kafka服务的节点来作为生产者 ]
$>bin/kafka-console-producer.sh --broker-list mini2:9092 --topic calllog
```

1. 启动控制台消费者

```shell
$>bin/kafka-console-consumer.sh --bootstrap-server mini2:9092 --topic test2 --from-beginning --zookeeper mini1:2181
```

1. 在生产者控制台输入hello world
2. 删除主题

```shell
bin/kafka-topics.sh --zookeeper mini1:2181 --delete --topic calllog
```



<br/>

### kafka集群在zk的配置

/controller			===>	{"version":1,"brokerid":202,"timestamp":"1490926369148"

<br/>

/controller_epoch	===>	1

/brokers

/brokers/ids			:记载kafka集群的每个节点的信息

/brokers/ids/202	===>	{"jmx_port":-1,"timestamp":"1490926370304","endpoints":["PLAINTEXT://s202:9092"],"host":"s202","version":3,"port":9092}

/brokers/ids/203

/brokers/ids/204	

**每个分区有自己的leader，创建时指定的副本数，指的是每个分区的副本数**

/brokers/topics/test/partitions/0/state ===>{"controller_epoch":1,"leader":203,"version":1,"leader_epoch":0,"isr":[203,204,202]}

/brokers/topics/test/partitions/1/state ===>...

/brokers/topics/test/partitions/2/state ===>...

/brokers/seqid		===> null

/admin

/admin/delete_topics/test		===>标记删除的主题

/isr_change_notification

/consumers/xxxx/		consumer下面是消费者组

/config



<br/>

### 容错

{% asset_img k2.png %}

<br/>

{% asset_img k3.png %}

<br/>

### 结构示意图

{% asset_img k4.png %}

注意：topic，broker(kafka集群节点)，consumer，都是会向集群注册的，consumer每次去zk上查询topic所在节点。

但是**唯独producer不会向集群注册**，它是直接连接用list的方式连接broker的。broker采用的是副本技术，一个连接不上会去连接另外的。

<br/>

### 多个kafka集群之间通信

{% asset_img k5.png %}

<br/>



### 创建主题

repliation_factor 2 partitions 5			

**副本数一定要小于broker数**

<br/>

```shell
$>bin/kafka-topics.sh --create --zookeeper mini1:2181 --replication-factor 2 --partitions 5 --topic test2
```

<br/>

2 x 5  = 10		//10个文件夹

<br/>

```shell
[s202]/home/centos/kafka/logs
test2-1			//
test2-2			//
test2-3			//
```

```shell
[s203]/home/centos/kafka/logs
test2-0
test2-2
test2-3
test2-4
```

```shell
[s204]/home/centos/kafka/logs
test2-0
test2-1
test2-4
```

<br/>



### 重新布局分区和副本，手动再平衡

```shell
$>kafka-topics.sh --create --zookeeper s202:2181 --topic test2 --replica-assignment 203:204,203:204,203:204,203:204,203:204
```

只能在创建的时候手动分区，修改的时候修改不成功，是一个bug。

<br/>



### 副本

{% asset_img k6.png %}

<br/>

1. kafka不是把消息切碎，然后把每部分发送到每个分区中，而是把整个消息发送到一个分区中。
2. [ kafka通过取哈希的方式来确定把消息发送到那个分区中去， 这样做也是为了负载均衡]
3. [ 所以说一个消息肯定会发送到一个分区中去，而不是切碎消息。  ]
4. broker存放消息以消息达到顺序存放。生产和消费都是副本感知的。
5. 支持到n-1故障。每个分区都有leader，follow.
6. leader挂掉时，消息分区写入到本地log或者，向生产者发送消息确认回执之前，生产者向新的leader发送消息。
7. 新leader的选举是通过isr进行，第一个注册的follower成为leader。

<br/>

### kafka支持副本模式

发送者向主题发送消息，主题是一个很庞大的概念，最终需要落实到分区上去，分区需要落实到分区的副本上去，落实到副本就是落实到副本的leader和follower上去，

{% asset_img k7.png %}

<br/>

- [同步复制]

1. producer联系zk识别leader
2. 向leader发送消息
3. leadr收到消息写入到本地log
4. follower从leader pull消息
5. follower向本地写入log
6. follower向leader发送ack消息
7. leader收到所有follower的ack消息
8. leader向producer回传ack

<br/>

同步就是生产者发来的消息要等到所有的follower都复制完成之后才能收到回执信息，才能继续往下进行。可以保证消息是可靠的。

<br/>

- [异步副本]

1. 和同步复制的区别在与leader写入本地log之后，
2. 直接向client回传ack消息，不需要等待所有follower复制完成。

<br/>

### 偏移量的概念：在zk的/consumer下可以看到

每个消费者对每个主题上的每个分区都会有一个偏移量的概念：从第二次收到消息之后，会从偏移量开始消费。

{% asset_img k8.png %}

<br/>

{% asset_img k9.png %}

<br/>

### 通过java API实现消息生产者，发送消息

```java
import org.junit.Test;

import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;

import java.util.HashMap;
import java.util.Properties;

public class TestProducer {
  @Test
  public void testSend(){
    Properties props = new Properties();
    //broker列表
    props.put("metadata.broker.list", "s202:9092");
    //串行化
    props.put("serializer.class", "kafka.serializer.StringEncoder");
    //
    props.put("request.required.acks", "1");

    //创建生产者配置对象
    ProducerConfig config = new ProducerConfig(props);

    //创建生产者
    Producer<String, String> producer = new Producer<String, String>(config);

    KeyedMessage<String, String> msg = new KeyedMessage<String, String>("test3","100" ,"hello world tomas100");
    producer.send(msg);
    System.out.println("send over!");
  }
}
```

<br/>

### 消息消费者

```java
@Test
public void testConumser(){
  //
  Properties props = new Properties();
  props.put("zookeeper.connect", "s202:2181");
  props.put("group.id", "g3");
  props.put("zookeeper.session.timeout.ms", "500");
  props.put("zookeeper.sync.time.ms", "250");
  props.put("auto.commit.interval.ms", "1000");
  props.put("auto.offset.reset", "smallest");
  //创建消费者配置对象
  ConsumerConfig config = new ConsumerConfig(props);
  //
  Map<String, Integer> map = new HashMap<String, Integer>();
  map.put("test3", new Integer(1));
  Map<String, List<KafkaStream<byte[], byte[]>>> msgs = Consumer.createJavaConsumerConnector(new ConsumerConfig(props)).createMessageStreams(map);
  List<KafkaStream<byte[], byte[]>> msgList = msgs.get("test3");
  for(KafkaStream<byte[],byte[]> stream : msgList){
    ConsumerIterator<byte[],byte[]> it = stream.iterator();
    while(it.hasNext()){
      byte[] message = it.next().message();
      System.out.println(new String(message));
    }
  }
}
```

<br/>



### flume集成kafka

{% asset_img k10.png %}

<br/>

1. KafkaSink

```shell
[生产者]
a1.sources = r1
a1.sinks = k1
a1.channels = c1

a1.sources.r1.type=netcat
a1.sources.r1.bind=localhost
a1.sources.r1.port=8888

a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.k1.kafka.topic = test2
a1.sinks.k1.kafka.bootstrap.servers = mini2:9092
a1.sinks.k1.kafka.flumeBatchSize = 20
a1.sinks.k1.kafka.producer.acks = 1

a1.channels.c1.type=memory

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

<br/>

1. KafkaSource

```shell
[消费者]
a1.sources = r1
a1.sinks = k1
a1.channels = c1

a1.sources.r1.type = org.apache.flume.source.kafka.KafkaSource
a1.sources.r1.batchSize = 5000
a1.sources.r1.batchDurationMillis = 2000
a1.sources.r1.kafka.bootstrap.servers = s202:9092
a1.sources.r1.kafka.topics = test3
a1.sources.r1.kafka.consumer.group.id = g4

a1.sinks.k1.type = logger

a1.channels.c1.type=memory

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

<br/>

1. Channel

```shell
生产者 + 消费者
a1.sources = r1
a1.sinks = k1
a1.channels = c1

a1.sources.r1.type = avro
a1.sources.r1.bind = localhost
a1.sources.r1.port = 8888

a1.sinks.k1.type = logger

a1.channels.c1.type = org.apache.flume.channel.kafka.KafkaChannel
a1.channels.c1.kafka.bootstrap.servers = s202:9092
a1.channels.c1.kafka.topic = test3
a1.channels.c1.kafka.consumer.group.id = g6
a1.channels.c1.parseAsFlumeEvent = false

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

