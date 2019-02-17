---
title: Spark简介及wc实现
tags:
  - Spark
categories:
  - big-data
abbrlink: b5801fa4
date: 2018-03-09 10:37:47
---

### Spark模块

Spark core		//核心模块

Spark SQL		//SQL

Spark Streaming	//流计算

Spark MLlib		//机器学习

Spark graph		//图计算

<br/>

### DAG

direct acycle graph,有向无环图。

<br/>

### spark实现word count

{% asset_img s1.png %}

<br/>

//加载文本文件,以换行符方式切割文本.Array(hello  world2,hello world2 ,...)

```scala
val rdd1 = sc.textFile("/home/centos/test.txt");
```

<br/>

//单词统计1

```shell
$scala>val rdd1 = sc.textFile("/home/hadoop/test.txt")

$scala>val rdd2 = rdd1.flatMap(line=>line.split(" "))		//压扁 /  炸裂

$scala>val rdd3 = rdd2.map(word => (word,1))			//变换成对偶 （key， 1）

$scala>val rdd4 = rdd3.reduceByKey(_ + _)

$scala>rdd4.collect
```

<br/>

//单词统计2

```scala
sc.textFile("/home/hadoop/test.txt").flatMap(.split(" ")).map((,1)).reduceByKey(_ + _).collect
```

<br/>

//统计所有含有wor字样到单词个数。filter

//过滤单词

```scala
sc.textFile("/home/hadoop/test.txt").flatMap(.split(" ")).filter(.contains("wor")).map((,1)).reduceByKey( + _).collect
```



<br/>

### [API]

SparkContext:

Spark功能的主要入口点。代表到Spark集群的连接，可以创建RDD、累加器和广播变量.

每个JVM只能激活一个SparkContext对象，在创建sc之前需要stop掉active的sc。

<br/>

SparkConf:

spark配置对象，设置Spark应用各种参数，kv形式。

<br/>



### scala 版本单词统计

1. 创建Scala模块,并添加pom.xml，

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.it18zhang</groupId>
  <artifactId>SparkDemo1</artifactId>
  <version>1.0-SNAPSHOT</version>
  <dependencies>
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-core_2.11</artifactId>
      <version>2.1.0</version>
    </dependency>
  </dependencies>
</project>
```

<br/>

1. 编写scala文件

```scala
import org.apache.spark.{SparkConf, SparkContext}

object WordCountDemo {
  def main(args: Array[String]): Unit = {
    //创建Spark配置对象
    val conf = new SparkConf();
    conf.setAppName("WordCountSpark")
    //设置master属性
    conf.setMaster("local") ;

    //通过conf创建sc
    val sc = new SparkContext(conf);

    //加载文本文件
    val rdd1 = sc.textFile("d:/scala/test.txt");
    //压扁
    val rdd2 = rdd1.flatMap(line => line.split(" ")) ;
    //映射w => (w,1)
    val rdd3 = rdd2.map((_,1))
    val rdd4 = rdd3.reduceByKey(_ + _)
    v	
    r.foreach(println)
  }
}
```

<br/>



### java版单词统计

```java
import org.apache.spark.SparkConf;
import org.apache.spark.SparkContext;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

/**
	 * java版
	 */
public class WordCountJava2 {
  public static void main(String[] args) {
    //创建SparkConf对象
    SparkConf conf = new SparkConf();
    conf.setAppName("WordCountJava2");
    conf.setMaster("local");

    //创建java sc
    JavaSparkContext sc = new JavaSparkContext(conf);
    //加载文本文件
    JavaRDD<String> rdd1 = sc.textFile("d:/scala//test.txt");

    //压扁	匿名内部类
    JavaRDD<String> rdd2 = rdd1.flatMap(new FlatMapFunction<String, String>() {
      public Iterator<String> call(String s) throws Exception {
        List<String> list = new ArrayList<String>();
        String[] arr = s.split(" ");
        for(String ss :arr){
          list.add(ss);
        }
        return list.iterator();
      }
    });

    //映射,word -> (word,1)
    JavaPairRDD<String,Integer> rdd3 = rdd2.mapToPair(new PairFunction<String, String, Integer>() {
      public Tuple2<String, Integer> call(String s) throws Exception {
        return new Tuple2<String, Integer>(s,1);
      }
    });

    //reduce化简
    JavaPairRDD<String,Integer> rdd4 = rdd3.reduceByKey(new Function2<Integer, Integer, Integer>() {
      public Integer call(Integer v1, Integer v2) throws Exception {
        return v1 + v2;
      }
    });

    //
    List<Tuple2<String,Integer>> list = rdd4.collect();
    for(Tuple2<String, Integer> t : list){
      System.out.println(t._1() + " : " + t._2());
    }
  }
}
```

<br/>

### Spark集群模式

1. local


1. standalone

<br/>

### 脚本分析

[start-all.sh]

```shell
sbin/spark-config.sh

sbin/spark-master.sh		//启动master进程

sbin/spark-slaves.sh		//启动worker进程
```

<br/>

[start-master.sh]

```shell
sbin/spark-config.sh

org.apache.spark.deploy.master.Master

spark-daemon.sh start org.apache.spark.deploy.master.Master --host --port --webui-port ...
```

<br/>

[spark-slaves.sh]

```shell
sbin/spark-config.sh

slaves.sh				//conf/slaves
```

<br/>

[slaves.sh]

```shell
for conf/slaves{

	ssh host start-slave.sh ...

}
```

<br/>

[start-slave.sh]

```shell
CLASS="org.apache.spark.deploy.worker.Worker"

sbin/spark-config.sh

for ((  .. )) ; do

	start_instance (( 1 + i )) "$@"

done 

```

<br/>

$>cd /soft/spark/sbin

$>./stop-all.sh				//停掉整个spark集群.

$>./start-master.sh			//启动master节点

$>./start-slaves.sh			//启动所有worker节点

$>./stop-slaves.sh			 //停掉所有worker节点

$>./stop-slave.sh				//停掉一个worker节点

$>./start-slave.sh				//开启一个worker节点