---
title: scala函数式编程和集合进阶
categories:
  - scala
tags:
  - 快学scala读书笔记
abbrlink: b391d3f0
date: 2018-03-12 17:19:22
---

### 函数式编程

注意点：

如果使用了匿名函数，那么匿名函数一定不能有返回值，必须有返回的话，需要写在函数的结尾。

{% asset_img z1.png %}

<br/>

```scala
scala>def add(a:Int,b:Int) = a + b

scala>val f = add _					//将函数赋值给一个变量,函数类型变量。

scala>def multi(n:Int) = n * 2
scala>def f = multi _				//_ 表示取出函数本身

scala>Array(1,2,3,4).map(f)
```

<br/>

### 匿名函数

```scala
scala>(n:Double)=>3 * n						//
scala>val f = (n:Double)=>3 * n				//也是匿名函数
scala>Array(1,2,3,4).map((x) => x * 3);		 //
scala>Array(1,2,3,4).map{(x) => x * 3};
```

<br/>

### 练习题目

两个数字，第一个数字大于0，返回这两个数的和，否则：返回这两个数字的差

f1:add

f2:sub

```scala
def call(a:Int,b:Int,f1:(Int,Int)=>Int,f2:(Int,Int)=>Int)={
  if(a > 0){
    f1(a,b) ;
  }
  else{
    f2(a,b) ;
  }
}

def add(a:Int,b:Int) = a + b
def sub(a:Int,b:Int) = a- b

val f1 = add _
val f2 = sub _

call(1,2,f1,f2)			//3
call(1,2,add _ ,sub _)	//
call(1,2,add,sub)	//
```

<br/>

提升：不再返回一个值，而是返回一个线性函数，斜率是上次返回的值

```scala
def call(a:Int,b:Int,f1:(Int,Int)=>Int,f2:(Int,Int)=>Int)= {
  var n = 0 ;
  if(a > 0){
    n = f1(a,b) ;
  }
  else{
    n = f2(a,b) ;
  }

  //
  def multi(x:Int) = x * n ;
  multi _
}

call(1,2,(a:Int,b:Int)=>a + b , (a:Int,b:Int)=> a- b)(100)		//right
call(1,2,a,b=>a + b , (a,b)=> a- b)(100)					//wrong	一个参数时才可以去掉()
```

<br/>



### 高阶函数演变

```scala
//定义高阶函数
def valueAt(f:(Double)=>Double) = f(0.25)

valueAt(ceil _)		//ceil：比当前数字小的最小的整数

//也就是说返回了一个函数，参数是duble类型，返回值也是Double类型
def mulby(factor : Double) = (x:Double) => x * factor
//mulby(2)(2)  返回4
```

<br/>

函数推断：

```scala
def valueAt(f:(Double)=>Double) = f(0.25)
valueAt((x:Double)=>x * 3)				//定义类型
valueAt((x)=>x * 3)						//推断类型
valueAt(x=>x * 3)						//省略()
valueAt(3 * _)							//参数在右侧出现1次，就可以使用_代替。
```

<br/>

高级函数：

```scala
scala>val arr = Array(1,2,3,4)
scala>arr.map(2 * _);					//每个元素x2
scala>arr.map((e:Int)=> e * 2);			//每个元素x2
scala>arr.map(_ * 2);					//每个元素x2
```

<br/>

数组中的元素先扩大3倍，然后再过滤出偶数

```scala
scala>var arr = Array(1,2,3,4);
scala>arr.map(3 * _).filter(e => e % 2 ==0) 
scala>arr.map(3 * _).filter(_ % 2 ==0) 
```

<br/>

输出三角形：

```scala
scala> "*" * 5	 打印5个*
scala> (1 to 20).map("*" * _).foreach(println)
```

<br/>

reduceLeft，由左至右：

```scala
scala>var arr = Array(1,2,3,4);
scala>arr.reduceLeft(_-_)
//1,2,3 ==> (1 - 2) -3)  = -4
```

<br/>

reduceRight,由右至左：

```scala
scala>arr.reduceRight(_-_)
//1,2,3 ==>1 - (2 - 3)= 2
//1,2,3,4 ==> 1 - (2 - (3 - 4)) = -2
//1,2,3,4 ==> 1 - (2 - (3 - 4)) = -2
```

<br/>

### 柯里化

{% asset_img z2.png %}

```scala
scala>def mul(a:Int,b:Int) = a  * b;				//1
scala>mul(1,2)

scala>def mulone(a:Int) = {(b:Int) => a * b ;}		//2
scala>mulone(1)(2)
```

​	其本质无非就是高阶函数的转化而已，mulone先传入一个参数作为a, mulone这个函数再返回一个函数，那么mulone(1) 就是再调用这个新的函数，新的函数又会接受b作为新的参数再执行a*b，所以mulone(1)(2)就是再调用这个新的函数，执行的就是1*2的过程

​	所以**1和2是等价的**

<br/>

### 控制抽象

定义过程，启动分线程执行block代码.

这样的方式比较灵活，没有把要执行的过程写在run方法中，而是定义在外面，通过函数传递进去

```scala
def newThread(block :()=>Unit){
  new Thread(){
    override def run(){
      block() ;
    }
  }.start();
}


newThread( () => {
  (1 to 10).foreach(e => {
    val tname = Thread.currentThread.getName();
    println(tname + " : " + e) ;
  })
}) ;


//省略()
def newThread(block: =>Unit){
  new Thread(){
    override def run(){
      block ;
    }
  }.start();
}


newThread{
  (1 to 10).foreach(e => {
    val tname = Thread.currentThread.getName();
    println(tname + " : " + e) ;
  })
};
```



<br/>

### 集合

{% asset_img z3.png %}

{% asset_img z4.png %}

{% asset_img z5.png %}

{% asset_img z6.png %}

<br/>

Nil：

```scala
scala>1::2::Nil				//Nil空集合
scala>var list = List(2,4)
scala>9::list
scala>var list = 1::2::3::4::Nil
```

<br/>

递归计算sum求和。

```scala
scala>def sum(list:List[Int]):Int = {
  if (list == Nil) 0 else list.head + sum(list.tail)
}
```

<br/>

通过模式匹配实现sum求和。

```scala
def sum(list:List[Int]):Int= list match{
  case Nil => 0
  case a::b=> a + sum(b) 
}
```

<br/>

添加删除元素操作符

{% asset_img z7.png %}

{% asset_img z8.png %}

```scala
scala>val set = Set(1,2,3)
scala>set + (1,2,3,5)
scala>set - (1,2,3,5)

scala>val l1 = List(1,2)
scala>val l2 = List(3,4)
scala>l1 ++ l2				//1234 ::			在l1的后面加入l2，
scala>l1 ++: l2				//1234 === :::		在l2的前面加入l1，

scala>val s1 = Set(1,2,3)
scala>val s2 = Set(2,3,4)
scala>s1 | s2				//并集
scala>s1 & s2				//交集
scala>s1 &~ s2			//差集(1,2,3) - (2,3,4) = (1)


// += 操纵的是可变元素,操纵一个元素
// ++= 操纵的是可变集合,操纵集合
// +: 操纵的是不可变集合，产生新集合
scala>import scala.collection.mutable.{Set => SSet}	//别名：因为很多不同包下的类重名

scala>val buf = scala.collection.mutable.ArrayBuffer(1,2,3)
scala>2 +=: buf			//右操作符
scala>buf += 5			//左操作符
```

<br/>

常用方法：

{% asset_img z9.png %}

{% asset_img z10.png %}

<br/>

```scala
scala>buf.take(2)			//提取前2个元素
scala>buf.drop(2)			//删除前2个元素
scala>buf.splitAt(2)			//在指定位置进行切割，形成两个集合。
scala>val b1 = ArrayBuffer(1,2,3)
scala>val b2 = ArrayBuffer(3,4,5,6,)
scala>b1.zip(b2)			//(1,3)(2,4)(3,5)
scala>b1.zipAll(b2,-1,-2)		//(1,3)(2,4)(3,5)(-1,6)	//b1不够补-1，b2不够补-2
scala>b1.zipWithIndex()		//(,0)(,1)(,2)(,3)		//元素和自己的索引形成tuple.


//折叠
scala>List(1,7,2,9).foldLeft(0)(_-_)		//柯里化	意思就是从0开始
//过程	(0 - 1) - 7 - 2 - 9			//-19
scala>List(1,7,2,9).foldRight(0)(_-_)
1  - (7 - (2 - (9 - 0))_				//-13
```