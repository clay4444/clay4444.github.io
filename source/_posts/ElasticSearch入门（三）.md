---
title: ElasticSearch入门（三）
tags:
  - elasticsearch
abbrlink: 5bb24833
date: 2018-10-07 22:36:17
---

## ElasticSearch的基础分布式架构

### 总体图示

{% asset_img ES的基础分布式架构.png %}

<br/>

### Elasticsearch对复杂分布式机制的透明隐藏特性

Elasticsearch是一套分布式的系统，分布式是为了应对大数据量
隐藏了复杂的分布式机制

分片机制（我们之前随随便便就将一些document插入到es集群中去了，我们有没有care过数据怎么进行分片的，数据到哪个shard中去）

cluster discovery（集群发现机制，我们之前在做那个集群status从yellow转green的实验里，直接启动了第二个es进程，那个进程作为一个node自动就发现了集群，并且加入了进去，还接受了部分数据，replica shard）

shard负载均衡（举例，假设现在有3个节点，总共有25个shard要分配到3个节点上去，es会自动进行均匀分配，以保持每个节点的均衡的读写负载请求）

shard副本，请求路由，集群扩容，shard重分配

<br/>

### Elasticsearch的垂直扩容与水平扩容

垂直扩容：采购更强大的服务器，成本非常高昂，而且会有瓶颈，假设世界上最强大的服务器容量就是10T，但是当你的总数据量达到5000T的时候，你要采购多少台最强大的服务器啊

水平扩容：业界经常采用的方案，采购越来越多的普通服务器，性能比较一般，但是很多普通服务器组织在一起，就能构成强大的计算和存储能力

扩容对应用程序的透明性

<br/>

### 增减或减少节点时的数据rebalance

保持负载均衡

<br/>

### master节点

（1）创建或删除索引
（2）增加或删除节点

<br/>

### 节点平等的分布式架构

（1）节点对等，每个节点都能接收所有的请求
（2）自动请求路由，接收请求的节点负责去请求数据所在的节点，并接收数据
（3）响应收集，接收请求的节点负责把数据收集起来返回给client

<br/>

<br/>

## shard&replica机制再次梳理以及单node环境中创建index图解

<br/>

### shard&replica机制再次梳理

（1）index包含多个shard
（2）每个shard都是一个最小工作单元，承载部分数据，lucene实例，完整的建立索引和处理请求的能力
（3）增减节点时，shard会自动在nodes中负载均衡
（4）primary shard和replica shard，每个document肯定只存在于某一个primary shard以及其对应的replica shard中，不可能存在于多个primary shard
（5）replica shard是primary shard的副本，负责容错，以及承担读请求负载
（6）primary shard的数量在创建索引的时候就固定了，replica shard的数量可以随时修改
（7）primary shard的默认数量是5，replica默认是1，默认有10个shard，5个primary shard，5个replica shard
（8）primary shard不能和自己的replica shard放在同一个节点上（否则节点宕机，primary shard和副本都丢失，起不到容错的作用），但是可以和其他primary shard的replica shard放在同一个节点上

<br/>

图示

{% asset_img shard&replica机制再次梳理.png %}

<br/>

### 图解单node环境下创建index是什么样子的

（1）单node环境下，创建一个index，有3个primary shard，3个replica shard
（2）集群status是yellow
（3）这个时候，只会将3个primary shard分配到仅有的一个node上去，另外3个replica shard是无法分配的
（4）集群可以正常工作，但是一旦出现节点宕机，数据全部丢失，而且集群不可用，无法承接任何请求

<br/>

```json
PUT /test_index
{
    "settings" : {
        "number_of_shards" : 3,
        "number_of_replicas" : 1
    }
}
```

<br/>

图示：

{% asset_img 单node环境下创建index.png %}

<br/>

## 两个node环境下replica shard是如何分配的

图示：

{% asset_img 图解2个node环境下replica shard是如何分配的.png %}

<br/>

（1）replica shard分配：3个primary shard，3个replica shard，1 node
（2）primary ---> replica同步
（3）读请求：primary/replica

<br/>

<br/>

## 横向扩容，超出扩容极限，以及提升容错性

（1）primary&replica自动负载均衡，6个shard，3 primary，3 replica
（2）每个node有更少的shard，IO/CPU/Memory资源给每个shard分配更多，每个shard性能更好
（3）扩容的极限，6个shard（3 primary，3 replica），最多扩容到6台机器，每个shard可以占用单台服务器的所有资源，性能最好
（4）超出扩容极限，动态修改replica数量，9个shard（3primary，6 replica），扩容到9台机器，比3台机器时，拥有3倍的读吞吐量
（5）3台机器下，9个shard（3 primary，6 replica），资源更少，但是容错性更好，最多容纳2台机器宕机，6个shard只能容纳1台机器宕机
（6）这里的这些知识点，综合起来看，就是说，一方面告诉你扩容的原理，怎么扩容，怎么提升系统整体吞吐量；另一方面要考虑到系统的容错性，怎么保证提高容错性，让尽可能多的服务器宕机，保证数据不丢失

<br/>

图示：（注意图中有部分错误，6个shard3台节点，可以容忍1台节点宕机）

{% asset_img 扩容过程分析.png %}

<br/>

<br/>

## Elasticsearch容错机制

master选举，replica容错，数据恢复

<br/>

（1）9 shard，3 node
（2）master node宕机，自动master选举，red
（3）replica容错：新master将replica提升为primary shard，yellow
（4）重启宕机node，master copy replica到该node，使用原有的shard并同步宕机后的修改，green

<br/>

图示：

{% asset_img es容错过程分析.png %}

<br/>

<br/>

## Document的核心元数据

1. _index元数据
2. _type元数据
3. _id元数据

```json
{
    "_index": "test_index",
    "_type": "test_type",
    "_id": "1",
    "_version": 1,
    "found": true,
    "_source": {
        "test_content": "test test"
    }
}
```

<br/>

### _index元数据

（1）代表一个document存放在哪个index中
（2）类似的数据放在一个索引，非类似的数据放不同索引：product index（包含了所有的商品），sales index（包含了所有的商品销售数据），inventory index（包含了所有库存相关的数据）。如果你把比如product，sales，human resource（employee），全都放在一个大的index里面，比如说company index，不合适的。
（3）index中包含了很多类似的document：类似是什么意思，其实指的就是说，这些document的fields很大一部分是相同的，你说你放了3个document，每个document的fields都完全不一样，这就不是类似了，就不太适合放到一个index里面去了。
（4）索引名称必须是小写的，不能用下划线开头，不能包含逗号：product，website，blog

<br/>

创建反例

{% asset_img index如何创建的反例分析.png %}

<br/>

### _type元数据

（1）代表document属于index中的哪个类别（type）
（2）一个索引通常会划分为多个type，逻辑上对index中有些许不同的几类数据进行分类：因为一批相同的数据，可能有很多相同的fields，但是还是可能会有一些轻微的不同，可能会有少数fields是不一样的，举个例子，就比如说，商品，可能划分为电子商品，生鲜商品，日化商品，等等。
（3）type名称可以是大写或者小写，但是同时不能用下划线开头，不能包含逗号

<br/>

### _id元数据

（1）代表document的唯一标识，与index和type一起，可以唯一标识和定位一个document
（2）我们可以手动指定document的id（put /index/type/id），也可以不指定，由es自动为我们创建一个id

<br/>

<br/>

## document 的id手动指定和自动生成两种方式解析

<br/>

### 手动指定document id

根据应用情况来说，是否满足手动指定document id的前提：

一般来说，是从某些其他的系统中，导入一些数据到es时，会采取这种方式，就是使用系统中已有数据的唯一标识，作为es中document的id。举个例子，比如说，我们现在在开发一个电商网站，做搜索功能，或者是OA系统，做员工检索功能。这个时候，数据首先会在网站系统或者IT系统内部的数据库中，会先有一份，此时就肯定会有一个数据库的primary key（自增长，UUID，或者是业务编号）。如果将数据导入到es中，此时就比较适合采用数据在数据库中已有的primary key。

如果说，我们是在做一个系统，这个系统主要的数据存储就是es一种，也就是说，数据产生出来以后，可能就没有id，直接就放es一个存储，那么这个时候，可能就不太适合说手动指定document id的形式了，因为你也不知道id应该是什么，此时可以采取下面要讲解的让es自动生成id的方式。

```json
PUT /test_index/test_type/2
{
  "test_content": "my test"
}
```

<br/>

### 自动生成document id

自动生成的id，长度为20个字符，URL安全，base64编码，GUID，分布式系统并行生成时不可能会发生冲突

```json
POST /test_index/test_type
{
  "test_content": "my test"
}
```

结果

```json
{
  "_index": "test_index",
  "_type": "test_type",
  "_id": "AVp4RN0bhjxldOOnBxaE",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "created": true
}
```

<br/>

图示

{% asset_img GUID不冲突解释.png %}

<br/>

<br/>

## document的_source元数据以及定制返回结果解析

### _source元数据

<br/>

```json
put /test_index/test_type/1
{
  "test_field1": "test field1",
  "test_field2": "test field2"
}
```

get /test_index/test_type/1

```json
{
    "_index": "test_index",
    "_type": "test_type",
    "_id": "1",
    "_version": 2,
    "found": true,
    "_source": {
        "test_field1": "test field1",
        "test_field2": "test field2"
    }
}
```

<br/>

_source元数据：就是说，我们在创建一个document的时候，使用的那个放在request body中的json串，默认情况下，在get的时候，会原封不动的给我们返回回来。

<br/>

### 定制返回结果

定制返回的结果，指定_source中，返回哪些field

GET /test_index/test_type/1?_source=test_field1,test_field2

```json
{
  "_index": "test_index",
  "_type": "test_type",
  "_id": "1",
  "_version": 2,
  "found": true,
  "_source": {
    "test_field2": "test field2"
  }
}
```

<br/>

## 全量替换、强制创建以及文档删除等操作的分析

<br/>

### document的全量替换

（1）语法与**创建文档**是一样的，如果document id不存在，那么就是创建；如果document id已经存在，那么就是全量替换操作，替换document的json串内容
（2）document是不可变的，如果要修改document的内容，第一种方式就是全量替换，直接对document重新建立索引，替换里面所有的内容
（3）es会将老的document标记为deleted，然后新增我们给定的一个document，当我们创建越来越多的document的时候，es会在适当的时机在后台自动删除标记为deleted的document

<br/>

图示

{% asset_img document-delete的原理.png %}

<br/>

### document的强制创建

（1）创建文档与全量替换的语法是一样的，有时我们只是想新建文档，不想替换文档，如果强制进行创建呢？
（2）PUT /index/type/id?op_type=create，PUT /index/type/id/_create

<br/>

### document的删除

（1）DELETE /index/type/id
（2）不会理解物理删除，只会将其标记为deleted，当数据越来越多的时候，在后台自动删除

<br/>

<br/>

## ElasticSearch的并发冲突问题

图示：

{% asset_img 深度图解剖析Elasticsearch并发冲突问题.png %}

<br/>

<br/>

## 悲观锁和乐观锁两种并发控制方案

图示：

{% asset_img 深度图解剖析悲观锁与乐观锁两种并发控制方案.png %}

<br/>

<br/>

## 图解内部如何基于_version进行乐观锁的并发控制

图示：

{% asset_img 图解Elasticsearch内部如何基于_version进行乐观锁并发控制.png %}

<br/>

### _version元数据

```json
PUT /test_index/test_type/6
{
  "test_field": "test test"
}
```

结果

```json
{
    "_index": "test_index",
    "_type": "test_type",
    "_id": "6",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "created": true
}
```

第一次创建一个document的时候，它的_version内部版本号就是1；以后，每次对这个document执行修改或者删除操作，都会对这个_version版本号自动加1；哪怕是删除，也会对这条数据的版本号加1

```json
{
    "found": true,
    "_index": "test_index",
    "_type": "test_type",
    "_id": "6",
    "_version": 4,
    "result": "deleted",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    }
}
```

我们会发现，在删除一个document之后，可以从一个侧面证明，它不是立即物理删除掉的，因为它的一些版本号等信息还是保留着的。先删除一条document，再重新创建这条document，其实会在delete version基础之上，再把version号加1