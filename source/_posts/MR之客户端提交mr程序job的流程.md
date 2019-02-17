---
title: MR之客户端提交mr程序job的流程
categories:
  - big-data
tags:
  - MapReduce
abbrlink: a9435451
date: 2017-09-25 15:45:26
---



## 客户端提交job

### 简明图示

{% asset_img 11.png %}



### 解析

在job.submit() 这个方法的执行过程中先去通过Proxy对象去获取存放路径，然后：

1. 调用getSplit()方法切片，形成资源切片规划，序列化成文件  job.split
2. 把客户端设置的job的相关参数写到一个文件中       job.xml
3. 获取客户端生成的job的jar包           job.jar

最后把这三个文件提交到Proxy返回的路径中，等待后续的流程处理。