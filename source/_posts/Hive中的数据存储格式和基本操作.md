---
title: Hive中的数据存储格式和基本操作
tags:
  - Hive
categories:
  - big-data
abbrlink: f92012d0
date: 2017-06-23 17:05:21
---

### 一、Hive的数据存储

1、Hive中所有的数据都存储在**HDFS** 中，没有专门的数据存储格式（可支持Text，SequenceFile，ParquetFile，RCFILE等）

2、只需要在创建表的时候告诉 Hive 数据中的列分隔符和行分隔符，Hive 就可以解析数据。

3、Hive 中包含以下数据模型：DB、Table，External Table，Partition，Bucket。

- db：在hdfs中表现为${hive.metastore.warehouse.dir}目录下一个文件夹

- table：在hdfs中表现所属db目录下一个文件夹

- external table：外部表, 与table类似，不过其数据存放位置可以在任意指定路径

  普通表: 删除表后, hdfs上的文件都删了

  External外部表删除后, hdfs上的文件没有删除, 只是把文件删除了

- partition：在hdfs中表现为table目录下的子目录，

  应用场景: 查询今天的订单表，可以把今天的订单表和历史的订单表存成几个子目录

- bucket：桶, 在hdfs中表现为同一个表目录下根据hash散列之后的多个文件, 会根据不同的文件把数据放到不同的文件中

  有多少个bucket就有多少个reduce，方便 **join操作** [ 本质 ]

  <br/>

### 二、Hive基本操作

#### 1. 创建表

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
   [(col_name data_type [COMMENT col_comment], ...)] 
   [COMMENT table_comment] 
   [PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
   [CLUSTERED BY (col_name, col_name, ...) 
   [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
   [ROW FORMAT row_format] 
   [STORED AS file_format] 
   [LOCATION hdfs_path]
```

说明：

1、CREATE TABLE 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXISTS 选项来忽略这个异常。

<br/>

2、EXTERNAL关键字可以让用户创建一个外部表，在建表的同时指定一个指向实际数据的路径（LOCATION），Hive 创建内部表时，会将数据移动到数据仓库指向的路径；若创建外部表，仅记录数据所在的路径，不对数据的位置做任何改变。在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。

<br/>

3、LIKE 允许用户复制现有的表结构，但是不复制数据。

<br/>

4、ROW FORMAT 

DELIMITED 【 FIELDS TERMINATED BY char 】【COLLECTION ITEMS TERMINATED BY char】 

​       【MAP KEYS TERMINATED BY char】【LINES TERMINATED BY char]】

​     | SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value,property_name=property_value, ...)]

用户在建表的时候可以自定义 SerDe 或者使用自带的 SerDe。如果没有指定 ROW FORMAT 或者 ROW FORMAT DELIMITED，将会使用自带的 SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的SerDe，Hive通过 SerDe 确定表的具体的列的数据。

<br/>

5、 STORED AS 

SEQUENCEFILE|TEXTFILE|RCFILE

如果文件数据是纯文本，可以使用 STORED AS TEXTFILE。如果数据需要压缩，使用 STORED AS
SEQUENCEFILE。

<br/>

6、CLUSTERED BY

对于每一个表（table）或者分区， Hive可以进一步组织成桶，也就是说桶是更为细粒度的数据范围划分。Hive也是 针对某一列进行桶的组织。Hive采用对列值哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶当中。

把表（或者分区）组织成桶（Bucket）有两个理由：

（1）获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接 （Map-side join）高效的实现。比如JOIN操作。对于JOIN操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行JOIN操作就可以，可以大大较少JOIN的数据量。

（2）使取样（sampling）更高效。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。

<br/>

##### 1.1 具体实例

1、 创建内部表mytable。

{% asset_img 内部表.png %}

2、 创建外部表pageview。

{% asset_img 外部表pageview.png %}

3、创建分区表invites。

{% asset_img 分区表invites.png %}

4、 创建带桶的表student。

{% asset_img 带桶的表student.png %}

<br/>

#### 2. 修改表

<br/>

##### 2.1 修改 / 删除 分区

##### 2.1.1 语法结构

ALTER TABLE table_name ADD [IF NOT EXISTS] partition_spec [ LOCATION'location1' ] partition_spec [ LOCATION 'location2' ] ...

partition_spec:

: PARTITION (partition_col = partition_col_value, partition_col =partiton_col_value, ...)

ALTER TABLE table_name DROP partition_spec, partition_spec,...

##### 2.1.2 具体实例

{% asset_img 修改分区1.png %}

{% asset_img 修改分区2.png %}

<br/>

##### 2.2 重命名表

##### 2.2.1 语法结构

```sql
ALTER TABLE table_name RENAME TO new_table_name
```

##### 2.2.2 具体实例

{% asset_img 修改表名.png %}

<br/>

##### 2.3 增加/更新列

##### 2.3.1 语法结构

```sql
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...) 
```

*注：ADD**是代表新增一字段，字段位置在所有列后面(partition**列前)**，REPLACE**则是表示替换表中所有字段。*

##### 2.3.2  具体实例

{% asset_img 修改列.png %}

<br/>

<br/>

#### 3. DML操作

##### 3.1 Load

##### 3.1.1  语法结构

```sql
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO 
TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)]
```

##### 3.1.2 说明：

1、Load 操作只是单纯的复制/移动操作，将数据文件移动到 Hive 表对应的位置。

<br/>

2、filepath

相对路径，例如：project/data1 

绝对路径，例如：/user/hive/project/data1 

包含模式的完整 URI，列如：

hdfs://namenode:9000/user/hive/project/data1

<br/>

3、LOCAL关键字

如果指定了 LOCAL，
load 命令会去查找本地文件系统中的 filepath。

如果没有指定 LOCAL 关键字，则根据inpath中的uri查找文件。

<br/>

4、OVERWRITE 关键字

如果使用了OVERWRITE 关键字，则目标表（或者分区）中的内容会被删除，然后再将 filepath 指向的文件/目录中的内容添加到表/分区中。

如果目标表（分区）已经有一个文件，并且文件名和 filepath 中的文件名冲突，那么现有的文件会被新文件所替代。

##### 3.1.3 具体实例

1、 加载相对路径数据。

{% asset_img 相对路径数据.png %}

2、 加载绝对路径数据。

{% asset_img 绝对路径数据.png %}

3、 加载包含模式数据。

{% asset_img 包含模式数据.png %}

4、 OVERWRITE关键字使用。

{% asset_img OVERWRITE关键字.png %}

<br/>

<br/>

##### 3.2 insert

##### 3.2.1  语法结构

INSERT OVERWRITE TABLE tablename1 [PARTITION (partcol1=val1,
partcol2=val2 ...)] select_statement1 FROM from_statement

<br/>

Multiple inserts:

FROM from_statement 

INSERT OVERWRITE TABLE tablename1 [PARTITION (partcol1=val1,partcol2=val2 ...)] select_statement1 

[INSERT OVERWRITE TABLE tablename2 [PARTITION ...]select_statement2] ...

<br/>

Dynamic partition inserts:

INSERT OVERWRITE TABLE tablename PARTITION (partcol1[=val1],partcol2[=val2] ...) select_statement FROM from_statement

<br/>

##### 3.2.2 具体实例

1、基本模式插入。

{% asset_img 基本模式插入.png %}

2、多插入模式。

{% asset_img 多插入模式.png %}

3、自动分区模式。

{% asset_img 自动分区模式.png %}