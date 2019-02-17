---
title: SparkRDD弹性分布式数据集简介及RDD变换
tags:
  - Spark
categories:
  - big-data
abbrlink: 3fb5d1b7
date: 2018-03-09 11:23:08
---

### RDD:

HDFS是物理切分

MR是逻辑切分

是spark的基本数据结构，是不可变数据集。RDD中的数据集进行逻辑分区，每个分区可以单独在集群节点进行计算。可以包含任何java,scala，python和自定义类型。

<br/>

{% asset_img s1.png %}

<br/>

RDD是只读的记录分区集合。RDD具有容错机制。

<br/>

创建RDD方式，一、并行化一个现有集合。

<br/>

hadoop 花费90%时间用于rw。

 多次MR的时候，前面的一次产生的结果一定要保存到磁盘中，极大程度上降低了速度。

{% asset_img s2.png %}

<br/>

内存处理计算。在job间进行数据共享。内存的IO速率高于网络和disk的10 ~ 100之间。

<br/>

{% asset_img s3.png %}

<br/>

1. 分布式内存：在内存地址上进行对齐，存储中间结果
   1. 所以在使用机器学习进行大数据量的回归，聚类的时候，一定会进行大量数据的迭代，此时计算量会非常巨大，使用spark的分布式内存就会有很大优势， 
2. 但是如果仅仅是数据存储，随机访问，实时读写，那么Hbase足够。
3. 如果仅仅是数据的离线分析，使用Hive即可。定时调度。
4. 实时推荐：spark  /  storm


<br/>

**每次计算都会产生一个新的RDD**

<br/>



### 内部包含5个主要属性

1. 分区列表
2. 针对每个split的计算函数。

Java：是从对象出发，在迭代过程期间，操作数据，**函数是主体**，在函数中处理数据，数据来回传递

scala：**数据是主体**，函数来回传，函数是计算法则，分布式计算，每个节点上的计算可能不同。

3. 对其他rdd的依赖列表


4. 可选，如果是KeyValueRDD的话，可以带分区类。


5. 可选，首选块位置列表(hdfs block location);本地优先策略

<br/>



### RDD变换

<br/>

**返回指向新rdd的指针**，在rdd之间创建依赖关系。每个rdd都有计算函数和指向父RDD的指针。

<br/>

1. map()	

对每个元素进行变换，应用变换函数

2. filter()	

过滤器,(T)=>Boolean

3. flatMap()

压扁,T => TraversableOnce[U]

4. mapPartitions()		

对每个分区进行应用变换，输入的Iterator,返回新的迭代器，可以对分区进行函数处理。

Iterator<T> => Iterator<U>

5. mapPartitionsWithIndex(func)	

同上，(Int, Iterator<T>) => Iterator<U>

6. sample(withReplacement, fraction, seed)

采样返回采样的RDD子集。withReplacement 元素是否可以多次采样.fraction : 期望采样数量.[0,1]

7. union()

类似于mysql union操作。

**符合两个条件的行都会查出来，但是只会显示一个**

select * from persons where id < 10 union select * from id persons where id > 29 ;

8. intersection

交集,提取两个rdd中都含有的元素。

9. distinct([numTasks]))	

去重,去除重复的元素。

10. groupByKey()

{% asset_img s4.png %}

{% asset_img s5.png %}

11. reduceByKey(*)

按key聚合。 

12. aggregateByKey(zeroValue)(seqOp, combOp, [numTasks])

按照key进行聚合key:String U:Int = 0

13. sortByKey

排序

14. join(otherDataset, [numTasks])

{% asset_img s6.png %}

{% asset_img s7.png %}

{% asset_img s8.png %}

15. cogroup

协分组：(K,V).cogroup(K,W) =>(K,(Iterable<V>,Iterable<!-- <W> -->)) 

{% asset_img s9.png %}

{% asset_img s10.png %}

{% asset_img s11.png %}

16. cartesian(otherDataset)					

//笛卡尔积,RR[T] RDD[U] => RDD[(T,U)]

17. pipe	

将rdd的元素传递给脚本或者命令，执行结果返回形成新的RDD

18. coalesce(numPartitions)	

减少分区

19. repartition

可增可减

20. repartitionAndSortWithinPartitions(partitioner)

再分区并在分区内进行排序

<br/>

### RDD Action

1. collect()

收集rdd元素形成数组.

2. count()

统计rdd元素的个数

3. reduce()	

聚合,返回一个值。

4. first

取出第一个元素take(1)

5. take

first就是用take实现的。

6. takeSample (withReplacement,num, [seed])
7. takeOrdered(n, [ordering])
8. saveAsTextFile(path)		

保存到文件

9. saveAsSequenceFile(path)

保存成序列化文件(压缩文件)

10. saveAsObjectFile(path) (Java and Scala)
11. countByKey()

按照key,统计每个key下value的个数.