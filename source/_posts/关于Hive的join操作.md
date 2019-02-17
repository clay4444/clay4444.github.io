---
title: 关于Hive的join操作
tags:
  - Hive
categories:
  - big-data
abbrlink: e4408e09
date: 2017-06-13 14:38:13
---

## Hive Join

### 1.基本特性：

Hive支持等值连接（equalityjoins）、外连接（outer joins）和（left/right joins）。Hive **不支持非等值的连接**，因为非等值连接非常难转化到 map/reduce 任务。

另外，Hive 支持多于 2 个表的连接。

<br/>

### 2.只支持等值join

例如： 

~~~sql
SELECT a.* FROMa JOIN b ON (a.id = b.id)
SELECT a.* FROM a JOIN b
ON (a.id = b.id AND a.department =b.department)
~~~

是正确的，然而:

~~~sql
SELECT a.* FROM a JOIN b ON (a.id>b.id)
~~~

是错误的。

<br/>

### 3. 可以 join 多于 2 个表

例如

  ~~~sql
SELECT a.val,b.val, c.val FROM a JOIN b
ON (a.key =b.key1) JOIN c ON (c.key = b.key2)
  ~~~

如果join中多个表的join key 是同一个，则 join 会被转化为单个map/reduce 任务，例如：

~~~sql
SELECT a.val,b.val, c.val FROM a JOIN b
ON (a.key =b.key1) JOIN c
ON (c.key =b.key1)
~~~

被转化为单个 map/reduce 任务，因为 join 中只使用了 b.key1 作为 join key。

~~~sql
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key =b.key1)
JOIN c ON(c.key = b.key2)
~~~

而这一 join 被转化为2 个 map/reduce 任务。因为 b.key1 用于第一次 join 条件，而 b.key2 用于第二次 join。

<br/>

### 4.join 时，每次 map/reduce 任务的逻辑：

reducer 会缓存 join 序列中除了最后一个表的所有表的记录，再通过最后一个表将结果序列化到文件系统。这一实现有助于在 reduce 端减少内存的使用量。实践中，应该 **把最大的那个表写在最后**（否则会因为缓存浪费大量内存）。例如：

 ~~~sql
SELECT a.val, b.val, c.val FROM a
JOIN b ON (a.key = b.key1) JOIN c ON (c.key= b.key1)
 ~~~

所有表都使用同一个 join key（使用 1 次 map/reduce 任务计算）。Reduce 端会缓存 a 表和 b 表的记录，然后每次取得一个 c 表的记录就计算一次 join 结果，类似的还有：

  ~~~sql
SELECT a.val,b.val, c.val FROM a
JOIN b ON(a.key = b.key1) JOIN c ON (c.key = b.key2)
  ~~~

这里用了 2 次map/reduce 任务。第一次缓存 a 表，用 b 表序列化；第二次缓存第一次 map/reduce 任务的结果，然后用 c 表序列化。

<br/>

### 5.LEFT，RIGHT 和 FULL OUTER 关键字用于处理 join 中空记录的情况

例如：

~~~sql
SELECT a.val,b.val FROM 
a LEFT OUTER  JOIN b ON (a.key=b.key)
~~~

对应所有 a 表中的记录都有一条记录输出。输出的结果应该是 a.val, b.val，当 a.key=b.key 时，而当 b.key 中找不到等值的 a.key 记录时也会输出:

a.val, NULL

所以 a 表中的所有记录都被保留了；

“a RIGHT OUTER JOIN b”会保留所有 b 表的记录。

<br/>

### 6. Join 发生在 WHERE 子句之前

如果你想限制 join 的输出，应该在 WHERE 子句中写过滤条件——或是在join 子句中写。这里面一个容易混淆的问题是表分区的情况：

 ~~~sql
SELECT a.val,b.val FROM a
LEFT OUTER JOIN b ON (a.key=b.key)
WHERE a.ds='2009-07-07' AND b.ds='2009-07-07'
 ~~~

会 join a 表到 b 表（OUTER JOIN），列出 a.val 和 b.val 的记录。WHERE 从句中可以使用其他列作为过滤条件。但是，如前所述，如果 b 表中找不到对应 a 表的记录，b 表的所有列都会列出 NULL，**包括 ds 列**。也就是说，join 会过滤 b 表中不能找到匹配a 表 join key 的所有记录。这样的话，LEFT OUTER 就使得查询结果与 WHERE 子句无关了。解决的办法是在 OUTER JOIN 时使用以下语法：

~~~sql
SELECT a.val,b.val FROM a LEFT OUTER JOIN b
ON (a.key=b.key AND b.ds='2009-07-07' AND a.ds='2009-07-07')
~~~

这一查询的结果是预先在 join 阶段过滤过的，所以不会存在上述问题。这一逻辑也可以应用于 RIGHT 和 FULL 类型的join 中。