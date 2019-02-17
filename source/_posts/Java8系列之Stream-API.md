---
title: Java8系列之Stream-API
tags:
  - Java8系列
abbrlink: 9b8b8703
date: 2017-12-28 11:40:48
---

### 流(Stream) 到底是什么呢？

- 是数据渠道，用于操作数据源（集合、数组等）所生成的元素序列。
- “集合讲的是数据，流讲的是计算！ ”
- 注意：

① Stream 自己不会存储元素。

② Stream 不会改变源对象。相反，他们会返回一个持有结果的新Stream。

③ Stream 操作是延迟执行的。这意味着他们会等到需要结果的时候才执行。

### Stream 的操作三个步骤

① 创建 Stream

- 一个数据源（如： 集合、数组）， 获取一个流

② 中间操作

- 一个中间操作链，对数据源的数据进行处理

③ 终止操作(终端操作)

- 一个终止操作，执行中间操作链，并产生结果

## 创建Stream

① Collection 提供了两个方法 stream() 与 parallelStream()

② 通过 Arrays 中的 stream() 获取一个数组流

③ 通过 Stream 类中静态方法 of()

④ 创建无限流

⑤ 生成

```java
@Test
public void test1(){
  //1. Collection 提供了两个方法  stream() 与 parallelStream()
  List<String> list = new ArrayList<>();
  Stream<String> stream = list.stream(); //获取一个顺序流
  Stream<String> parallelStream = list.parallelStream(); //获取一个并行流

  //2. 通过 Arrays 中的 stream() 获取一个数组流
  Integer[] nums = new Integer[10];
  Stream<Integer> stream1 = Arrays.stream(nums);

  //3. 通过 Stream 类中静态方法 of()
  Stream<Integer> stream2 = Stream.of(1,2,3,4,5,6);

  //4. 创建无限流
  //迭代
  Stream<Integer> stream3 = Stream.iterate(0, (x) -> x + 2).limit(10);
  stream3.forEach(System.out::println);

  //生成
  Stream<Double> stream4 = Stream.generate(Math::random).limit(2);
  stream4.forEach(System.out::println);

}
```

## 筛选和切片

```java
 /*
 筛选与切片
 filter——接收 Lambda ， 从流中排除某些元素。
 limit——截断流，使其元素不超过给定数量。
 skip(n) —— 跳过元素，返回一个扔掉了前 n 个元素的流。若流中元素不足 n 个，则返回一个空流。与 limit(n) 互补
 distinct——筛选，通过流所生成元素的 hashCode() 和 equals() 去除重复元素
 */

//内部迭代：迭代操作 Stream API 内部完成
@Test
public void test2(){
  //所有的中间操作不会做任何的处理
  Stream<Employee> stream = emps.stream()
    .filter((e) -> {
      System.out.println("测试中间操作");
      return e.getAge() <= 35;
    });

  //只有当做终止操作时，所有的中间操作会一次性的全部执行，称为“惰性求值”
  stream.forEach(System.out::println);
}

//外部迭代
@Test
public void test3(){
  Iterator<Employee> it = emps.iterator();

  while(it.hasNext()){
    System.out.println(it.next());
  }
}

@Test
public void test4(){
  emps.stream()
    .filter((e) -> {
      System.out.println("短路！"); // &&  ||
      return e.getSalary() >= 5000;
    }).limit(3)
    .forEach(System.out::println);
}

@Test
public void test5(){
  emps.parallelStream()
    .filter((e) -> e.getSalary() >= 5000)
    .skip(2)
    .forEach(System.out::println);
}

@Test
public void test6(){
  emps.stream()
    .distinct()
    .forEach(System.out::println);
}
```

## 映射

~~~java
/*
映射
map——接收 Lambda ， 将元素转换成其他形式或提取信息。接收一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素。
flatMap——接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有流连接成一个流。
两者的区别就类似于add和addAll方法的区别。add是把一个集合添加进去，addAll就是把集合中的每个元素添加到另一个集合中。
*/
@Test
public void test1(){
  Stream<String> str = emps.stream()
    .map((e) -> e.getName());

  System.out.println("-------------------------------------------");

  List<String> strList = Arrays.asList("aaa", "bbb", "ccc", "ddd", "eee");

  Stream<String> stream = strList.stream()
    .map(String::toUpperCase);

  stream.forEach(System.out::println);

 //============================================================================================
  
  Stream<Stream<Character>> stream2 = strList.stream()
    .map(TestStreamAPI1::filterCharacter);//{{a,a,a},{b,b,b}}

  stream2.forEach((sm) -> {
    sm.forEach(System.out::println);
  });

  System.out.println("---------------------------------------------");

  Stream<Character> stream3 = strList.stream()
    .flatMap(TestStreamAPI1::filterCharacter);//{a,a,a,b,b,b}

  stream3.forEach(System.out::println);
}

//==============================================================================================
public static Stream<Character> filterCharacter(String str){
  List<Character> list = new ArrayList<>();

  for (Character ch : str.toCharArray()) {
    list.add(ch);
  }

  return list.stream();
}
~~~

## 排序

```java
/*
sorted()——自然排序(Comparable)
sorted(Comparator com)——定制排序(Comparator)
*/
@Test
public void test2(){
  emps.stream()
    .map(Employee::getName)
    .sorted()
    .forEach(System.out::println);

  System.out.println("------------------------------------");

  emps.stream()
    .sorted((x, y) -> {
      if(x.getAge() == y.getAge()){
        return x.getName().compareTo(y.getName());
      }else{
        return Integer.compare(x.getAge(), y.getAge());
      }
    }).forEach(System.out::println);
}
```

## 终止操作

```java
public class TestStreamAPI2 {

  List<Employee> emps = Arrays.asList(
    new Employee(102, "李四", 59, 6666.66, Status.BUSY),
    new Employee(101, "张三", 18, 9999.99, Status.FREE),
    new Employee(103, "王五", 28, 3333.33, Status.VOCATION),
    new Employee(104, "赵六", 8, 7777.77, Status.BUSY),
    new Employee(104, "赵六", 8, 7777.77, Status.FREE),
    new Employee(104, "赵六", 8, 7777.77, Status.FREE),
    new Employee(105, "田七", 38, 5555.55, Status.BUSY)
  );

  //3. 终止操作
  /*
		allMatch——检查是否匹配所有元素
		anyMatch——检查是否至少匹配一个元素
		noneMatch——检查是否没有匹配的元素
		findFirst——返回第一个元素
		findAny——返回当前流中的任意元素
		count——返回流中元素的总个数
		max——返回流中最大值
		min——返回流中最小值
	 */
  @Test
  public void test1(){
    boolean bl = emps.stream()
      .allMatch((e) -> e.getStatus().equals(Status.BUSY));

    System.out.println(bl);

    boolean bl1 = emps.stream()
      .anyMatch((e) -> e.getStatus().equals(Status.BUSY));

    System.out.println(bl1);

    boolean bl2 = emps.stream()
      .noneMatch((e) -> e.getStatus().equals(Status.BUSY));

    System.out.println(bl2);
  }

  @Test
  public void test2(){
    Optional<Employee> op = emps.stream()
      .sorted((e1, e2) -> Double.compare(e1.getSalary(), e2.getSalary()))
      .findFirst();

    System.out.println(op.get());

    System.out.println("--------------------------------");

    Optional<Employee> op2 = emps.parallelStream()
      .filter((e) -> e.getStatus().equals(Status.FREE))
      .findAny();

    System.out.println(op2.get());
  }

  @Test
  public void test3(){
    long count = emps.stream()
      .filter((e) -> e.getStatus().equals(Status.FREE))
      .count();

    System.out.println(count);

    Optional<Double> op = emps.stream()
      .map(Employee::getSalary)
      .max(Double::compare);

    System.out.println(op.get());

    Optional<Employee> op2 = emps.stream()
      .min((e1, e2) -> Double.compare(e1.getSalary(), e2.getSalary()));

    System.out.println(op2.get());
  }

  //注意：流进行了终止操作后，不能再次使用
  @Test
  public void test4(){
    Stream<Employee> stream = emps.stream()
      .filter((e) -> e.getStatus().equals(Status.FREE));

    long count = stream.count();

    stream.map(Employee::getSalary)
      .max(Double::compare);
  }
}
```

## 归约

```java
/*
归约
reduce(T identity, BinaryOperator) / reduce(BinaryOperator) ——可以将流中元素反复结合起来，得到一个值。
	 */
@Test
public void test1(){
  List<Integer> list = Arrays.asList(1,2,3,4,5,6,7,8,9,10);

  Integer sum = list.stream()
    .reduce(0, (x, y) -> x + y);

  System.out.println(sum);

  System.out.println("----------------------------------------");

  Optional<Double> op = emps.stream()
    .map(Employee::getSalary)
    .reduce(Double::sum);

  System.out.println(op.get());
}
```

## 收集

```java
//collect——将流转换为其他形式。接收一个 Collector接口的实现，用于给Stream中元素做汇总的方法
@Test
public void test3(){
  List<String> list = emps.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());

  list.forEach(System.out::println);

  System.out.println("----------------------------------");

  Set<String> set = emps.stream()
    .map(Employee::getName)
    .collect(Collectors.toSet());

  set.forEach(System.out::println);

  System.out.println("----------------------------------");

  HashSet<String> hs = emps.stream()
    .map(Employee::getName)
    .collect(Collectors.toCollection(HashSet::new));

  hs.forEach(System.out::println);
}

@Test
public void test4(){
  Optional<Double> max = emps.stream()
    .map(Employee::getSalary)
    .collect(Collectors.maxBy(Double::compare));

  System.out.println(max.get());

  Optional<Employee> op = emps.stream()
    .collect(Collectors.minBy((e1, e2) -> Double.compare(e1.getSalary(), e2.getSalary())));

  System.out.println(op.get());

  Double sum = emps.stream()
    .collect(Collectors.summingDouble(Employee::getSalary));

  System.out.println(sum);

  Double avg = emps.stream()
    .collect(Collectors.averagingDouble(Employee::getSalary));

  System.out.println(avg);

  Long count = emps.stream()
    .collect(Collectors.counting());

  System.out.println(count);

  System.out.println("--------------------------------------------");

  DoubleSummaryStatistics dss = emps.stream()
    .collect(Collectors.summarizingDouble(Employee::getSalary));

  System.out.println(dss.getMax());
}

//分组
@Test
public void test5(){
  Map<Status, List<Employee>> map = emps.stream()
    .collect(Collectors.groupingBy(Employee::getStatus));

  System.out.println(map);
}

//多级分组
@Test
public void test6(){
  Map<Status, Map<String, List<Employee>>> map = emps.stream()
    .collect(Collectors.groupingBy(Employee::getStatus, Collectors.groupingBy((e) -> {
      if(e.getAge() >= 60)
        return "老年";
      else if(e.getAge() >= 35)
        return "中年";
      else
        return "成年";
    })));

  System.out.println(map);
}

//分区
@Test
public void test7(){
  Map<Boolean, List<Employee>> map = emps.stream()
    .collect(Collectors.partitioningBy((e) -> e.getSalary() >= 5000));

  System.out.println(map);
}

//
@Test
public void test8(){
  String str = emps.stream()
    .map(Employee::getName)
    .collect(Collectors.joining("," , "----", "----"));

  System.out.println(str);
}

@Test
public void test9(){
  Optional<Double> sum = emps.stream()
    .map(Employee::getSalary)
    .collect(Collectors.reducing(Double::sum));

  System.out.println(sum.get());
}
```

## 练习题目

```java
public class TestTransaction {

  List<Transaction> transactions = null;

  @Before
  public void before(){
    Trader raoul = new Trader("Raoul", "Cambridge");
    Trader mario = new Trader("Mario", "Milan");
    Trader alan = new Trader("Alan", "Cambridge");
    Trader brian = new Trader("Brian", "Cambridge");

    transactions = Arrays.asList(
      new Transaction(brian, 2011, 300),
      new Transaction(raoul, 2012, 1000),
      new Transaction(raoul, 2011, 400),
      new Transaction(mario, 2012, 710),
      new Transaction(mario, 2012, 700),
      new Transaction(alan, 2012, 950)
    );
  }

  //1. 找出2011年发生的所有交易， 并按交易额排序（从低到高）
  @Test
  public void test1(){
    transactions.stream()
      .filter((t) -> t.getYear() == 2011)
      .sorted((t1, t2) -> Integer.compare(t1.getValue(), t2.getValue()))
      .forEach(System.out::println);
  }

  //2. 交易员都在哪些不同的城市工作过？
  @Test
  public void test2(){
    transactions.stream()
      .map((t) -> t.getTrader().getCity())
      .distinct()
      .forEach(System.out::println);
  }

  //3. 查找所有来自剑桥的交易员，并按姓名排序
  @Test
  public void test3(){
    transactions.stream()
      .filter((t) -> t.getTrader().getCity().equals("Cambridge"))
      .map(Transaction::getTrader)
      .sorted((t1, t2) -> t1.getName().compareTo(t2.getName()))
      .distinct()
      .forEach(System.out::println);
  }

  //4. 返回所有交易员的姓名字符串，按字母顺序排序
  @Test
  public void test4(){
    transactions.stream()
      .map((t) -> t.getTrader().getName())
      .sorted()
      .forEach(System.out::println);

    System.out.println("-----------------------------------");

    String str = transactions.stream()
      .map((t) -> t.getTrader().getName())
      .sorted()
      .reduce("", String::concat);

    System.out.println(str);

    System.out.println("------------------------------------");

    transactions.stream()
      .map((t) -> t.getTrader().getName())
      .flatMap(TestTransaction::filterCharacter)
      .sorted((s1, s2) -> s1.compareToIgnoreCase(s2))
      .forEach(System.out::print);
  }

  public static Stream<String> filterCharacter(String str){
    List<String> list = new ArrayList<>();

    for (Character ch : str.toCharArray()) {
      list.add(ch.toString());
    }

    return list.stream();
  }

  //5. 有没有交易员是在米兰工作的？
  @Test
  public void test5(){
    boolean bl = transactions.stream()
      .anyMatch((t) -> t.getTrader().getCity().equals("Milan"));

    System.out.println(bl);
  }


  //6. 打印生活在剑桥的交易员的所有交易额
  @Test
  public void test6(){
    Optional<Integer> sum = transactions.stream()
      .filter((e) -> e.getTrader().getCity().equals("Cambridge"))
      .map(Transaction::getValue)
      .reduce(Integer::sum);

    System.out.println(sum.get());
  }


  //7. 所有交易中，最高的交易额是多少
  @Test
  public void test7(){
    Optional<Integer> max = transactions.stream()
      .map((t) -> t.getValue())
      .max(Integer::compare);

    System.out.println(max.get());
  }

  //8. 找到交易额最小的交易
  @Test
  public void test8(){
    Optional<Transaction> op = transactions.stream()
      .min((t1, t2) -> Integer.compare(t1.getValue(), t2.getValue()));

    System.out.println(op.get());
  }

}
```