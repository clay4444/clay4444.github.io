---
title: Storm架构及其资源分配问题
tags:
  - Storm
categories:
  - big-data
abbrlink: 9799a940
date: 2018-03-08 17:22:29
---

### storm优点

1. 跨语言
2. 可伸缩的
3. 低延迟,秒级/分钟级
4. 容错。

### 图示

{% asset_img s1.png %}

<br/>

### 核心概念

1. Tuple

主要的数据结构，有序元素的列表。

1. Stream

Tuple的序列。

1. Spouts

数据流源头。可以读取kafka队列消息。可以自定义。

1. Bolts

转接头.

逻辑处理单元。spout的数据传递个bolt，bolt计算，完成后产生新的数据。

IBolt是接口。

<br/>

### Topology

Spout + bolt连接在一起形成一个top，形成有向图，定点就是计算，边是数据流。

<br/>

### task

Bolt中每个Spout或者bolt都是一个task.

<br/>

### Storm架构

{% asset_img s2.png %}

<br/>

1. Nimbus(灵气)

master节点。

核心组件，运行top。

分析top并收集运行task。分发task给supervisor.

监控top。

无状态，依靠zk监控top的运行状况。

<br/>

1. Supervisor(监察)

每个supervisor有n个worker进程，负责代理task给worker。

worker在孵化执行线程最终运行task。

storm使用内部消息系统在nimbus和supervisor之间进行通信。

接受nimbus指令，管理worker进程完成task派发。

<br/>

1. worker

执行特定的task，worker本身不执行任务，而是孵化executors，

让executors执行task。

<br/>

1. Executor

本质上有worker进程孵化出来的一个线程而已。

executor运行task都属于同一spout或者bolt.

<br/>

1. task

执行实际上的任务处理。或者是Spout或者是bolt.

<br/>

### storm工作流程

1. nimbus等待提交的top
2. 提交top后，nimbus收集task，
3. nimbus分发task给所有可用的supervisor
4. supervisor周期性发送心跳给nimbus表示自己还活着。
5. 如果supervisor挂掉，不会发送心跳给nimubs，nimbus将task发送给其他的supervisor
6. nimubs挂掉，super会继续执行自己task。
7. task完成后，supervisor等待新的task
8. 同时，挂掉的nimbus可以通过监控工具软件自动重启。

<br/>

### 没有设置并发程度和任务时候的工作流程

{% asset_img s3.png %}

<br/>

worker是进程，worker里面开的是(executor)线程，线程里面运行的是任务， 任务就是程序执行时的任务对象。

一台机器可以有多个进程，配置文件中配置的supervisor.slots.ports:  就是每台机器最多允许开多少个进程。

<br/>

### 设置top的并发程度和任务

配置并发度.

​	**并发度 ==== 所有的task个数的总和**

<br/>

1. 设置worker数据

```java
conf.setNumWorkers(1);
```

1. 设置executors个数

```java
//设置Spout的并发暗示 (executor个数)
builder.setSpout("wcspout", new WordCountSpout(),3);

//设置bolt的并发暗示
builder.setBolt("split-bolt", new SplitBolt(),4)
```

1. 设置task个数

```java
每个线程可以执行多个task.
builder.setSpout("wcspout", new WordCountSpout(),3).setNumTasks(2);
//
builder.setBolt("split-bolt", new SplitBolt(),4).shuffleGrouping("wcspout").setNumTasks(3);
```



<br/>

### 示例

{% asset_img s4.png %}

<br/>

三个worker进程，

{% asset_img s5.png %}

<br/>

nc 汇集日志结果：spout数据源：

{% asset_img s6.png %}

spout：因为有3个任务，且并发度是3，所以要开启三个线程，每个线程运行一个spout，而且在三个woker进程中各自开启一个线程执行，

<br/>

nc 汇集日志结果：spilt bolt：

{% asset_img s7.png %}

因为有4个bolt任务，且并发度是4，所以需要开启4个线程，又因为只有三个worker进程，所以会有一个节点(woker进程) 开启两个进程。

<br/>

nc 汇集日志结果   counter bolt：

{% asset_img s8.png %}

因为有5个bolt任务，且并发度是5，所以需要开启5个线程，又因为只有三个worker进程，所以会有两个节点(woker进程) 同时开启两个进程。



<br/>

### 资源的均衡分配

假设：两个进程，spout三个，并发度为3，splitbolt 4个，并发度为4，counterbolt5个，并发度为5，

资源的分配情况如图所示：

{% asset_img s9.png %}

