---
title: Java8系列之Lambda表达式
tags:
  - Java8系列
abbrlink: ccf34b81
date: 2017-12-24 11:27:22
---

## 为什么使用Lambda表达式：

Lambda 是一个匿名函数，我们可以把 Lambda表达式理解为是一段可以传递的代码（将代码像数据一样进行传递）。可以写出更简洁、更灵活的代码。作为一种更紧凑的代码风格，使Java的语言表达能力得到了提升。

<br/>

## 一：代码对比

#### 1. 字符串排序 

~~~java
//原来的匿名内部类
@Test
public void test1(){
  //方法一
  Comparator<String> com = new Comparator<String>(){
    @Override
    public int compare(String o1, String o2) {
      return Integer.compare(o1.length(), o2.length());
    }
  };
  TreeSet<String> ts = new TreeSet<>(com);
  
  //方法二
  TreeSet<String> ts2 = new TreeSet<>(new Comparator<String>(){
    @Override
    public int compare(String o1, String o2) {
      return Integer.compare(o1.length(), o2.length());
    }
  });
}
~~~

~~~java
//现在的 Lambda 表达式
@Test
public void test2(){
  Comparator<String> com = (x, y) -> Integer.compare(x.length(), y.length());
  TreeSet<String> ts = new TreeSet<>(com);
}
~~~

#### 2. 假设有数据如下

~~~java
List<Employee> emps = Arrays.asList(
  new Employee(101, "张三", 18, 9999.99),
  new Employee(102, "李四", 59, 6666.66),
  new Employee(103, "王五", 28, 3333.33),
  new Employee(104, "赵六", 8, 7777.77),
  new Employee(105, "田七", 38, 5555.55)
);
~~~

#### 需求：获取公司中年龄小于 35 的员工信息

普通方法：

~~~java
public List<Employee> filterEmployeeAge(List<Employee> emps){
  List<Employee> list = new ArrayList<>();

  for (Employee emp : emps) {
    if(emp.getAge() <= 35){
    	list.add(emp);
    }
  }
  return list;
}

@Test
public void test3(){
  List<Employee> list = filterEmployeeAge(emps);

  for (Employee employee : list) {
    System.out.println(employee);
  }
}
~~~

#### 新需求：获取公司中工资大于 5000 的员工信息

普通方法：

~~~java
public List<Employee> filterEmployeeSalary(List<Employee> emps){
  List<Employee> list = new ArrayList<>();

  for (Employee emp : emps) {
    if(emp.getSalary() >= 5000){
      list.add(emp);
    }
  }
  return list;
}
~~~

优化方式一：**策略设计模式**

~~~java
public interface MyPredicate<T> {
	public boolean test(T t);
}
~~~

~~~java
public class FilterEmployeeForAge implements MyPredicate<Employee>{
	@Override
	public boolean test(Employee t) {
		return t.getAge() <= 35;
	}
}
~~~

~~~java
public List<Employee> filterEmployee(List<Employee> emps, MyPredicate<Employee> mp){
  List<Employee> list = new ArrayList<>();

  for (Employee employee : emps) {
    if(mp.test(employee)){
    	list.add(employee);
    }
  }
  return list;
}

@Test
public void test4(){
  List<Employee> list = filterEmployee(emps, new FilterEmployeeForAge());
  for (Employee employee : list) {
    System.out.println(employee);
  }

  System.out.println("------------------------------------------");

  List<Employee> list2 = filterEmployee(emps, new FilterEmployeeForSalary());
  for (Employee employee : list2) {
    System.out.println(employee);
  }
}
~~~

优化方式二：**匿名内部类**

~~~java
@Test
public void test5(){
  List<Employee> list = filterEmployee(emps, new MyPredicate<Employee>() {
    @Override
    public boolean test(Employee t) {
      return t.getId() <= 103;
    }
  });

  for (Employee employee : list) {
    System.out.println(employee);
  }
}
~~~

优化方式三：**Lambda 表达式**

~~~java
@Test
public void test6(){
  List<Employee> list = filterEmployee(emps, (e) -> e.getAge() <= 35);
  list.forEach(System.out::println);

  System.out.println("------------------------------------------");

  List<Employee> list2 = filterEmployee(emps, (e) -> e.getSalary() >= 5000);
  list2.forEach(System.out::println);
}
~~~

优化方式四：**Stream API**

~~~java
@Test
public void test7(){
  emps.stream()
    .filter((e) -> e.getAge() <= 35)
    .forEach(System.out::println);

  System.out.println("----------------------------------------------");

  emps.stream()
    .map(Employee::getName)
    .limit(3)
    .sorted()
    .forEach(System.out::println);
}
~~~

<br/>

### 二：基础语法：

一：Lambda 表达式的基础语法：Java8中引入了一个新的操作符 "->" 该操作符称为箭头操作符或 Lambda 操作符， 箭头操作符将 Lambda 表达式拆分成两部分：

- 左侧：Lambda 表达式的参数列表
- 右侧：Lambda 表达式中所需执行的功能， 即 Lambda 体

<br/>

① 语法格式一：无参数，无返回值

- () -> System.out.println("Hello Lambda!");

② 语法格式二：有一个参数，并且无返回值

- (x) -> System.out.println(x)

③ 语法格式三：若只有一个参数，小括号可以省略不写

- x -> System.out.println(x)

④ 语法格式四：有两个以上的参数，有返回值，并且 Lambda 体中有多条语句

~~~java
Comparator<Integer> com = (x, y) -> {
  System.out.println("函数式接口");
  return Integer.compare(x, y);
};
~~~

⑤ 语法格式五：若 Lambda 体中只有一条语句， return 和 大括号都可以省略不写

- Comparator<Integer> com = (x, y) -> Integer.compare(x, y);

⑥ 语法格式六：Lambda 表达式的参数列表的数据类型可以省略不写，因为JVM编译器通过上下文推断出，数据类型，即“类型推断”

- (Integer x, Integer y) -> Integer.compare(x, y);

<br/>

二： Lambda 表达式需要“函数式接口”的支持

- 函数式接口：接口中只有一个抽象方法的接口，称为函数式接口。 可以使用注解 @FunctionalInterface 修饰，可以检查是否是函数式接口

~~~java
@Test
public void test1(){
  int num = 0;//jdk 1.7 前，必须是 final

  Runnable r = new Runnable() {
    @Override
    public void run() {
      System.out.println("Hello World!" + num);
    }
  };

  r.run();

  System.out.println("-------------------------------");

  Runnable r1 = () -> System.out.println("Hello Lambda!");
  r1.run();
}
~~~

<br/>

### 三：Java内置四大核心函数接口

① Consumer<T> : 消费型接口

- void accept(T t);

~~~java
//Consumer<T> 消费型接口 :
@Test
public void test1(){
  happy(10000, (m) -> System.out.println("大宝剑，每次消费：" + m + "元"));
} 

public void happy(double money, Consumer<Double> con){
  con.accept(money);
}
~~~

<br/>

② Supplier<T> : 供给型接口

- T get();

~~~java
//Supplier<T> 供给型接口 :
@Test
public void test2(){
  List<Integer> numList = getNumList(10, () -> (int)(Math.random() * 100));

  for (Integer num : numList) {
    System.out.println(num);
  }
}

//需求：产生指定个数的整数，并放入集合中
public List<Integer> getNumList(int num, Supplier<Integer> sup){
  List<Integer> list = new ArrayList<>();

  for (int i = 0; i < num; i++) {
    Integer n = sup.get();
    list.add(n);
  }

  return list;
}
~~~

<br/>

③ Function<T, R> : 函数型接口

- R apply(T t);

~~~java
//Function<T, R> 函数型接口：
@Test
public void test3(){
  String newStr = strHandler("\t\t\t ng威武   ", (str) -> str.trim());
  System.out.println(newStr);

  String subStr = strHandler("ng威武霸气", (str) -> str.substring(2, 5));
  System.out.println(subStr);
}

//需求：用于处理字符串
public String strHandler(String str, Function<String, String> fun){
  return fun.apply(str);
}
~~~

<br/>

④ Predicate<T> : 断言型接口

- boolean test(T t);

~~~java
//Predicate<T> 断言型接口：
@Test
public void test4(){
  List<String> list = Arrays.asList("Hello", "atguigu", "Lambda", "www", "ok");
  List<String> strList = filterStr(list, (s) -> s.length() > 3);

  for (String str : strList) {
    System.out.println(str);
  }
}

//需求：将满足条件的字符串，放入集合中
public List<String> filterStr(List<String> list, Predicate<String> pre){
  List<String> strList = new ArrayList<>();

  for (String str : list) {
    if(pre.test(str)){
      strList.add(str);
    }
  }

  return strList;
}
~~~

<br/>

#### 总结：

- Consumer<T> : 消费型接口  ：  只消费，没有返回，所以只接受参数，没有返回值
- Supplier<T>    : 供给型接口  ：  只提供，没有消费，所以不接受参数，只有返回值
- Function<T, R> : 函数型接口：  提供一个函数，把T类型，转换成R类型，一般用于数据转换
- Predicate<T> : 断言型接口   ：  提供一个判断的函数，返回一个boolean值，一般用于过滤

<br/>

### 四：方法引用和构造器引用

一、方法引用：若 Lambda 体中的功能，已经有方法提供了实现，可以使用方法引用

<br/>

- （可以将方法引用理解为 Lambda 表达式的另外一种表现形式）
- 注意：
- ①方法引用所引用的方法的参数列表与返回值类型，需要与函数式接口中抽象方法的参数列表和返回值类型保持一致！
- ②若Lambda 的参数列表的第一个参数，是实例方法的调用者，第二个参数(或无参)是实例方法的参数时，格式： ClassName::MethodName

<br/>
① 对象的引用 :: 实例方法名

~~~~java
//对象的引用 :: 实例方法名
@Test
public void test2(){
  Employee emp = new Employee(101, "张三", 18, 9999.99);

  Supplier<String> sup = () -> emp.getName();
  System.out.println(sup.get());

  System.out.println("----------------------------------");

  Supplier<String> sup2 = emp::getName;
  System.out.println(sup2.get());
}

@Test
public void test1(){
  PrintStream ps = System.out;
  Consumer<String> con = (str) -> ps.println(str);
  con.accept("Hello World！");

  System.out.println("--------------------------------");

  Consumer<String> con2 = ps::println;
  con2.accept("Hello Java8！");

  Consumer<String> con3 = System.out::println;
}
~~~~

<br/>

② 类名 :: 静态方法名

~~~java
//类名 :: 静态方法名
@Test
public void test4(){
  Comparator<Integer> com = (x, y) -> Integer.compare(x, y);

  System.out.println("-------------------------------------");

  Comparator<Integer> com2 = Integer::compare;
}

@Test
public void test3(){
  BiFunction<Double, Double, Double> fun = (x, y) -> Math.max(x, y);
  System.out.println(fun.apply(1.5, 22.2));

  System.out.println("--------------------------------------------------");

  BiFunction<Double, Double, Double> fun2 = Math::max;
  System.out.println(fun2.apply(1.2, 1.5));
}
~~~

<br/>

③ 类名 :: 实例方法名

~~~java
//类名 :: 实例方法名
@Test
public void test5(){
  BiPredicate<String, String> bp = (x, y) -> x.equals(y);
  System.out.println(bp.test("abcde", "abcde"));

  System.out.println("-----------------------------------------");

  BiPredicate<String, String> bp2 = String::equals;
  System.out.println(bp2.test("abc", "abc"));

  System.out.println("-----------------------------------------");


  Function<Employee, String> fun = (e) -> e.show();
  System.out.println(fun.apply(new Employee()));

  System.out.println("-----------------------------------------");

  Function<Employee, String> fun2 = Employee::show;
  System.out.println(fun2.apply(new Employee()));

}
~~~

<br/>

二、构造器引用 :构造器的参数列表，需要与函数式接口中参数列表保持一致！

 ① 类名 :: new

- 注意
- 需要调用的构造器的参数列表要与函数式接口中抽象方法的参数列表保持一致

```java
//构造器引用
@Test
public void test7(){
  Function<String, Employee> fun = Employee::new;

  BiFunction<String, Integer, Employee> fun2 = Employee::new;
}

@Test
public void test6(){
  Supplier<Employee> sup = () -> new Employee();
  System.out.println(sup.get());

  System.out.println("------------------------------------");

  Supplier<Employee> sup2 = Employee::new;
  System.out.println(sup2.get());
}
```

<br/>

三、数组引用

① 类型[] :: new;

~~~java
//数组引用
@Test
public void test8(){
  Function<Integer, String[]> fun = (args) -> new String[args];
  String[] strs = fun.apply(10);
  System.out.println(strs.length);

  System.out.println("--------------------------");

  Function<Integer, Employee[]> fun2 = Employee[] :: new;
  Employee[] emps = fun2.apply(20);
  System.out.println(emps.length);
}
~~~

<br/>

### 五：常用函数式接口

{% asset_img Java8中的函数式接口.png %}