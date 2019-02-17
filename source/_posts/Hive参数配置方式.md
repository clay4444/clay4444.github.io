---
title: Hive参数配置方式
tags:
  - Hive
categories:
  - big-data
abbrlink: 5ec70f63
date: 2017-06-29 18:39:56
---



### 关于Hive的参数配置

​	开发Hive应用时，不可避免地需要设定Hive的参数。设定Hive的参数可以调优HQL代码的执行效率，或帮助定位问题。然而实践中经常遇到的一个问题是，为什么设定的参数没有起作用？这通常是错误的设定方式导致的。

<br/>

#### 对于一般参数，有以下三种设定方式：

**1.配置文件 **

**2.命令行参数 **

**3.参数声明 **

<br/>

**配置文件**：

Hive的配置文件包括：

l 用户自定义配置文件：$HIVE_CONF_DIR/hive-site.xml

l 默认配置文件：$HIVE_CONF_DIR/hive-default.xml

**用户自定义配置会覆盖默认配置 **。

<br/>

**命令行参数**：

启动Hive（客户端或Server方式）时，可以在命令行添加-hiveconf param=value来设定参数，例如：

bin/hive-hiveconf hive.root.logger=INFO,console

这一设定对本次启动的Session（对于Server方式启动，则是所有请求的Sessions）有效。

<br/>

**参数声明**：

可以在HQL中使用SET关键字设定参数，例如：

setmapred.reduce.tasks=100;

这一设定的作用域也是session级的。

<br/>

上述三种设定方式的优先级依次递增。即参数声明覆盖命令行参数，命令行参数覆盖配置文件设定。注意某些系统级的参数，例如log4j相关的设定，必须用前两种方式设定，因为那些参数的读取在Session建立以前已经完成了。