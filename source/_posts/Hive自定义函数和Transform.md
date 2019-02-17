---
title: Hive自定义函数和Transform
tags:
  - Hive
categories:
  - big-data
abbrlink: 2dd0f618
date: 2017-07-02 18:53:45
---



### Hive自定义函数和Transform

​	当Hive提供的内置函数无法满足你的业务处理需要时，此时就可以考虑使用用户自定义函数（UDF：user-defined function）

<br/>

#### 自定义函数类别

UDF  作用于单个数据行，产生一个数据行作为输出。（数学函数，字符串函数）

UDAF（用户定义聚集函数）：接收多个输入数据行，并产生一个输出数据行。（count，max）

<br/>

#### UDF开发实例

1、先开发一个java类，继承UDF，并重载evaluate方法

~~~java
package cn.itcast.bigdata.udf
import org.apache.hadoop.hive.ql.exec.UDF;
import org.apache.hadoop.io.Text;

public final class Lower extends UDF{
	public Text evaluate(final Text s){
		if(s==null){return null;}
		return new Text(s.toString().toLowerCase());
	}
}
~~~

2、打成jar包上传到服务器

3、将jar包添加到hive的classpath

~~~shell
hive>add JAR /home/hadoop/udf.jar;
~~~

4、创建临时函数与开发好的java class关联

~~~
Hive>create temporary function toprovince as 'org.clay.bigdata.udf.ToProvince';
~~~

5、即可在hql中使用自定义的函数strip 

~~~sql
Select strip(name),age from t_test;
~~~

<br/>

#### Transform实现

Hive的 TRANSFORM 关键字**提供了在**SQL  **中调用自写脚本的功能**

适合实现Hive中没有的功能又不想写UDF的情况

<br/>

使用示例1：下面这句sql就是借用了weekday_mapper.py对数据进行了处理.

~~~sql
CREATE TABLE u_data_new (
  movieid INT,
  rating INT,
  weekday INT,
  userid INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t';

add FILE weekday_mapper.py;

INSERT OVERWRITE TABLE u_data_new
SELECT
  TRANSFORM (movieid, rating, unixtime,userid)
  USING 'python weekday_mapper.py'
  AS (movieid, rating, weekday,userid)
FROM u_data;
~~~

其中weekday_mapper.py内容如下

~~~python
#!/bin/python
import sys
import datetime

for line in sys.stdin:
  line = line.strip()
  movieid, rating, unixtime,userid = line.split('\t')
  weekday = datetime.datetime.fromtimestamp(float(unixtime)).isoweekday()
  print '\t'.join([movieid, rating, str(weekday),userid])
~~~