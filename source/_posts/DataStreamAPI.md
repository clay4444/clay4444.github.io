---
title: DataStreamAPI
tags:
  - Flink
abbrlink: 589f5c61
date: 2018-10-06 00:28:46
---

## Graph

### 1. StreamGraph（对应在java代码中是一个类）

- 根据用户代码生成的最初的类
- 表示程序的拓扑结构
- 在client端生成
- env中存储LIst<StreamTransformation<?>> ，每个算子都会存储在list集合中
- StreamTransformation
  1. 描述DataStream之间的转化关系
  2. 包含StreamOperator / UDF （自定义函数）
  3. OneInputTransFormation / transformationTwoInputTransform / SourceTransformation / SinkTransformation
- StreamNode / StreamEdge
  1. 通过StreamTransformation构造

<br/>

### 2. JobGraph（对应在java代码中是一个类）

- 优化StreamGraph
- 将多个符合条件的Node Chain在一起
- 在client端生成
- StreamNode  ->  JobVertex
- StreamEdge -> JobEdge
- 将多个StreamNode Chain成一个JobVertex 的要求
  1. 上下游并行度一致，且下游入度为1
  2. 属于同一个slot group
  3. 上游节点策略为ALWAYS / HEAD，下游节点必须是ALWAYS
  4. 节点之间数据分区方式为forward
  5. 没有主动禁止chain
- 根据group指定JobVertex所属的SlotSharingGroup
- 配置checkpoint
- 配置重启策略

<br/>

{% asset_img 1.png %}

<br/>

### 3. ExcutionGraph

- jobmanager 根据JonGraph生成，并行化
- ExecutionGraph <- JobVertex
- ExecutionVertex并发任务
- ExecutionEdge  <- JobEdge
- JobGraph 2维结构
- 根据二维结构分发对应Vertex到指定的Slot  ( 绑定关系 )

<br/>

{% asset_img 2.png %}

<br/>

#### 4. 物理执行图

- 实际执行图，不可见

<br/>

<br/>

## 数据流转换关系

{% asset_img 3.png %}

<br/>

<br/>

## rescale、shuffle、rebalance

可以解决数据倾斜的问题

rescale针对并行度 （例如两个source，四个trans，每个source总是往固定的trans发送数据）

shuffle针对数据，随机

rebalance针对数据，round-robin

各有优缺点

<br/><br/>

## connect & union

connect会对两个流的数据应用不同的处理方法，并且双流之间可以共享状态，这在第一个流输入会影响第二个流时，会非常有用，union合并多个流，新的流包含所有流的数据

connect只能链接两个流，而union可以连接多个流

connect连接的两个流的数据类型可以不一致，而union连接的流的类型必须一致

<br/>

<br/>

## 重启策略配置

streamExecutionEnvironment.setRestartStrategy