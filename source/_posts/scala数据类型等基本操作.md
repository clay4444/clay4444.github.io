---
title: scala数据类型等基本操作
categories:
  - scala
tags:
  - 快学scala读书笔记
abbrlink: 90c61e02
date: 2018-03-12 14:45:55
---

### scala

java语言的脚本化。

<br/>

### REPL

read + evaluate + print + loop

<br/>

### 常用类型

7种数值类型：Byte  Char   Short   Int  long   Float   Double  

1种 ：Boolean

和Java不同的是：**这些类型都是类，scala并不刻意区分基本类型和引用类型**

<br/>

### 基本使用

<br/>

#### 定义变量，

```scala
scala>var a = 100			//变量
```

#### 定义常量

```scala
scala>val a = 100			//常量，不能重新赋值。
```

#### 定义类型

```scala
scala>val a:String = "hello" ;
scala>a = "world"			//wrong  val 定义的是常量
```

#### 操作符重载：所有的操作都是方法

```scala
scala>1 + 2
scala>1.+(2)				// 所有的操作都是方法
```

#### 函数和方法的区别

scala函数,没有对象.

scala方法，通过对象调用。

```scala
import scala.math._		//_ ===> *   通配符
min(1,2)
```

<br/>

```scala
scala>1.toString				//方法
scala>1.toString()				//方法形式
scala>1 toString				//运算符方式
```

<br/>

#### apply：目的：千方百计节省代码量

```scala
scala>"hello".apply(1)			// String内置的
scala>"hello"(1)				//等价于xxx.apply()     
```

#### 条件表达式,

scala的表达式有值,是**最后一条语句的值**

```scala
scala>val x = 1 ;
scala>val b = if (x > 0) 1 else -1 ;				//x>0 一定要加( )
```

<br/>

#### Any 是Int和String的超类。

<br/>

#### 类型转换

```scala
scala>1.toString()
scala>1.toString			 //和上面等价
scala>"100".toInt			//"100".toInt()  是错误的,不能加()
```

#### 粘贴复制模式

```scala
scala>:paste
ctrl + d					//结束粘贴模式
```

#### 输出

```scala
scala>print("hello")
scala>println("hello")
scala>printf("name is %s , age is %d", "tom",12);
```

#### 读行

```scala
val password = readLine("请输入密码 : ") ;
```

#### while循环

```scala
scala>:paste
var i = 0 ;
while(i < 10 ){
  println(i) ;
  i += 1;
}
```

#### 99乘法表

```scala
scala>:paste
var row = 1 ; 
while(row <= 9 ){
  var col = 1 ; 
  while(col <= row){
    printf("%d x %d = %d\t",col,row,(row * col)) ;
    col += 1 ;
  }
  println();
  row += 1 ;
}
```

#### 百钱买白鸡问题.

100块钱100只鸡。

公鸡:5块/只

母鸡:3块/只

小鸡:1块/3只

```scala
//公鸡
var cock = 0 ;			//公鸡
while(cock <= 20){		//公鸡最多买20只
  //母鸡
  var hen = 0 ;
  while(hen <= 100/3){	//母鸡最多买33只
    var chicken = 0 ;	//小鸡
    while(chicken <= 100){
      var money = cock * 5 + hen * 3 + chicken / 3 ;
      var mount = cock + hen + chicken ;
      if(money == 100 && mount == 100){
        println("cock : %d , hen : %d , chicken : %d",cock,hen,chicken) ;
      }
    }
  }
}
```

#### for循环

```scala
//to []			//两边都是闭区间
scala>for (x <- 1 to 10){
  println(x) ;
}
```

<br/>

```scala
//until [1,...10)		//半闭半开区间
scala>for (x <- 1 until 10){
  println(x) ;
}
```

<br/>

#### scala没有break continue语句。

可以使用Breaks对象的break()方法。

```scala
scala>import scala.util.control.Breaks._
scala>for(x <- 1 to 10) {break() ; print(x)} ;
```

<br/>

#### for循环高级

双循环, **守卫条件**

```scala
scala>for(i <- 1 to 3 ; j <- 1 to 4 if i != j )
{printf("i = %d, j = %d , res = %d ",i,j,i*j);println()} ;	
```

**yield**，是循环中处理每个元素，**产生新集合**

```scala
scala>for (x <- 1 to 10 ) yield x % 2 ; 
```

{% asset_img z1.png %}

<br/>

#### 定义函数

```scala
def add(a:Int,b:Int):Int = {
  var c = a + b  ;
  return c  ;
}
```

<br/>

```scala
scala>def add(a:Int,b:Int):Int = a + b 
```

<br/>

#### 递归

scala实现递归 n! = n * (n - 1)!

**递归函数必须显式定义返回类型**

```scala
scala>def fac(n:Int):Int = if(n ==1 ) 1 else n * fac(n-1) ;
```

<br/>

#### 函数的默认值和命名参数

```scala
scala>def decorate(prefix:String = "[[",str:String,suffix:String = "]]") = {
  prefix + str + suffix 
}
```

<br/>

```scala
scala>decorate(str="hello")
	scala>decorate(str="hello",prefix="<<")
```

<br/>

#### 变长参数

```scala
scala>def sum(a:Int*) = {
  var s = 0 ;
  for (x <- a) s += x;
  s
}
```

<br/>

```scala
scala>add(1 to 10)			//wrong
scala>add(1 to 10:_*)		//将1 to 10当做序列处理
scala>def sum(args:Int*):Int = {if (args.length == 0) 0 else args.head + sum(args.tail:_*)}	
//累加和
```

<br/>

#### 定义过程

没有返回值，没有=号。

```scala
scala>def out(a:Int){
  println(a) ;
}
```

<br/>

#### lazy延迟计算

```scala
scala>lazy val x = scala.io.Source.fromFile("d:/scala/buy.scala00").mkString
x:<lazy>		//返回值
x				//再次访问时才会打印结果
```

<br/>

#### 异常

```scala
scala>
try{
  "hello".toInt;
}
catch{					//交给
  case _:Exception => print("xxxx") ;		//异常匹配
  case ex:java.io.IOException => print(ex)	
}
```

<br/>

#### _的意义

1. 通配相当于*	---------> import java.io._
2. 1 to 10 :_*  --------->转成序列
   1. case _:Exception    => print("xxxx") ; ------->变量只用一次就不用了。

#### 数组(定长)

java : int[] arr = int int[4] ;

```scala
scala>var arr = new Array[Int](10);			//apply(10)
scala>var arr = Array(1,2,3,4,);			//类型推断
scala>arr(0)							//按照下标访问元素
```

<br/>

#### 变长数组

```scala
scala>import scala.collection.mutable.ArrayBuffer			//mutable是可变的
scala>val buf = ArrayBuffer[Int]();						//创建数组缓冲区对象

//+=在末尾追加
scala>buf += 1

//追加任何集合
scala>buf ++= (1 to 10)

//trimStart,从起始移除元素
scala>buf.trimStart(2)		//参数2是移除两个的意思

//trimEnd,从末尾移除元素
scala>buf.trimEnd(2)

//insert,在0元素位置插入后续数据
scala>buf.insert(0,1,2)

//remove按照索引移除
scala>buf.remove(0)

//toArray
scala>buf.toArray
```

<br/>

#### 数组操作

```scala
scala>for (x <- 1 to 10 if x % 2 ==0) yield x 	//偶数集合
scala>var a = Array(1 to 10)				  //这样插入数组中的只是一个Range对象
scala>var a = Array(1 to 10:_*)				  //正确插入元素的方法
```

{% asset_img z2.png %}

**把范围对象转成序列**	:_*

注意：

~~~scala
new Array(100)			//100个元素
Array(100)				//1个元素，值是100，apply();
~~~

数组常用方法：

```scala
scala>arr.sum
scala>arr.min
scala>arr.max
```

排序：

```scala
scala>import scala.util.Sorting._
scala>val arr = Array(1,4,3,2)
scala>quickSort(arr)				//arr有序
```

Array.mkString	//数组变字符串

```scala
scala>arr.mkString("<<",",",">>")	//<<1,2,3,4>>
```

多维数组：

```scala
//一维数组
scala>var arr:Array[Int] =new Array[Int](4);
//二维数组,3行4列
scala>val arr = Array.ofDim[Int](3,4)
//下标访问数组元素
scala>arr(0)(1)
scala>arr.length
```

和java对象交互，导入转换类型,使用的隐式转换

```scala
scala>import scala.collection.JavaConversions.bufferAsJavaList
scala>val buf = ArrayBuffer(1,2,3,4);
scala>val list:java.util.List[Int] = buf ;	//转换
```

<br/>

#### 映射Map ： 不可变

key->value

scala.collection.immutable.Map[Int,String] =**不可变集合**

```scala
scala>val map = Map(100->"tom",200->"tomas",300->"tomasLee")
//通过key访问value
scala>map(100)
scala>val newmap = map + (4->"ttt")	//产生一个新的map
```

<br/>

#### 映射Map ：可变

```scala
scala>val map = new scala.collection.mutable.HashMap[Int,Int]
scala>val map = scala.collection.mutable.HashMap[Int,Int]()		//apply()方法
scala>map += (1->100,2->200)		//追加	
scala>map -= 1					//移除元素
```

<br/>

#### 迭代map

```scala
scala>for ((k,v)<- map) println(k + ":::" + v);

//使用yield操作进行倒排序(kv对调),
scala>for ((k,v)<- map) yield (v,k);
```

<br/>

#### 元组： tuple，元数最多22-->Tuple22

```scala
scala>val t = (1,"tom",12) ;

//访问元组指定元，注意：从1开始，而不是从0开始
scala>t._2
scala>t _2

//直接取出元组中的各分量
scala>val (a,b,c) = t		//a=1,b="tom",c=12
```

<br/>

#### 数组的zip,	咬合

```scala
//西门庆 -> 潘金莲  牛郎 -> 侄女  ,
scala>val hus = Array(1,2,3);
scala>val wife = Array(4,5,6);
scala>hus.zip(wife)		//(1,4),(2,5),(3,6)
```