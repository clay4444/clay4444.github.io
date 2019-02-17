---
title: 详解hbase架构、数据读取、存储、拆分风暴等问题
tags:
  - Hbase
categories:
  - big-data
abbrlink: 615ff67f
date: 2018-03-08 13:57:01
---

### hbase进程

hmaster

hregionserver

<br/>

### hbase 端口

16010

<br/>

### start-hbase.sh

hbase-daemon.sh start master

hbase-daemons.sh start regionserver

<br/>

### hbase的HA设置

启动多个master即可。因为habase本身自己就依赖于zookeeper，所以直接用hbase-daemon.sh start master启动一次就会自动成为back up节点，

zk节点下:/hbase/backup-master

<br/>

### hbase-shell操作

​	$>hbase shell							//登录shell终端.

​	$hbase>help								//

​	$hbase>help	'list_namespace'				//查看特定的命令帮助

​	$hbase>list_namespace					//列出名字空间(数据库)

​	$hbase>list_namespace_tables 'defalut'		//列出名字空间(数据库)

​	$hbase>create_namespace 'ns1'			//创建名字空间

​	$hbase>help 'create'

​	$hbase>create 'ns1:t1','f1'					//创建表,指定空间下  f1是列族

​	$hbase>put 'ns1:t1','row1','f1:id',100		//插入数据

​	$hbase>put 'ns1:t1','row1','f1:name','tom'	//

​	$hbase>get 'ns1:t1','row1'					//查询指定row

​	$hbase>scan 'ns1:t1'						//扫描表



<br/>

### Hbase整体架构

一个 RegionServer 对应多个多个 HRegion，

一张表属于多个HRegion（区域），

读写过程是由 RegionServer 完成的，

{% asset_img h1.png %}

<br/>

至关重要的meta表 ( 所有至关重要的元数据信息都存储在meta表中 )，前面就是（一行）的四列，所有的namespace都存储在这一行中，看它的 rowkey 设计，前面的两个逗号表示起始行和结束行，因为并没有切分，所以是两个逗号。

{% asset_img h2.png %}



<br/>

### 查询/写入过程

<br/>

WAL			//write ahead log, 写前日志。

<br/>

查询tx表，首先不知道它在哪台主机，所以先查询mete表(hbase命名空间下)，因为mete表标识了整个hbase中的所有区域，他们分别在哪个服务器上。

{% asset_img h3.png %}

那meta表在哪台主机呢？还是zk。

{% asset_img h4.png %}



<br/>

### hbase基于hdfs

1. 相同列族的数据存放在一个文件中。hdfs上，
2. [表数据的存储目录结构构成]
3. hdfs://s201:8020/hbase/data/${名字空间}/${表名}/${区域名称}/${列族名称}/${文件名}

{% asset_img h5.png %}

<br/>

[WAL目录结构构成]

hdfs://s201:8020/hbase/WALs/${区域服务器名称,主机名,端口号,时间戳}/

<br/>



### client端交互过程

1. hbase集群启动时，master负责分配区域到指定区域服务器。
2. 联系zk，找出meta表所在rs(regionserver)：/hbase/meta-region-server
3. 定位row key,找到对应region server
4. 缓存信息在本地。
5. 联系RegionServer
6. HRegionServer负责open HRegion对象，为每个列族创建Store对象，Store包含多个StoreFile实例，他们是对HFile的轻量级封装。每个Store还对应了一个MemStore，用于内存存储数据。

<br/>



### 百万数据批量插入

[100万:33921.]

```java
DecimalFormat format = new DecimalFormat();	//想要完整排序就需要这个，
format.applyPattern("0000");

long start = System.currentTimeMillis() ;
Configuration conf = HBaseConfiguration.create();
Connection conn = ConnectionFactory.createConnection(conf);
TableName tname = TableName.valueOf("ns1:t1");
HTable table = (HTable)conn.getTable(tname);
//不要自动清理缓冲区
table.setAutoFlush(false);

for(int i = 4 ; i < 1000000 ; i ++){
  Put put = new Put(Bytes.toBytes("row" + format.format(i))) ;
  //关闭写前日志
  put.setWriteToWAL(false);
  put.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("id"),Bytes.toBytes(i));
  put.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("name"),Bytes.toBytes("tom" + i));
  put.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("age"),Bytes.toBytes(i % 100));
  table.put(put);

  if(i % 2000 == 0){
    table.flushCommits();
  }
}
//
table.flushCommits();
System.out.println(System.currentTimeMillis() - start );
```



<br/>

### region切分

{% asset_img h6.png %}

<br/>

{% asset_img h7.png %}

<br/>

hbase切割文件默认10G进行切割。

```xml
<property>
  <name>hbase.hregion.max.filesize</name>
  <value>10737418240</value>
  <source>hbase-default.xml</source>
</property>
```

<br/>

切分示例(可以切表，也可以切region)

初始状态：

{% asset_img h8.png %}

切分命令：

{% asset_img h9.png %}

结果：

{% asset_img h10.png %}

切完之后hafs上存储的变化：

{% asset_img h11.png %}

<br/>

此时再切就不能按照表来切了，因为表已经切分过了，还可以按照regionName来切，column=info:regionName 这个列族中的value后面的name就是regionName，直接复制过来即可。(后面执行rowKey)

{% asset_img h12.png %}

<br/>



### hbase 存储文件移动

{% asset_img h13.png %}

<br/>

{% asset_img h14.png %}

<br/>

{% asset_img h15.png %}

<br/>

{% asset_img h16.png %}

<br/>



### hbase合并两个Region

{% asset_img h17.png %}

<br/>

{% asset_img h18.png %}

<br/>



### hbase-shell 切割命令

​	$hbase>flush 'ns1:t1'	       //清理内存数据到磁盘。新put的数据要flush一下才能在hdfs上看到

​	$hbase>count 'ns1:t1'		//统计函数       行数

​	$hbase>disable 'ns1:t1'		//删除表之前需要禁用表

​	$hbase>drop 'ns1:t1'		//删除表

​	$hbase>scan 'hbase:meta'	//查看元数据表

​	$hbase>split 'ns1:t1'		//切割表 

​	$hbase>split ''				//切割区域



<br/>

### Hbase的文件分类

1. 一类位于Hbase根目录下，
2. 另一类位于根目录的表目录下

{% asset_img h19.png %}

<br/>

{% asset_img h20.png %}

<br/>

{% asset_img h21.png %}

<br/>

{% asset_img h22.png %}

<br/>

{% asset_img h23.png %}

<br/>

{% asset_img h24.png %}

<br/>

### 拆分风暴

向hbase写入数据的时候，很有可能，几个区域，同时增长，那么一旦到达默认的10G，就会同时切分，服务器的负载瞬间增大，解决办法是手动去切，把默认的切割大小设置很大，到达之前就手动去切割了。