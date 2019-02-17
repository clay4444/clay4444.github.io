---
title: Hive最常用之统计报表
tags:
  - Hive
categories:
  - big-data
abbrlink: e01e83db
date: 2017-06-20 15:21:05
---

## Hive 数据处理技巧之 统计报表

#### <br/>

#### 		问题：求每个月以来每个员工的累计金额

<br/>

### 1. 准备数据

~~~sql
create table t_access_times(username string,month string,salary int)
row format delimited fields terminated by ',';
~~~

~~~sql
load data local inpath '/home/hdfs/t_access_times.dat' into table t_access_times;
~~~

数据：

~~~
A,2015-01,5
A,2015-01,15
B,2015-01,5
A,2015-01,8
B,2015-01,25
A,2015-01,5
A,2015-02,4
A,2015-02,6
B,2015-02,10
B,2015-02,5
~~~

<br/>

### 2.求每个用户的月总金额

~~~sql
select username,month,sum(salary) as salary from t_access_times group by username,month;
~~~

数据：

~~~sql
+-----------+----------+---------+--+
| username  |  month   | salary  |
+-----------+----------+---------+--+
| A         | 2015-01  | 33      |
| A         | 2015-02  | 10      |
| B         | 2015-01  | 30      |
| B         | 2015-02  | 15      |
+-----------+----------+---------+--+
~~~

<br/>

### 3.将月总金额表 自己连接 自己连接 ( join之后的结果 )

~~~
+-------------+----------+-----------+-------------+----------+-----------+--+
| a.username  | a.month  | a.salary  | b.username  | b.month  | b.salary  |
+-------------+----------+-----------+-------------+----------+-----------+--+
| A           | 2015-01  | 33        | A           | 2015-01  | 33        |
| A           | 2015-01  | 33        | A           | 2015-02  | 10        |
| A           | 2015-02  | 10        | A           | 2015-01  | 33        |
| A           | 2015-02  | 10        | A           | 2015-02  | 10        |
| B           | 2015-01  | 30        | B           | 2015-01  | 30        |
| B           | 2015-01  | 30        | B           | 2015-02  | 15        |
| B           | 2015-02  | 15        | B           | 2015-01  | 30        |
| B           | 2015-02  | 15        | B           | 2015-02  | 15        |
+-------------+----------+-----------+-------------+----------+-----------+--+
~~~

<br/>

### 4.从上一步的结果中，进行分组查询，分组的字段是a.username a.month，求月累计值：  将b.month <= a.month的所有b.salary求和即可

~~~sql
select A.username,A.month,max(A.salary) as salary,sum(B.salary) as accumulate
from 
(select username,month,sum(salary) as salary from t_access_times group by username,month) A 
inner join 
(select username,month,sum(salary) as salary from t_access_times group by username,month) B
on
A.username=B.username
where B.month <= A.month
group by A.username,A.month
order by A.username,A.month;
~~~

结果：

~~~
A       2015-01 33      33

A       2015-02 10      43

B       2015-01 30      30

B       2015-02 15      45
~~~



