---
title: sql语句中where与having的区别
tags:
  - MySql
abbrlink: b8ce2b2d
date: 2018-11-03 20:44:20
---

#### where

Where 是一个**约束**声明，使用Where约束来自数据库的数据，

1. Where是在结果返回之前起作用的，
2. Where中不能使用聚合函数。 

<br/>

#### having

Having 是一个**过滤**声明，

1. 是在查询返回结果集以后对查询结果进行的过滤操作，
2. 在Having中可以使用聚合函数。 

<br/>

下面用一个例子进一步说明问题。假设有数据表：

```sql
CREATE TABLE  `test`.`salary_info` (
  `id` int(10) unsigned NOT NULL auto_increment,
  `deparment` varchar(16) NOT NULL default '',
  `name` varchar(16) NOT NULL default '',
  `salary` int(10) unsigned NOT NULL default '0',
   PRIMARY KEY  (`id`)
) ENGINE=MyISAM AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;
```

<br/>

**例1：要查找平均工资大于3000的部门**

则sql语句应为：

```sql
select deparment, avg(salary) as average from salary_info 
group by deparment having average > 3000
```

此时只能使用having，而不能使用where。一来，我们要使用聚合语句avg；二来，我们要对聚合**后**的结果进行筛选（average > 3000），因此使用where会被告知sql有误。

<br/>

**例2：要查询每个部门工资大于3000的员工个数** 

sql语句应为：

```sql
select deparment, count(*) as c from salary_info 
where salary > 3000 group by deparment
```

<br/>

此处的where不可用having进行替换，因为是直接对库中的数据进行筛选，而非对结果集进行筛选。