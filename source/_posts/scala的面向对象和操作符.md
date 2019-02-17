---
title: scala的面向对象和操作符
categories:
  - scala
tags:
  - 快学scala读书笔记
abbrlink: ccad62d6
date: 2018-03-12 16:30:22
---

### OOP 面向对象

```scala
scala>class Person{
  //定义变量,私有类型,必须初始化
  //set/get也私有
  private var id = 0 ;

  //只有get方法，没有set方法	val
  val age = 100 ;

  //生成私有属性，和公有的get/set方法。 get对应 name，  set对应 name_=
  var name = "tom" ;
  //默认public
  def incre(a:Int) = {id += a ;}
  //如果定义时，没有(),调用就不能加()
  def current() = id 
}
```

<br/>

```scala
scala>var p = new Person();
scala>p.current()
scala>p.current
scala>p.incr(100)
scala>p.name
scala>p.name_=("kkkk")					//一定要注意加  ()   试了无数遍的结果！！！
scala>p.name = "kkkk"
```

<br/>

### private[this]

作用,**控制成员只能在自己的对象中访问**

```scala
class Counter{
  private[this] var value =  0 ;
  def incre(n:Int){value += n}

  def isLess(other:Counter) = value < other.value ;	//错误：private[this]不允许其他对象访问
}
```

<br/>

### 构造函数

主构造器

辅助构造

```scala
class Person{
  var id = 1 ;
  var name = "tom" ;
  var age = 12 ;

  //辅助构造
  def this(name:String){
    this();
    this.name = name ;
  }
  //辅助构造
  def this(name:String,age:Int){
    //调用辅助构造
    this(name) ;
    this.age = age ;
  }
}
```

<br/>

```scala
//主构造   ----------> scala把主构造和类整合在一起了。
//val ===> 只读
//var ==> get/set ( 自动生成 )
//none ==> none
class Person(val name:String,var age:Int , id :Int){
  //也就是说这样的方式可以省略需要定义的属性和其get/set
  def hello() = println(id)
}
```



<br/>

### Object

说明：scala没有静态的概念，如果需要定义静态成员，可以通过object实现。

编译完成后，会生成对应的类， 会生成两个类，一个工具类，一个单例类

方法都是静态方法，非静态成员对应到单例类中

单例类以Util$作为类名称。

```scala
scala>object Util{
  
  //单例类中.(Util$)
  private var brand = "benz" ;
  
  //静态方法.
  def hello() = println("hello world");
}
```

<br/>

工具类：

{% asset_img z1.png %}

单例类：

{% asset_img z2.png %}

<br/>

apply方法：减少代码。

```scala
object Util{
	def apply(s:String) = println(s) ;
}

Util("hello world");
Util.apply("hello world");
```

<br/>



### 伴生对象(companions object)

类名和object名称相同，而且**必须在一个scala文件中定义**。

```scala
class Car{
  def stop() = println("stop....")
}

object Car{
  def run() = println("run...")
}
```

<br/>

### 抽象类

```scala
$scala>abstract class Animal(val name:String){
  //抽象字段，没有初始化。
  val id:Int  ;
  //抽象方法，没有方法体，不需要抽象关键字修饰。
  def run() ;
}
```

<br/>

### 继承和重写

```scala
class Dog extends Animal{
  //重写,覆盖
  //overload
  override def run()={...}
}
```

<br/>

### 导入其他包：

```scala
import java.io.Exception
import java.io.{A,B,C}			//
import java.io.{A => A0}		//别名
```

<br/>

### 类型检查和转换

```scala
$scala> class Animal{}
$scala> class Dog extends Animal{}
$scala> val d = new Dog();
$scala> d.isInstanceOf[Animal]			//true,===> instanceOf
$scala>val a = d.asInstanceOf[Animal]		//强转,===> (Animal)d


//得带对象的类
$scala>d.getClass						//d.getClass();
$scala>d.getClass == classOf[Dog]		//精确匹配

$scala>class Animal(val name:String){}
$scala>class Dog(name:String,val age:Int) extends Animal(name){}
```

<br/>



### scala类型树

{% asset_img z3.png %}

Any

/|\

 |------------AnyVal ：<|------Int|Boolean|...|Unit

 |------------AnyRef ：<|------class ...

<br/>

### 文件操作

```scala
import scala.io.Source ;

object FileDemo {
  def main(args: Array[String]): Unit = {
    val s = Source.fromFile("d:/hello.txt") ;
    val lines = s.getLines();
    for(line <- lines){
      println(line)
    }
  }
}
```

```scala
scala.io.Source.fromFile(...).mkString()
```

```scala
//通过正则
val str = Source.fromFile("d:/hello.txt").mkString
val it = str.split("\\s+")
for(i <- it){
  println(i)
}
```

<br/>

### trait

特质，等价于java中的接口.

如果只有一个trait使用extends进行扩展，如果多个，使用with对剩余的trait进行扩展。

```scala
trait logger1{
  def log1() = println("hello log1");
}

trait logger2{
  def log2() = println("hello log2");
}

class Dog extends logger1 with logger2{	
}
```

<br/>

trait之间也存在扩展。

```scala
trait logger1{
  def log1() = println("hello log1");
}
trait logger2 {
}
trait logger3 extends logger2 with logger1{
}
```

<br/>

with trait是需要对每个trait都是用with

```scala
class xxx extends A with T1 with T2 with ...{
  ...
}
```

<br/>

### 自身类型：

只能让Dog的子类继承，不允许其他类继承

注意下面这些代码必须写在同一个文件中或者一起复制才可以。

```scala
trait logger{
  this:Dog =>
  def run() = println("run....")
}
class Dog {	
}
class Jing8 extends Dog with logger{	//所以如果这里没有 extends Dog 就会出错
}

class Cat extends logger{	//错误
}
```

<br/>

### 操作符

```scala
//中置操作符（操作符在中间，两个参数）
scala> 1 + 2			//
scala> 1.+(2)			//

//单元操作符（一元操作符）
scala> 1 toString		//+:        -:取反	       !:boolean取反         ~:按位取反

//赋值操作符
$scala>+=            -=          *=         /=
```

<br/>

### 结合性

在scala中，所有操作符都是左结合的，除了

- 以冒号（：）结尾的操作符
- 赋值操作符

尤其值得一提的是，用于构造列表的::操作符是右结合的，例如：

1 :: 2 :: Nil

的意思是

1 :: (2 :: Nil)

本就应该这样，我们首先要创建出包含2的列表，这个列表又被作为尾巴拼到以1作为头部的列表当中。

<br/>

2  :: Nil 

的意思是：

Nil.::(2)

<br/>

### unapply()

是apply的逆向过程

```scala
//定义类
class Fraction(val n:Int,val d:Int){
}

object Fraction{
  //正向的apply过程
  def apply(n : Int,d:Int)= new Fraction(n,d) 
  //逆向过程
  def unapply(f:Fraction) = Some(f.n,f.d)
}

scala>val f = Fraction(1,2)			//apply(...)
scala>val Fraction(a,b) = f			//unapply(...)
```