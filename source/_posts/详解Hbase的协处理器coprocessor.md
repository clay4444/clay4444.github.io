---
title: 详解Hbase的协处理器coprocessor
tags:
  - Hbase
categories:
  - big-data
abbrlink: f7fb5894
date: 2018-03-08 15:21:21
---

### coprocessor   协处理器

批处理的，等价于存储过程或者触发器

<br/>

执行顺序

{% asset_img h1.png %}

<br/>

执行状态

{% asset_img h2.png %}

{% asset_img h3.png %}

<br/>

图示：系统协处理器优先加载

{% asset_img h4.png %}

<br/>

### [Observer]

{% asset_img h5.png %}

观察者,类似于触发器，基于事件。发生动作时，回调相应方法。

<br/>

1. RegionObserver：用户可以使用这种协处理器处理数据修改事件，他们与表的region紧密联系
2. MasterObserver：可以被用作管理或DDL类型的操作，这些是集群级事件。
3. WAlObserver：提供控制WAL的钩子函数

<br/>

### [Endpoint]

终端,类似于存储过程。

<br/>

{% asset_img h6.png %}



<br/>

### 使用过程

1. 加载

```xml
[hbase-site.xml]
<property>
  <name>hbase.coprocessor.region.classes</name>
  <value>coprocessor.RegionObserverExample, coprocessor.AnotherCoprocessor</value>
</property>
<property>
  <name>hbase.coprocessor.master.classes</name>
  <value>coprocessor.MasterObserverExample</value>
</property>
<property>
  <name>hbase.coprocessor.wal.classes</name>
  <value>coprocessor.WALObserverExample, bar.foo.MyWALObserver</value>
</property>
```

<br/>

1. 自定义观察者

```java
[MyRegionObserver]

import org.apache.hadoop.hbase.Cell;
import org.apache.hadoop.hbase.CoprocessorEnvironment;
import org.apache.hadoop.hbase.client.Delete;
import org.apache.hadoop.hbase.client.Durability;
import org.apache.hadoop.hbase.client.Get;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.coprocessor.BaseRegionObserver;
import org.apache.hadoop.hbase.coprocessor.ObserverContext;
import org.apache.hadoop.hbase.coprocessor.RegionCoprocessorEnvironment;
import org.apache.hadoop.hbase.regionserver.wal.WALEdit;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.FileWriter;
import java.io.IOException;
import java.util.List;

/**
		 * 自定义区域观察者
		 */
public class MyRegionObserver extends BaseRegionObserver{

  private void outInfo(String str){
    try {
      FileWriter fw = new FileWriter("/home/centos/coprocessor.txt",true);
      fw.write(str + "\r\n");
      fw.close();
    } catch (Exception e) {
      e.printStackTrace();
    }

  }
  public void start(CoprocessorEnvironment e) throws IOException {
    super.start(e);
    outInfo("MyRegionObserver.start()");
  }

  public void preOpen(ObserverContext<RegionCoprocessorEnvironment> e) throws IOException {
    super.preOpen(e);
    outInfo("MyRegionObserver.preOpen()");
  }

  public void postOpen(ObserverContext<RegionCoprocessorEnvironment> e) {
    super.postOpen(e);
    outInfo("MyRegionObserver.postOpen()");
  }

  @Override
  public void preGetOp(ObserverContext<RegionCoprocessorEnvironment> e, Get get, List<Cell> results) throws IOException {
    super.preGetOp(e, get, results);
    String rowkey = Bytes.toString(get.getRow());
    outInfo("MyRegionObserver.preGetOp() : rowkey = " + rowkey);
  }

  public void postGetOp(ObserverContext<RegionCoprocessorEnvironment> e, Get get, List<Cell> results) throws IOException {
    super.postGetOp(e, get, results);
    String rowkey = Bytes.toString(get.getRow());
    outInfo("MyRegionObserver.postGetOp() : rowkey = " + rowkey);
  }

  public void prePut(ObserverContext<RegionCoprocessorEnvironment> e, Put put, WALEdit edit, Durability durability) throws IOException {
    super.prePut(e, put, edit, durability);
    String rowkey = Bytes.toString(put.getRow());
    outInfo("MyRegionObserver.prePut() : rowkey = " + rowkey);
  }

  @Override
  public void postPut(ObserverContext<RegionCoprocessorEnvironment> e, Put put, WALEdit edit, Durability durability) throws IOException {
    super.postPut(e, put, edit, durability);
    String rowkey = Bytes.toString(put.getRow());
    outInfo("MyRegionObserver.postPut() : rowkey = " + rowkey);
  }

  @Override
  public void preDelete(ObserverContext<RegionCoprocessorEnvironment> e, Delete delete, WALEdit edit, Durability durability) throws IOException {
    super.preDelete(e, delete, edit, durability);
    String rowkey = Bytes.toString(delete.getRow());
    outInfo("MyRegionObserver.preDelete() : rowkey = " + rowkey);
  }

  @Override
  public void postDelete(ObserverContext<RegionCoprocessorEnvironment> e, Delete delete, WALEdit edit, Durability durability) throws IOException {
    super.postDelete(e, delete, edit, durability);
    String rowkey = Bytes.toString(delete.getRow());
    outInfo("MyRegionObserver.postDelete() : rowkey = " + rowkey);
  }
}
```

<br/>

1. 注册协处理器并分发

```xml
<property>
  <name>hbase.coprocessor.region.classes</name>
  <value>org.clay.hbaseDemo.test.MyRegionObserver</value>
</property>
```

<br/>

1. 导出jar包。

<br/>

1. 复制jar到共享目录，分发到jar到hbase集群的hbase lib目录下.

<br/>

1. 重启hbase集群

<br/>

1. 查看结果

{% asset_img h7.png %}

<br/>

{% asset_img h8.png %}

<br/>

{% asset_img h9.png %}

<br/>

每个方法的执行数目和region的数量是一样的，因为这是**RegionObserver**