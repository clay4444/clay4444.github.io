---
title: Hbase和Hive的集成
tags:
  - Hbase
categories:
  - big-data
abbrlink: 67e1a306
date: 2018-03-08 15:52:47
---

### 将hbase的表影射到hive上

使用hive的查询语句。

```sql
CREATE TABLE t11(key string, name string) STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'  WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,cf:name") TBLPROPERTIES("hbase.table.name" = "ns1:t11");	
```