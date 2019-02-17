---
title: Hive常见优化
tags:
  - Hive
categories:
  - big-data
abbrlink: 22d66bb6
date: 2018-06-04 20:22:36
---

### Join优化

mr实现的过程分为map端join，和reduceJoin，两种方式，这两种方式在之前的mr介绍的论文中都有详细描述，这里不再赘述，请参考MapReduce tag标签下的内容

Hql 底层默认执行join的过程中，在 Join 操作的 Reduce 阶段，位于 Join 操作符左边的表的内容会被加载进内存 ，所以，

**将条目少的表/子查询放在 Join 操作符的左边**  可以有效减少发生内存溢出错误的几率。 

其实从底层实现上来说，就是mr的分布式缓存。



<br/>

### group by 优化

Map端聚合，首先在map端进行初步聚合，最后在reduce端得出最终结果，相关参数：

```properties
//是否在 Map 端进行聚合，默认为 True 
hive.map.aggr = true
```

**数据倾斜的聚合优化** 对数据进行聚合优化，可以进行如下的参数设置 

```properties
hive.groupby.skewindata = true 
```

<br/>

起到至关重要的作用的其实是第二个参数的设置，它使计算变成了两个mapreduce，先在第一个中在 shuffle 过程 partition 时随机给 key 打标记，使每个key 随机均匀分布到各个 reduce 上计算，但是这样只能完成部分计算，因为相同key没有分配到相同reduce上，所以需要第二次的mapreduce,这次就回归正常 shuffle,但是数据分布不均匀的问题在第一次mapreduce已经有了很大的改善，因此基本解决数据倾斜。



<br/>

### 合并小文件

文件数目过多，会给 HDFS 带来压力，并且会影响处理效率，可以通过合并 Map 和 Reduce 的结果文件来消除这样的影响： 

```properties
// 是否和并 Map 输出文件，默认为 True
hive.merge.mapfiles = true
// 是否合并 Reduce 输出文件，默认为 False
hive.merge.mapredfiles = false
// 合并文件的大小
hive.merge.size.per.task = 256*1000*1000
```



<br/>

### Hive实现(not) in

```sql
select a.key from a left outer join b on a.key=b.key where b.key1 is null
```



<br/>

### 排序优化

**order By**：  实现全局排序，一个reduce实现，效率低 

**sort by**： 实现部分有序，如果用sort by进行排序，并且设置mapred.reduce.tasks >  1，则sort by只保证每个reducer的输出有序，不保证全局有序。单个reduce输出的结果是有序的，效率高。 

通常和**DISTRIBUTE BY**关键字一起使用（DISTRIBUTE BY关键字 可以指定map 到 reduce端的分发key）

 **CLUSTER BY col1 等价于 DISTRIBUTE BY col1 SORT BY col1**

使用sort by 你可以指定执行的reduce 个数 （ `set mapred.reduce.tasks=<number>`）,对输出的数据再执行归并排序，即可以得到全部结果。  

<br/>

**distribute by: ** 按照指定的字段对数据进行划分到不同的输出reduce / 文件中。 

```sql
insert overwrite local directory '/home/hadoop/out' 
select * from test order by name distribute by length(name); 
```

此方法会根据name的长度划分到不同的reduce中，最终输出到不同的文件中。

 <br/>

length 是内建函数，也可以指定其他的函数或这使用自定义函数。 

**Cluster By: ** cluster by 除了具有 distribute by 的功能外还兼具 sort by 的功能。 但是排序只能是倒序排序，不能指定排序规则为 asc 或者 desc。 

<br/>

**distribute by ** 的作用其实是和分捅表关联的：创建表时，指定**clustered  by**

对于每一个表（table）或者分区， Hive可以进一步组织成桶，也就是说**桶是更为细粒度的数据范围划分**。Hive也是 针对某一列进行桶的组织。Hive采用**对列值哈希，然后除以桶的个数求余的方式**决定该条记录存放在哪个桶当中。 

把表（或者分区）组织成桶（Bucket）有两个理由：

 （1）获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接 （Map-side join）高效的实现。比如JOIN操作。对于JOIN操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行JOIN操作就可以，可以大大较少JOIN的数据量。

（2）使取样（sampling）更高效。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。

 

 

<br/>

### 少用COUNT DISTINCT

数据量小的时候无所谓，数据量大的情况下，由于COUNT DISTINCT操作需要用一个Reduce Task来完成，这一个Reduce需要处理的数据量太大，就会导致整个Job很难完成，一般COUNT DISTINCT使用先GROUP BY再COUNT的方式替换： 

```sql
SELECT day, COUNT(DISTINCT id) AS uv  FROM lxw1234  GROUP BY day
```

**可以转换成：** 

```sql
SELECT day, COUNT(id) AS uv FROM (SELECT day,id FROM lxw1234 GROUP BY day,id) a GROUP BY day;
```

虽然会多用一个Job来完成，但在数据量大的情况下，这个绝对是值得的。

 

<br/>

### 是否存在多对多的关联

只要遇到表关联，就必须得调研一下，是否存在多对多的关联，起码得保证有一个表或者结果集的关联键不重复。

 如果某一个关联键的记录数非常多，那么分配到该Reduce Task中的数据量将非常大，导致整个Job很难完成，甚至根本跑不出来。

还有就是避免笛卡尔积，同理，如果某一个键的数据量非常大，也是很难完成Job的。