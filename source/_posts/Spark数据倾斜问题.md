---
title: Spark数据倾斜问题
tags:
  - Spark
categories:
  - big-data
abbrlink: 1761f4b4
date: 2018-03-09 11:26:39
---

### spark数据倾斜问题：

{% asset_img s1.png %}

<br/>

```java
/**
  * 数据倾斜问题
  */
object DataLeanDemo1 {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf()
      conf.setAppName("WordCountScala")
      conf.setMaster("local[4]") ;
    val sc = new SparkContext(conf)
      val rdd1 = sc.textFile("d:/scala/test.txt",4)
      rdd1.flatMap(_.split(" ")).map((_,1)).map(t=>{
      val word = t._1
        val r = Random.nextInt(100)
        (word + "_" + r,1)          //hello_30，1
    }).reduceByKey(_ + _,4).map(t=>{
      val word = t._1;
      val count = t._2;           //hello_30，30
      val w = word.split("_")(0)
        (w,count)                   //hello，30
    }).reduceByKey(_ + _,4).saveAsTextFile("d:/scala/out/lean");
  }
}
```