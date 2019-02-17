---
title: 关于mapreduce程序开发的一些总结
tags:
  - MapReduce
categories:
  - big-data
abbrlink: ecfdb360
date: 2017-09-30 20:38:22
---



### MR

mapreduce在编程的时候，基本上一个固化的模式，没有太多可灵活改变的地方，除了以下几处：

<br/>

- 1、输入数据接口：InputFormat   --->     FileInputFormat(文件类型数据读取的通用抽象类)  DBInputFormat （数据库数据读取的通用抽象类）

     默认使用的实现类是： TextInputFormat     job.setInputFormatClass(TextInputFormat.class) TextInputFormat的功能逻辑是：一次读一行文本，然后将该行的起始偏移量作为key，行内容作为value返回

  <br/>

- 2、逻辑处理接口： Mapper  

     完全需要用户自己去实现其中  map()   setup()   clean()   

  <br/>

- 3、map输出的结果在shuffle阶段会被partition以及sort，此处有两个接口可自定义：

  - Partitioner

  ​	有默认实现 HashPartitioner，逻辑是  根据key和numReduces来返回一个分区号； key.hashCode()&Integer.MAXVALUE % numReduces)

  ​	通常情况下，用默认的这个HashPartitioner就可以，如果业务上有特别的需求，可以自定义

  - Comparable

  ​	当我们用自定义的对象作为key来输出时，就必须要实现WritableComparable接口，override其中的compareTo()方法

  <br/>

- 4、reduce端的数据分组比较接口 ： Groupingcomparator

  ​	reduceTask拿到输入数据（一个partition的所有数据）后，首先需要对数据进行分组，其分组的默认原则是key相同，然后对每一组kv数据调用一次reduce()方法，并且将这一组kv中的第一个kv的key作为参数传给reduce的key，将这一组数据的value的迭代器传给reduce()的values参数

  <br/>

  ​	利用上述这个机制，我们可以实现一个高效的分组取最大值的逻辑：自定义一个bean对象用来封装我们的数据，然后改写其compareTo方法产生倒序排序的效果，然后自定义一个Groupingcomparator，将bean对象的分组逻辑改成按照我们的业务分组id来分组（比如订单号），这样，我们要取的最大值就是reduce()方法中传进来key

  <br/>

- 5、逻辑处理接口：Reducer

  完全需要用户自己去实现其中  reduce()   setup()   clean()   

  <br/>

- 6、输出数据接口： OutputFormat  ---> 有一系列子类  FileOutputformat  DBoutputFormat  ..... 默认实现类是TextOutputFormat，功能逻辑是：  将每一个KV对向目标文本文件中输出为一行