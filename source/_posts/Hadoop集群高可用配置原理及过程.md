---
title: Hadoop集群高可用配置原理及过程
categories:
  - big-data
tags:
  - HA机制
abbrlink: a45c106d
date: 2017-09-15 15:07:04
---

## 高可用 ( high availability )

### 简明图示

{% asset_img hadoop的高可用机制.png %}

<br/>

### hdfs的高可用：

​	不能用keeplived，因为keeplived只是提供一个虚拟的IP地址，哪个web服务器能用，就把这个IP地址绑定到自己的网卡，这样只能支持无状态的进行切换，这里涉及到了内存状态的同步，所以不能用keeplived

<br/>

​	要进行状态的同步，那么active状态的namenode修改数据之后，肯定要同步到standby状态的namenode，fsimage太大，不适合同步，那么只能移动edis日志文件，把edis日志文件交给第三方保管，每次更新到第三方，standby去读取edis日志文件并合并， 那么 active的挂了，standby的节点立马就能用。

<br/>

​	那么第三方就要是高可用，qjounnal是一个edits日志管理文件，每台机器都会保存一份edits文件，qjounnal借用zookeeper来进行分布式协调。但是根据CAP原理，可靠性提高了，但是数据一致性就降低了，

<br/>

​	edit文件本地也会保持一份，因为日志占用的资源也不是很大，修改数据的时候，通过线程池，本地写一份，jounnalnode上每个节点也要写一份，往jounnalnode上写时，只要多数成功，就算成功。

<br/>

问题：standby和active的切换，状态的感知，谁的资源使用率过高了，都是需要东西来管理的，

<br/>

ZKFC的作用就是这个，ZKFC通过RPC接口调用namenope的数据，感知namenode的状态，如果zkfc发现active挂掉了，就通过删除zookeeper上锁节点的方式，通知standby端的ZKFC，standby端的ZKFC就通过RPC调用通知standby启动，启动之前，需要确保原来的active确实死了，此时会发送和一个kill的命令，试图杀死原来active的机器，如果超时，就会调用用户设置的脚本，总之就是要确保原来的active已经死了，然后它就去zookeeper上注册一把自己的锁，此时已经死掉的节点再启动，发现zookeeper上已经有锁了，那么它的状态就是standby。

<br/>

### YARN的高可用：

没有太大必要，因为resourcemanager 连接不上就连接不上，不会涉及到任何状态的同步问题。

解决办法，就是在zookeeper上注册一把锁，

<br/>

### 脑裂

脑裂:两个节点都是激活态。

为防止脑裂，JNs只允许同一时刻只有一个节点向其写数据。容灾发生时，成为active节点的namenode接管向jn的写入工作。

<br/>

### 配置过程

Hadoop每个版本的高可用配置都不近相同，所以最好按照官方网站上的步骤配置。

<br/>

### HA管理

​	$>hdfs haadmin -transitionToActive nn1				//切成激活态

​	$>hdfs haadmin -transitionToStandby nn1				//切成待命态

​	$>hdfs haadmin -transitionToActive --forceactive nn2//强行激活

​	$>hdfs haadmin -failover nn1 nn2					//模拟容灾演示,从nn1切换到nn2