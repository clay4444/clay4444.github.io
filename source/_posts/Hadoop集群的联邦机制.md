---
title: Hadoop集群的联邦机制
categories:
  - big-data
tags:
  - HA机制
abbrlink: ac72d308
date: 2017-09-22 10:31:32
---

### 联邦机制

#### 原因

如果一个namenode一个不够用，数据特别多的话，就要进行**水平扩展**。

<br/>

### 简明图示

{% asset_img 11.png %}

<br/>

### 问题及解决办法

要提供不同的服务。存储不同的数据，就面临很大问题，例如一个数据修改了，另一个人通过另一个namenode查询，发现没改。。。。

解决办法：分目录，一个负责/japanese,   一个负责/art-x，类似于磁盘中的C盘，D盘。

nn1和nn2 存储数据一样，是高可用，clusterID一致。

nn3和nn4存储数据一样，是高可用，clusterID一致。

两个clusterID通过不同的目录来提供不同服务。

他们共用所有的datanade，这时，datanode配置文件上的blockpool能解决问题了，

blockpool1代表属于clusterID1的block块，

blockpool2代表属于clusterID2的block块，

