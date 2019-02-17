---
title: Storm和kafka、Hbase之间的整合
tags:
  - Storm
categories:
  - big-data
abbrlink: d5a66bc5
date: 2018-03-08 18:21:15
---

### kafka + storm

1. 描述·

storm以消费者从kafka队列中提取消息。

<br/>

1. 添加storm-kafka依赖项

```xml
[pom.xml]
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.it18zhang</groupId>
  <artifactId>StormDemo</artifactId>
  <version>1.0-SNAPSHOT</version>

  <dependencies>
    <dependency>
      <groupId>org.apache.storm</groupId>
      <artifactId>storm-core</artifactId>
      <version>1.0.3</version>
    </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
    </dependency>
    <dependency>
      <groupId>org.apache.storm</groupId>
      <artifactId>storm-kafka</artifactId>
      <version>1.0.2</version>
    </dependency>
    <dependency>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
      <version>1.2.17</version>
    </dependency>
    <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka_2.10</artifactId>
      <version>0.8.1.1</version>
      <exclusions>
        <exclusion>
          <groupId>org.apache.zookeeper</groupId>
          <artifactId>zookeeper</artifactId>
        </exclusion>
        <exclusion>
          <groupId>log4j</groupId>
          <artifactId>log4j</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
  </dependencies>

</project>
```

<br/>

2. 启动kafka + storm集群.

```java
public class App {
  public static void main(String[] args) throws Exception {
    TopologyBuilder builder = new TopologyBuilder();

    //zk连接串
    String zkConnString = "s202:2181" ;
    //
    BrokerHosts hosts = new ZkHosts(zkConnString);
    //Spout配置
    SpoutConfig spoutConfig = new SpoutConfig(hosts, "test2", "/test2", UUID.randomUUID().toString());
    spoutConfig.scheme = new SchemeAsMultiScheme(new StringScheme());
    KafkaSpout kafkaSpout = new KafkaSpout(spoutConfig);

    builder.setSpout("kafkaspout", kafkaSpout).setNumTasks(2);
    builder.setBolt("split-bolt", new SplitBolt(),2).shuffleGrouping("kafkaspout").setNumTasks(2);

    Config conf = new Config();
    conf.setNumWorkers(2);
    conf.setDebug(true);

    /**
	* 本地模式storm
	*/
    LocalCluster cluster = new LocalCluster();
    cluster.submitTopology("wc", conf, builder.createTopology());
  }
}
```

<br/>

3. 启动kafka集群和storm,使用生产者发送消息给kafka。

<br/>

4. 看storm是否消费到。

<br/>



### Hbase + storm

两种方案：

<br/>

第一种：第二个bolt使用HbaseBolt，第一个bolt完成之后，通过shuffe进行分组，然后直接进入Hbase，此时不会产生数据倾斜的问题，而且数据都是直接进入Hbase进行聚合，incr操作也是原子操作，所以最后的结果肯定也是正确的。但是会频繁的访问Hbase数据库，降低性能。

<br/>

第二种：第二个HbaseBolt 之前，再加一个bolt进行聚合，聚合完成之后在进入Habse，提供了Hbase整体的访问性能。但是有可能出现数据倾斜的问题，注意在HbaseBolt 聚合的时候就不是1了即可，最后的结果也都是正确的。

<br/>

1. 引入pom.xml依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.it18zhang</groupId>
  <artifactId>StormDemo</artifactId>
  <version>1.0-SNAPSHOT</version>

  <dependencies>
    <dependency>
      <groupId>org.apache.storm</groupId>
      <artifactId>storm-core</artifactId>
      <version>1.0.3</version>
    </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
    </dependency>
    <dependency>
      <groupId>org.apache.storm</groupId>
      <artifactId>storm-kafka</artifactId>
      <version>1.0.2</version>
    </dependency>
    <dependency>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
      <version>1.2.17</version>
    </dependency>
    <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka_2.10</artifactId>
      <version>0.8.1.1</version>
      <exclusions>
        <exclusion>
          <groupId>org.apache.zookeeper</groupId>
          <artifactId>zookeeper</artifactId>
        </exclusion>
        <exclusion>
          <groupId>log4j</groupId>
          <artifactId>log4j</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.apache.storm</groupId>
      <artifactId>storm-hbase</artifactId>
      <version>1.0.3</version>
    </dependency>
    <dependency>
      <groupId>org.apache.hbase</groupId>
      <artifactId>hbase-client</artifactId>
      <version>1.2.3</version>
    </dependency>
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-common</artifactId>
      <version>2.7.3</version>
    </dependency>
  </dependencies>

</project>
```

<br/>

2. HbaseBolt

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Table;
import org.apache.hadoop.hbase.util.Bytes;
import org.apache.storm.shade.org.apache.http.conn.HttpConnectionFactory;
import org.apache.storm.task.OutputCollector;
import org.apache.storm.task.TopologyContext;
import org.apache.storm.topology.IRichBolt;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.tuple.Tuple;

import java.io.IOException;
import java.util.Map;

/**
		 * HbaseBolt,写入数据到hbase库中。
		 */
public class HbaseBolot implements IRichBolt {

  private Table t ;
  public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
    try {
      Configuration conf = HBaseConfiguration.create();
      Connection conn = ConnectionFactory.createConnection(conf);
      TableName tname = TableName.valueOf("ns1:wordcount");
      t = conn.getTable(tname);
    } catch (IOException e) {
      e.printStackTrace();
    }

  }

  public void execute(Tuple tuple) {
    String word = tuple.getString(0);
    Integer count = tuple.getInteger(1);
    //使用hbase的increment机制进行wordcount
    byte[] rowkey = Bytes.toBytes(word);
    byte[] f = Bytes.toBytes("f1");
    byte[] c = Bytes.toBytes("count");
    try {
      t.incrementColumnValue(rowkey,f,c,count);
    } catch (IOException e) {
    }
  }

  public void cleanup() {
  }

  public void declareOutputFields(OutputFieldsDeclarer declarer) {
  }

  public Map<String, Object> getComponentConfiguration() {
    return null;
  }
}
```

<br/>

3. 复制配置hbase配置文件到resources下

[resources]

hbase-site.xml

hdfs-site.xml

<br/>

4. 启动hbase集群 + storm。

<br/>

5. 查看 hbase 表数据

```shell
$hbase>get_counter 'ns1:wordcount' , 'word' , 'f1:count'
```