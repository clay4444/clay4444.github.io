---
title: Flink基本概念与部署
tags:
  - Flink
abbrlink: 4042ab31
date: 2018-10-05 21:08:25
---

## 一：Flink 编程模型

### Flink 分层架构：

{% asset_img 一1.png %}

<br/>

#### （1）Stateful Stream Processing

1. 位于最底层，是core API 的底层实现
2. ProcessingFunction
3. 利用地界，构建一些新的组件（比如：利用其定时做一定情况下的匹配和缓存）
4. 灵活性高，但开发比较复杂

<br/>

#### （2）Core API

1. DataStrea
2. DataSet

<br/>

#### （3）Table & Sql

1. SQL构建在Table之上，都需要构建Table环境
2. 不用类型的Table构建不同的Table 环境
3. Table可以与DataStream或者TableSet进行相互转换
4. StreamSql不同于存储的sql，最终会转化为流式的执行计划

<br/>

### State

1. Flink提供了一套自己的状态托管机制
2. Operator State ：算子的状态（存在内存中）
3. Keyed State：计算的中间状态 （根据backend的方式区分到底存在哪个地方）
4. State Backend ：rocksdb + hdfs （rocksdb是同步操作，hdfs是异步操作）

<br/>

### checkpoint

1. 轻量级容错机制（全局异步，局部同步）
2.  保证exactly-once语义 （flink内部整一致，不包含第三方组件）
3. 用于内部失败的恢复 （例如 taskmanager 挂掉了）
4. 基本原理：**jobmanager通过往sourece注入barrier，使用barrier作为checkpoint的标致**

{% asset_img 一2.png %}

<br/>

### savepoint

1. 流处理过程中的状态历史版本
2. 具有可以replay的功能
3. 外部恢复（应用重启和升级）
4. 两种方式触发：canle with savepoint   /  手动主动触发

<br/>

<br/>

## 二：Flink 运行时

<br/>

### Flink 运行时架构

1. Client
2. JobManager
3. TaskManager
4. 角色间的通信（Akka）
5. 数据的传输（Netty）

{% asset_img 一3.png %}

<br/>

### TaskManager Slot（槽）

1. TaskManager对应进程，一个槽会执行很多的线程（task）
2. 槽会平分taskmanager的资源

{% asset_img 一4.png %}

<br/>

### CoLocationGroup（task分组策略）

1. 保证所有并行度id相同的task都运行在同一个slot中
2. 主要用于迭代流

<br/>

### SlotSharingGroup（task分组策略）

1. 保证同一个group的并行度一致的subtask共享同一个slots
2. 算子的默认group为default
3. 根据input的group和自身是否设置group共同确定一个算子的SlotSharingGroup
4. 适当设置可以减少每个slot运行的线程数，从而整体上减少机器的负载（例如：3个tm，9个slot，每个槽运行10个线程，使用SlotSharingGroup策略之后，原来在同一个槽中的线程现在根据策略要进入不同的槽，此时每个槽运行的线程数目减少，机器负载减少）
5. 其实就是确定哪几个算子运行在同一个槽上

{% asset_img 一5.png %}

<br/>

### 槽和并行度

#### 一个应用需要多少个slot

1. 不设置SlotSharingGroup的情况下：槽的数量等于应用的最大并行度
2. 设置SlotSharingGroup的情况下：槽的数量等于所有SlotSharingGroup中的最大并行度之和

{% asset_img 一6.png %}

<br/>

### OperatorChain 和 Task

1. OperatorChain就是两个算子运行在同一个线程中（合并为一个任务）

{% asset_img 一7.png %}

<br/>

#### （1） OperatorChain的优点

1. 减少线程的切换
2. 减少序列化和反序列化
3. 减少延迟并且提高吞吐能力（如果没有OperatorChain的话，算子和算子之间是通过**buffer**连接的，上行算子写到buffer，下行算子再去读取，buffer不够用时，就会出现反压问题）

#### （2） OperatorChain的组成条件

1. 没有禁用Chain
2. 上下游算子并行度一致
3. 下游算子入度为1
4. 上下游算子在同一个slot group
5. 上下游算子之间没有数据shuffle（数据直流）

<br/>

### 几个概念小结

1. JobManager（master，负责任务分发）
2. TaskManager（worker）进程
3. TaskManager Slot （资源的分割）
4. Operator （算子）
5. Task（任务执行） 线程
6. Parallelism（并行度）

<br/>

<br/>

## Flink On Yarn  原理 ，部署与生产

### 部署方式

1. Local
2. Standalone（无法最大程度利用机器）
3. Yarn
4. Mesos
5. Docker
6. Kubernetes
7. AWS

<br/>

### Flink On Yarn

1. ResourceManager
2. NodeManager
3. AppMaster（JobManager运行在其上）
4. Container（TaskManager运行在其上）
5. YarnSession（作为资源去看）
6. 选择On Yarn的理由（提高资源的利用率，Hadoop开源活跃，且成熟）

{% asset_img 一8.png %}

<br/>

### 理解Job的启动过程

1. Graph的不用阶段的转换
2. Scheduler（task的分配方式）
3. Blob服务（jar包传输）
4. HA保证

{% asset_img 一9.png %}

<br/>

### Graph

1. StreamGraph（客户端生成）
2. JobGraph（客户端生成，主要进行OperatorChain优化）
3. ExecuteGraph（JobGraph之后只是设置了并行度，JobManager生成ExecuteGraph会把并行度展开）
4. 物理执行图（worker生成的）

<br/>

### HA服务

1. 选举leader（JobManager）
2. 持久化checkpoint元数据
3. 持久化最近成功的checkpoint
4. 持久化提交的JobGraph（SubmittedJobGraph）
5. 持久化BlobStore（用户保存应用的jar）

<br/>

### Flink-conf 配置

<br/>

#### Akka方面配置

akka.ask.timeout

akka.framesize

akka.watch.heartbeat.interval

akka.watch.heartbeat.pause

#### checkpoint方面配置

state.backend  (rocksdb)

state.backend.fs.checkpointdir

state.checkpoints.dir   (savepoint目录)

#### HA配置

high-availability (zookeeper)

high-availability.cluster-id

high-availability.zookeeper.path.root

high-availability.zookeeper.quorum

high-availability.zookeeper.storageDir

#### 内存配置

#### MetricReporter

#### Yarn方面的配置

<br/>

### YarnSession启动命令

1. -n ( Container ) TaskManager的数量 = 应用所需要的槽的数量 / 每个taskManager的slot的数量
2. -s ( slot ) 每个taskmanager的slot数量，默认一个slot对应一个vcore  = 建议设置： 6 - 10 
3. -jm   jobmanager 的内存大小，= 2G 以上，（单位是M）
4. -tm 每个taskmanager的内存大小 = 槽数乘以1G (etl) 或2G ( window )
5. -qu  Yarn的队列名字
6. -nm  应用的应用名字
7. -d 后台执行

<br/>

### 应用启动命令

1. -j 运行的jar包
2. -a 应用运行的参数
3. -c 应用的主类
4. -p 应用的并行度
5. -yid  yarnsession对应的appid
6. -nm 应用的名字，在flink-ui上显示
7. -d 后台执行

<br/><br/>

## 附：flink on yarn 部署文档

flink on yarn需要的组件与版本如下

1. Zookeeper 3.4.9 用于做Flink的JobManager的HA服务
2. hadoop 2.7.2 搭建HDFS和Yarn
3. flink 1.3.2 或者 1.4.1版本（scala 2.11）

Zookeeper, HDFS 和 Yarn 的组件的安装可以参照网上的教程。

在zookeeper，HDFS 和Yarn的组件的安装好的前提下，在客户机上提交Flink任务，具体流程如下：

- 在启动Yarn-Session 之前， 设置好HADOOP_HOME,YARN_CONF_DIR ， HADOOP_CONF_DIR环境变量中三者的一个。如下所示， 根据具体的hadoop 路径来设置

```command
   $ export HADOOP_HOME=/usr/local/hadoop-current
```

- 配置flink 目录下的flink-conf.yaml, 如下所示

```yaml
jobmanager.rpc.address: localhost
jobmanager.rpc.port: 6123
jobmanager.heap.mb: 256
taskmanager.heap.mb: 512
taskmanager.numberOfTaskSlots: 1
taskmanager.memory.preallocate: false
parallelism.default: 1
jobmanager.web.port: 8081

# yarn
yarn.maximum-failed-containers: 99999

#akka config
akka.watch.heartbeat.interval: 5 s
akka.watch.heartbeat.pause: 20 s
akka.ask.timeout: 60 s
akka.framesize: 20971520b

#high-avaliability
high-availability: zookeeper
## 根据安装的zookeeper信息填写
high-availability.zookeeper.quorum: 10.141.61.226:2181,10.141.53.244:2181,10.141.18.219:2181
high-availability.zookeeper.path.root: /flink
## HA 信息存储到HDFS的目录，根据各自的Hdfs情况修改
high-availability.zookeeper.storageDir: hdfs://hdcluster/flink/recovery/

#checkpoint config
state.backend: rocksdb
## checkpoint到HDFS的目录 根据各自安装的HDFS情况修改
state.backend.fs.checkpointdir: hdfs://hdcluster/flink/checkpoint
## 对外checkpoint到HDFS的目录
state.checkpoints.dir: hdfs://hdcluster/flink/savepoint

#memory config
env.java.opts: -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+AlwaysPreTouch -server -XX:+HeapDumpOnOutOfMemoryError
yarn.heap-cutoff-ratio: 0.2
taskmanager.memory.off-heap: true

```

- 提交Yarn-Session，切换到flink的bin 目录下,提交命令如下

```command
   $ ./yarn-session.sh -n 2 -s 6 -jm 3072 -tm 6144 -nm test -d
```

启动yarn-session的参数解释如下

| 参数            | 参数解释                                                     | 设置推荐                                                     |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| -n(--container) | taskmanager的数量                                            |                                                              |
| -s(--slots)     | 用启动应用所需的slot数量/ -s 的值向上取整，有时可以多一些taskmanager，做冗余 每个taskmanager的slot数量，默认一个slot一个core，默认每个taskmanager的slot的个数为1 | 6～10                                                        |
| -jm             | jobmanager的内存（单位MB)                                    | 3072                                                         |
| -tm             | 每个taskmanager的内存（单位MB)                               | 根据core 与内存的比例来设置，-s的值＊ （core与内存的比）来算 |
| -nm             | yarn 的appName(现在yarn的ui上的名字)｜                       |                                                              |
| -d              | 后台执行                                                     |                                                              |

- 提交yarn－session 后，可以在yarn的ui上看到一个应用（应用有一个appId）, 切换到flink的bin目录下，提交flink 应用。命令如下

```command
 $ ./flink -run file:///home/yarn/test.jar -a 1 -p 12 -yid appId -nm flink-test -d
```

启动flink 应用的参数解释如下

| 参数            | 参数解释                                    |
| --------------- | ------------------------------------------- |
| -j              | 运行flink 应用的jar所在的目录               |
| -a              | 运行flink 应用的主方法的参数                |
| -p              | 运行flink应用的并行度                       |
| -c              | 运行flink应用的主类, 可以通过在打包设置主类 |
| -nm             | flink 应用名字，在flink-ui 上面展示         |
| -d              | 后台执行                                    |
| --fromsavepoint | flink 应用启动的状态恢复点                  |

- 启动flink应用成功，即可在yarn ui 点击对应应用的ApplicationMaster链接,既可以查看flink-ui ，并查看flink 应用运行情况。