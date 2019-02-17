---
title: 详解hbase扫描、过滤、缓存等问题
tags:
  - Hbase
categories:
  - big-data
abbrlink: e73169bb
date: 2018-03-08 14:59:56
---

### hbase 大体架构

{% asset_img h1.png %}

<br/>

### 预先切割

创建表时，预先对表进行切割。

切割线是rowkey.

$hbase>create 'ns1:t2','f1',SPLITS=>['row3000','row6000']		切三份

<br/>

hbase：meta结果

{% asset_img h2.png %}

<br/>

web UI界面

 {% asset_img h3.png %}

<br/>

hdfs结果

 {% asset_img h4.png %}

<br/>

### 刷新入库

插入一条数据如果想立即在hdfs上看见结果，需要对数据进行刷新

 {% asset_img h5.png %}

<br/>

### 指定版本数

创建表时指定列族的版本数,该列族的所有列都具有相同数量的版本

```shell
$hbase>create 'ns1:t3',{NAME=>'f1',VERSIONS=>3}
```

创建表时，指定列族的版本数。这样指定版本之后，那么hbase中对f1这个列族中的数据只保留3个版本，过期的只有通过**原生扫描** 才能查询到(并没有被删除，只是被标记了删除而已)。

<br/>

```shell
$hbase>get 'ns1:t3','row1',{COLUMN=>'f1',VERSIONS=>4}
```

检索的时候，查询多少个版本

<br/>

按照指定版本数目查询：

```java
@Test
public void getWithVersions() throws IOException {
  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t3");
  Table table = conn.getTable(tname);
  Get get = new Get(Bytes.toBytes("row1"));
  //检索所有版本
  get.setMaxVersions();
  //get.setMaxVersions(2);     只查询两个版本。
  //get.setTimeRanfe(start, end);  查询指定时间范围
  Result r = table.get(get);
  List<Cell> cells = r.getColumnCells(Bytes.toBytes("f1"), Bytes.toBytes("name"));
  for(Cell c : cells){
    String f = Bytes.toString(c.getFamily());
    String col = Bytes.toString(c.getQualifier());
    long ts = c.getTimestamp();
    String val = Bytes.toString(c.getValue());
    System.out.println(f + "/" + col + "/" + ts + "=" + val);
  }
}
```

<br/>

### 其他查询方式

 {% asset_img h6.png %}

<br/>

通过时间戳查询示例  [ 左闭右开 ]

 {% asset_img h7.png %}

<br/>

通过时间戳指定时间范围查询

 {% asset_img h8.png %}

<br/>

### 原生扫描(专家)

原生扫描：

```shell
$hbase>scan 'ns1:t3',{COLUMN=>'f1',RAW=>true,VERSIONS=>10}	
```

包含标记了delete的数据

<br/>

删除数据：

```shell
$hbase>delete 'nd1:t3','row1','f1:name',148989875645
```

删除数据，标记为删除.小于该删除时间的数据都会被标记为删除。

<br/>

TTL：

time to live ,存活时间。

影响所有的数据，包括没有删除的数据。

超过该时间，原生扫描也扫不到数据。

```shell
$hbase>create 'ns1:tx' , {NAME=>'f1',TTL=>10,VERSIONS>=3}
```

<br/>

KEEP_DELETED_CELLS

删除key之后，数据是否还保留。 如果设置为true，删除数据之后通过原生扫描还能查询。但是如果ttl时间到了，即使设置它为true了，通过原生扫描也不会查询到删除的数据，也就是说**TTL是最优先控制的**

```shell
$hbase>create 'ns1:tx' , {NAME=>'f1',TTL=>10,VERSIONS=>3,KEEP_DELETED_CELLS=>true}
```

<br/>

整体步骤：

 {% asset_img h9.png %}

<br/>



### 缓存和批处理

 {% asset_img h10.png %}

<br/>

1. 开启服务器端扫描器缓存

   a)表层面(全局)     已经开启了。

```xml
<property>
  <name>hbase.client.scanner.caching</name>
  <!-- 整数最大值 -->
  <value>2147483647</value>
  <source>hbase-default.xml</source>
</property>
```

<br/>

​	b)操作层面

//设置量

```java
scan.setCaching(10);   //10是行数      默认是-1
```

<br/>

测试结果：

cache row nums : 1000			//632

cache row nums : 5000			//423

cache row nums : 1				//7359



<br/>

### 扫描器缓存

```java
//面向行级别的
@Test
public void getScanCache() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t1");
  Scan scan = new Scan();
  scan.setCaching(5000);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  long start = System.currentTimeMillis() ;
  Iterator<Result> it = rs.iterator();
  while(it.hasNext()){
    Result r = it.next();
    System.out.println(r.getColumnLatestCell(Bytes.toBytes("f1"), Bytes.toBytes("name")));
  }
  System.out.println(System.currentTimeMillis() - start);
}
```

<br/>

## 批量扫描

**面向列级别**

控制每次next()服务器端返回的列的个数。

 {% asset_img h11.png %}

<br/>

```java
scan.setBatch(5);				//每次next返回5列。
```

<br/>

实验：

 {% asset_img h12.png %}

图示：

 {% asset_img h13.png %}

<br/>

代码：

```java
/**
     * 测试缓存和批处理
     */
@Test
public void testBatchAndCaching() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();
  scan.setCaching(2);
  scan.setBatch(4);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    System.out.println("========================================");
    //得到一行的所有map,key=f1,value=Map<Col,Map<Timestamp,value>>
    NavigableMap<byte[], NavigableMap<byte[], NavigableMap<Long, byte[]>>> map = r.getMap();
    //
    for (Map.Entry<byte[], NavigableMap<byte[], NavigableMap<Long, byte[]>>> entry : map.entrySet()) {
      //得到列族
      String f = Bytes.toString(entry.getKey());
      Map<byte[], NavigableMap<Long, byte[]>> colDataMap = entry.getValue();
      for (Map.Entry<byte[], NavigableMap<Long, byte[]>> ets : colDataMap.entrySet()) {
        String c = Bytes.toString(ets.getKey());
        Map<Long, byte[]> tsValueMap = ets.getValue();
        for (Map.Entry<Long, byte[]> e : tsValueMap.entrySet()) {
          Long ts = e.getKey();
          String value = Bytes.toString(e.getValue());
          System.out.print(f + "/" + c + "/" + ts + "=" + value + ",");
        }
      }
    }
    System.out.println();
  }
}
```

<br/>

结果：

========================================

f1/id/1490595148588=1,f2/addr/1490595182150=hebei,f2/age/1490595174760=12,

========================================

f2/id/1490595164473=1,f2/name/1490595169589=tom,

<br/>

========================================

f1/id/1490595196410=2,f1/name/1490595213090=tom2.1,f2/addr/1490595264734=tangshan,

========================================

f2/age/1490595253996=13,f2/id/1490595233568=2,f2/name/1490595241891=tom2.2,

<br/>

========================================

f1/age/1490595295427=14,f1/id/1490595281251=3,f1/name/1490595289587=tom3.1,

========================================

f2/addr/1490595343690=beijing,f2/age/1490595336300=14,f2/id/1490595310966=3,

========================================

f2/name/1490595327531=tom3.2,

<br/>



### Filter 过滤器

 {% asset_img h14.png %}

<br/>

```java
/**
 * 测试RowFilter过滤器
 */
@Test
public void testRowFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t1");
  Scan scan = new Scan();
  RowFilter rowFilter = new RowFilter(CompareFilter.CompareOp.LESS_OR_EQUAL, new BinaryComparator(Bytes.toBytes("row0100")));
  scan.setFilter(rowFilter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    System.out.println(Bytes.toString(r.getRow()));
  }
}
```

<br/>

```java
/**
 * 测试FamilyFilter过滤器
*/
@Test
public void testFamilyFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();
  FamilyFilter filter = new FamilyFilter(CompareFilter.CompareOp.LESS, new BinaryComparator(Bytes.toBytes("f2")));
  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    System.out.println(f1id + " : " + f2id);
  }
}
```

<br/>

```java
/**
 * 测试QualifierFilter(列过滤器)
 */
@Test
public void testColFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();
  QualifierFilter colfilter = new QualifierFilter(CompareFilter.CompareOp.EQUAL, new BinaryComparator(Bytes.toBytes("id")));
  scan.setFilter(colfilter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + f2name);
  }
}
```

<br/>

```java
/**
* 测试ValueFilter(值过滤器)
* 过滤value的值，含有指定的字符子串
*/
@Test
public void testValueFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();
  ValueFilter filter = new ValueFilter(CompareFilter.CompareOp.EQUAL, new SubstringComparator("to"));
  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```

<br/>

```java
/**
 * 依赖列过滤器
 */
@Test
public void testDepFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();
  //只取出f2这个列族中addr这一列值是"beijing"的这一列。
  DependentColumnFilter filter = new DependentColumnFilter(Bytes.toBytes("f2"),
                                                           Bytes.toBytes("addr"),
                                                           false,
                                                           CompareFilter.CompareOp.NOT_EQUAL,
                                                           new BinaryComparator(Bytes.toBytes("beijing"))
                                                          );

  //ValueFilter filter = new ValueFilter(CompareFilter.CompareOp.EQUAL, new SubstringComparator("to"));
  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```

<br/>

```java
 /**
  * 单列值value过滤，
  * 如果value不满足，整行滤掉
  */
@Test
public void testSingleColumValueFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();
  SingleColumnValueFilter filter = new SingleColumnValueFilter(Bytes.toBytes("f2",
                                                                             Bytes.toBytes("name"),
                                                                             CompareFilter.CompareOp.NOT_EQUAL),
                                                               new BinaryComparator(Bytes.toBytes("tom2.1")));

  //ValueFilter filter = new ValueFilter(CompareFilter.CompareOp.EQUAL, new SubstringComparator("to"));
  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```

<br/>

```java
/**
     * 单列值排除过滤器,去掉过滤使用的列,对列的值进行过滤
     */
@Test
public void testSingleColumValueExcludeFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();
  SingleColumnValueExcludeFilter filter = new SingleColumnValueExcludeFilter(Bytes.toBytes("f2"),
                                                                             Bytes.toBytes("name"),
                                                                             CompareFilter.CompareOp.EQUAL,
                                                                             new BinaryComparator(Bytes.toBytes("tom2.2")));

  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```

<br/>

```java
/**
     * 前缀过滤,是rowkey过滤. where rowkey like 'row22%'
     */
@Test
public void testPrefixFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t1");
  Scan scan = new Scan();
  PrefixFilter filter = new PrefixFilter(Bytes.toBytes("row222"));

  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```

<br/>

```java
/**
     * 分页过滤,是rowkey过滤,在region上扫描时，对每次page设置的大小。
    *  也就是每个region上取10个。
     * 返回到到client，涉及到每个Region结果的合并。
     */
@Test
public void testPageFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t1");
  Scan scan = new Scan();

  PageFilter filter = new PageFilter(10);

  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```

<br/>

```java
/**
     * keyOnly过滤器，只提取key,丢弃value.
     * @throws IOException
     */
@Test
public void testKeyOnlyFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t1");
  Scan scan = new Scan();

  KeyOnlyFilter filter = new KeyOnlyFilter();

  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```

<br/>

```java
/**
     * ColumnPageFilter,列分页过滤器，过滤指定范围列，
     * select ,,a,b from ns1:t7
     */
@Test
public void testColumnPageFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();

  ColumnPaginationFilter filter = new ColumnPaginationFilter(2,2);   
  //第一个是开始列，第二个是偏移量(每行的)

  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```

<br/>

```java
/*
* like 查询
*/
@Test
public void testLike() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();

  ValueFilter filter = new ValueFilter(CompareFilter.CompareOp.EQUAL,
                                       new RegexStringComparator("^tom2")
                                      );

  scan.setFilter(filter);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```

<br/>



FilterList  过滤器（复杂查询）

```java
@Test
public void testComboFilter() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t7");
  Scan scan = new Scan();

  //where ... f2:age <= 13
  SingleColumnValueFilter ftl = new SingleColumnValueFilter(
    Bytes.toBytes("f2"),
    Bytes.toBytes("age"),
    CompareFilter.CompareOp.LESS_OR_EQUAL,
    new BinaryComparator(Bytes.toBytes("13"))		//注意这里的13数字要加引号
  );

  //where ... f2:name like %t
  SingleColumnValueFilter ftr = new SingleColumnValueFilter(
    Bytes.toBytes("f2"),
    Bytes.toBytes("name"),
    CompareFilter.CompareOp.EQUAL,
    new RegexStringComparator("^t")
  );
  //ft
  FilterList ft = new FilterList(FilterList.Operator.MUST_PASS_ALL);
  ft.addFilter(ftl);
  ft.addFilter(ftr);

  //where ... f2:age > 13
  SingleColumnValueFilter fbl = new SingleColumnValueFilter(
    Bytes.toBytes("f2"),
    Bytes.toBytes("age"),
    CompareFilter.CompareOp.GREATER,
    new BinaryComparator(Bytes.toBytes("13"))
  );

  //where ... f2:name like %t
  SingleColumnValueFilter fbr = new SingleColumnValueFilter(
    Bytes.toBytes("f2"),
    Bytes.toBytes("name"),
    CompareFilter.CompareOp.EQUAL,
    new RegexStringComparator("t$")
  );
  //ft
  FilterList fb = new FilterList(FilterList.Operator.MUST_PASS_ALL);
  fb.addFilter(fbl);
  fb.addFilter(fbr);


  FilterList fall = new FilterList(FilterList.Operator.MUST_PASS_ONE);
  fall.addFilter(ft);
  fall.addFilter(fb);

  scan.setFilter(fall);
  Table t = conn.getTable(tname);
  ResultScanner rs = t.getScanner(scan);
  Iterator<Result> it = rs.iterator();
  while (it.hasNext()) {
    Result r = it.next();
    byte[] f1id = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("id"));
    byte[] f2id = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("id"));
    byte[] f1name = r.getValue(Bytes.toBytes("f1"), Bytes.toBytes("name"));
    byte[] f2name = r.getValue(Bytes.toBytes("f2"), Bytes.toBytes("name"));
    System.out.println(f1id + " : " + f2id + " : " + Bytes.toString(f1name) + " : " + Bytes.toString(f2name));
  }
}
```



<br/>

### 计数器

```shell
$hbase>incr 'ns1:t8','row1','f1:click',1
$hbase>get_counter 'ns1:t8','row1','f1:click'
```

<br/>

[API编程]

~~~java
@Test
public void testIncr() throws IOException {

  Configuration conf = HBaseConfiguration.create();
  Connection conn = ConnectionFactory.createConnection(conf);
  TableName tname = TableName.valueOf("ns1:t8");
  Table t = conn.getTable(tname);
  Increment incr = new Increment(Bytes.toBytes("row1"));
  incr.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("daily"),1);
  incr.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("weekly"),10);
  incr.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("monthly"),100);
  t.increment(incr);
}
~~~