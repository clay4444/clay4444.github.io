---
title: Flume日志收集框架初探
tags:
  - flume
categories:
  - big-data
abbrlink: 117ed1fe
date: 2018-03-08 10:39:51
---

## flume

{% asset_img f1.png %}

<br/>

收集、移动、聚合大量日志数据的服务。

基于**流数据**的架构，用于在线日志分析。

基于**事件**。

在生产和消费者之间启动协调作用。

提供了**事务**保证，确保消息一定被分发。

Source 多种

sink多种.

<br/>

multihop		//多级跃点.

{% asset_img f2.png %}

<br/>



### Source

​	接受数据，类型有多种。

<br/>



### Channel

临时存放地，对Source中来的数据进行缓冲，直到sink消费掉。



<br/>

### Sink

从channel提取数据存放到中央化存储(hadoop / hbase)。

<br/>

### 安装flume1.7.0

1. 下载

2. tar

3. 环境变量

4. 验证flume是否成功

   $>flume-ng version			//next generation.下一代.

<br/>

### source到channel的执行过程

{% asset_img f3.png %}

<br/>

{% asset_img f4.png %}

<br/>



### sink的执行过程

{% asset_img f5.png %}

<br/>{% asset_img f6.png %}

<br/>

### 安装nc

$>sudo yum install nc

<br/>

### 清除仓库缓存

​	$>修改ali.repo --> ali.repo.bak文件。

​	$>sudo yum clean all

​	$>sudo yum makecache

​	#例如阿里基本源 

​	$>sudo wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

​	#阿里epel源

​	$>sudo wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

<br/>

### 配置flume

1. 创建配置文件

```shell
[/soft/flume/conf/hello.conf]

#声明三种组件

a1.sources = r1
a1.channels = c1
a1.sinks = k1

#定义source信息
a1.sources.r1.type=netcat
a1.sources.r1.bind=localhost
a1.sources.r1.port=8888

#定义sink信息
a1.sinks.k1.type=logger

#定义channel信息
a1.channels.c1.type=memory

#绑定在一起
a1.sources.r1.channels=c1

a1.sinks.k1.channel=c1			注意：source可以到多个通道中去，但是sink只能从一个通道中提取。
```



<br/>

1. 运行

<br/>

a)启动flume agent

```shell
$>bin/flume-ng agent -f ./conf/hello.conf -n a1 -Dflume.root.logger=INFO,console
```

<br/>

b)启动nc的客户端

```shell
$>nc localhost 8888
$nc>hello world
```

<br/>

c)在flume的终端输出hello world.

<br/>



### flume source

<br/>

#### 1. exec

​	**实时日志**收集,实时收集日志。

```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1

a1.sources.r1.type=exec
a1.sources.r1.command=tail -F /home/hadoop/test.txt
a1.sinks.k1.type=logger
a1.channels.c1.type=memory
a1.sources.r1.channels=c1

a1.sinks.k1.channel=c1
```

<br/>

```shell
bin/flume-ng agent -f ./conf/exec_r.conf -n a1 -Dflume.root.logger=INFO,console
```

<br/>

#### 2. 批量收集

监控一个文件夹，静态文件。

收集完之后，会重命名文件成新文件。.compeleted.

​	a) 配置文件

```shell
[spooldir_r.conf]

a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type=spooldir
a1.sources.r1.spoolDir=/home/hadoop/spool
a1.sources.r1.fileHeader=true
a1.sinks.k1.type=logger
a1.channels.c1.type=memory
a1.sources.r1.channels=c1

a1.sinks.k1.channel=c1

```

​	b)创建目录

```shell
$>mkdir ~/spool
```

​	c)启动flume

```shell
$>bin/flume-ng agent -f ./conf/spooldir_r.conf -n a1 -Dflume.root.logger=INFO,console
```

​        d) 结果	

{% asset_img f7.png %}

<br/>

{% asset_img f8.png %}

<br/>



#### 3. 序列source

<br/>

```shell
[seq_r.conf]
a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type=seq
a1.sources.r1.totalEvents=1000

a1.sinks.k1.type=logger

a1.channels.c1.type=memory

a1.sources.r1.channels=c1
a1.sinks.k1.channel=c1
```

运行

```shell
$>bin/flume-ng agent -f ../conf/helloworld.conf -n a1 -Dflume.root.logger=INFO,console
```

<br/>

#### 4. StressSource

```shell
a1.sources = stresssource-1
a1.channels = memoryChannel-1
a1.sources.stresssource-1.type = org.apache.flume.source.StressSource
a1.sources.stresssource-1.size = 10240
a1.sources.stresssource-1.maxTotalEvents = 1000000
a1.sources.stresssource-1.channels = memoryChannel-1
```

<br/>



### flume sink

<br/>

#### 1. hdfs_k.conf

```shell
a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 8888

a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = /user/hadoop/flume/%y-%m-%d/%H/%M/%S
a1.sinks.k1.hdfs.filePrefix = events-

#是否是产生新目录,每十分钟产生一个新目录,一般控制的目录方面。
# round产生新目录，roll产生新文件
#2017-12-12 -->
#2017-12-12 -->%H%M%S

a1.sinks.k1.hdfs.round = true			
a1.sinks.k1.hdfs.roundValue = 10
a1.sinks.k1.hdfs.roundUnit = second

a1.sinks.k1.hdfs.useLocalTimeStamp=true

#是否产生新文件。
a1.sinks.k1.hdfs.rollInterval=10
a1.sinks.k1.hdfs.rollSize=10
a1.sinks.k1.hdfs.rollCount=3

a1.channels.c1.type=memory

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

<br/>

运行

```shell
$>bin/flume-ng agent -f ./conf/hdfs_k.conf -n a1 
```

<br/>

[启动nc的客户端]

```shell
$>nc localhost 8888
$nc>hello world
```

<br/>

结果：

{% asset_img f9.png %}

<br/>{% asset_img f10.png %}



<br/>

#### 2.hbase_k.conf

```shell
a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 8888

a1.sinks.k1.type = hbase
a1.sinks.k1.table = ns1:t12
a1.sinks.k1.columnFamily = f1
a1.sinks.k1.serializer = org.apache.flume.sink.hbase.RegexHbaseEventSerializer

a1.channels.c1.type=memory

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

<br/>

[运行]

```
$>bin/flume-ng agent -f ./conf/hbase_k.conf -n a1 
```

<br/>

[启动nc的客户端]

```shell
$>nc localhost 8888
$nc>hello world
```



<br/>

#### 3. 使用avroSource和AvroSink实现跃点agent处理

{% asset_img f11.png %}

<br/>

为什么不建议让每个agent直接连接目的地端，因为如果目的地端是hadoop，那么就意味着每个agent都会和hadoop建立一个socket连接，降低了效率，所以一般会有一个agent负责收集前面agent的数据，做汇总，然后再统一交给hadoop。	

<br/>

这时就需要用到avro， 此时多个agent之间的通信就是进程间的通信，  服务端(a2)是bind，客户端(a1)是host

<br/>

1. 创建配置文件

```shell
[avro_hop.conf]
#a1
a1.sources = r1
a1.sinks= k1
a1.channels = c1

a1.sources.r1.type=netcat
a1.sources.r1.bind=localhost
a1.sources.r1.port=8888

a1.sinks.k1.type = avro
a1.sinks.k1.hostname=localhost
a1.sinks.k1.port=9999

a1.channels.c1.type=memory

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1

#a2
a2.sources = r2
a2.sinks= k2
a2.channels = c2

a2.sources.r2.type=avro
a2.sources.r2.bind=localhost
a2.sources.r2.port=9999

a2.sinks.k2.type = logger

a2.channels.c2.type=memory

a2.sources.r2.channels = c2
a2.sinks.k2.channel = c2
```

<br/>

1. 启动a2

```shell
$>flume-ng agent -f /home/hadoop/apps/flume/conf/avro_hop.conf -n a2 -Dflume.root.logger=INFO,console
```

<br/>

1. 验证a2

```shell
$>netstat -anop | grep 9999
```

<br/>

1. 启动a1

```shell
$>flume-ng agent -f /home/hadoop/apps/flume/conf/avro_hop.conf -n a1
```

<br/>

1. 验证a1

```shell
$>netstat -anop | grep 8888
```

<br/>



### channel

1. MemoryChannel

略

<br/>

1. FileChannel：

~~~shell
a1.sources = r1
a1.sinks= k1
a1.channels = c1

a1.sources.r1.type=netcat
a1.sources.r1.bind=localhost
a1.sources.r1.port=8888

a1.sinks.k1.type=logger

a1.channels.c1.type = file
a1.channels.c1.checkpointDir = /home/centos/flume/fc_check
a1.channels.c1.dataDirs = /home/centos/flume/fc_data

a1.sources.r1.channels=c1
a1.sinks.k1.channel=c1

可溢出文件通道
------------------
a1.channels = c1
a1.channels.c1.type = SPILLABLEMEMORY
#0表示禁用内存通道，等价于文件通道
a1.channels.c1.memoryCapacity = 0
#0,禁用文件通道，等价内存通道。
a1.channels.c1.overflowCapacity = 2000

a1.channels.c1.byteCapacity = 800000
a1.channels.c1.checkpointDir = /user/hadoop/flume/fc_check
a1.channels.c1.dataDirs = /user/hadoop/flume/fc_data
~~~

