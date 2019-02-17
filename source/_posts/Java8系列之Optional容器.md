---
title: Java8系列之Optional容器
tags:
  - Java8系列
abbrlink: 91ca9518
date: 2018-01-03 10:43:32
---



### 一、Optional 容器类：用于尽量避免空指针异常

①. Optional.of(T t) : 创建一个 Optional 实例

~~~java
@Test
public void test1(){
  Optional<Employee> op = Optional.of(new Employee());
  Employee emp = op.get();
  System.out.println(emp);
}
~~~

<br/>

②. Optional.empty() : 创建一个空的 Optional 实例

③.Optional.ofNullable(T t):若 t 不为 null,创建 Optional 实例,否则创建空实例

~~~java
@Test
public void test2(){
  	Optional<Employee> op = Optional.ofNullable(null);
	System.out.println(op.get());

//	Optional<Employee> op = Optional.empty();
//	System.out.println(op.get());
}
~~~

<br/>

④.isPresent() : 判断是否包含值

⑤.orElse(T t) :  如果调用对象包含值，返回该值，否则返回t

⑥.orElseGet(Supplier s) :如果调用对象包含值，返回该值，否则返回 s 获取的值

~~~java
@Test
public void test3(){
  Optional<Employee> op = Optional.ofNullable(new Employee());

  if(op.isPresent()){
    System.out.println(op.get());
  }

  Employee emp = op.orElse(new Employee("张三"));
  System.out.println(emp);

  Employee emp2 = op.orElseGet(() -> new Employee());
  System.out.println(emp2);
}
~~~

<br/>
⑦.map(Function f): 如果有值对其处理，并返回处理后的Optional，否则返回 Optional.empty()

⑧.flatMap(Function mapper):与 map 类似，要求返回值必须是Optional

~~~java
@Test
public void test4(){
  Optional<Employee> op = Optional.of(new Employee(101, "张三", 18, 9999.99));

  Optional<String> op2 = op.map(Employee::getName);
  System.out.println(op2.get());

  Optional<String> op3 = op.flatMap((e) -> Optional.of(e.getName()));
  System.out.println(op3.get());
}
~~~

<br/>

### 二、新需求：获取一个男人心中女神的名字

1. 以前的写法

~~~java
public String getGodnessName(Man man){
  if(man != null){
    Godness g = man.getGod();

    if(g != null){
      return g.getName();
    }
  }
  return "苍老师";
}

@Test
public void test5(){
  Man man = new Man();

  String name = getGodnessName(man);
  System.out.println(name);
}
~~~

<br/>

1. 现在的写法

~~~java
public String getGodnessName2(Optional<NewMan> man){
  return man.orElse(new NewMan())
    .getGodness()
    .orElse(new Godness("苍老师")) //如果godness没有初始化，那么这里会报空指针异常
    .getName();
}

//运用 Optional 的实体类
@Test
public void test6(){
  Optional<Godness> godness = Optional.ofNullable(new Godness("林志玲"));

  Optional<NewMan> op = Optional.ofNullable(new NewMan(godness));
  String name = getGodnessName2(op);
  System.out.println(name);
}
~~~

<br/>

#### 注意一个非常容易忽略的点：

~~~java
public class NewMan {
  public NewMan() {
  }

  public Optional<Godness> godness = Optional.ofNullable(null);//一定要记得初始化，否则会空指针

  public NewMan(Optional<Godness> godness) {
    this.godness = godness;
  }

  public Optional<Godness> getGodness() {
    return godness;
  }

  public void setGodness(Optional<Godness> godness) {
    this.godness = godness;
  }
}
~~~

其中 **godness一定要记得初始化**，否则在getGodnessName2方法中会一直出现空指针异常。