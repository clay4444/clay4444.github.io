---
title: 关于Hive的Select查询
tags:
  - Hive
categories:
  - big-data
abbrlink: 73b70227
date: 2017-06-25 14:18:00
---

## 基本的Select操作

### 1.语法结构

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ... 
FROM table_reference
[WHERE where_condition] 
[GROUP BY col_list [HAVING condition]] 
[CLUSTER BY col_list 
  | [DISTRIBUTE BY col_list] [SORT BY| ORDER BY col_list] 
] 
[LIMIT number]
```

<br/>

###  2. 注：

*1**、order by **会对输入做全局排序，因此只有一个reducer，会导致当输入规模较大时，需要较长的计算时间。*

*2**、sort by**不是全局排序，其在数据进入reducer**前完成排序。因此，如果用sort by**进行排序，并且设置mapred.reduce.tasks>1**，则sort by**只保证每个reducer的输出有序，不保证全局有序。*

*3**、distribute by**根据指定的字段将数据分到不同的  reducer 。且分发算法是hash散列*

*4**、Cluster by **除了具有Distribute  by的功能外，还会对该字段进行排序。因此，常常认为cluster by = distribute by + sort by*

<br/>

​	因此，如果分桶（hive里面的分桶和mr里面的分区是一个概念，hive里面的分区(partition)的概念只是单纯的把数据分成几个文件夹存储，并没有进行真正的shuffle）和sort是一个字段，可以直接采用cluser by = distribute by + sort by 

​	分桶的意义就在于提高join的效率，而且在往分桶表里插入数据的时候，是要插入已经分好桶的数据，所以针对分桶表不能load数据。

<br/>

​	distribute by的作用就是根据字段进行分区，也就是mapreduce过程中map阶段的指定的分区的分区过程的getParation(key, value, numreducetask)  ，这里可以回顾下mapreduce的过程。

<br/>

​	(思考这个问题，Select a.id, a.name b.address from a join b on a.id = b.id

如果a表和b表已经是分桶表，而且分桶的字段是id，那么做这个join操作时，还需要全表做笛卡尔积吗？)

<br/>

​	肯定不会了，因为这两张表相同的ID肯定在同一个桶中，因为他们的hashcode % 桶 的值  永远会进到同一个桶中，这时只需要每个桶内进行join即可，

​	**分桶的本质意义就是提高join的效率**

<br/>

### 2.具体实例

1、获取年龄大的3个学生

{% asset_img 获取年龄大的3个学生.png %}

<br/>

2、查询学生信息按年龄，降序排序。

{% asset_img 查询学生信息按年龄，降序排序.png %}

{% asset_img 查询学生信息按年龄，降序排序2.png %}

{% asset_img 查询学生信息按年龄，降序排序3.png %}

<br/>

3、按学生名称汇总学生年龄。

{% asset_img 按学生名称汇总学生年龄.png %}