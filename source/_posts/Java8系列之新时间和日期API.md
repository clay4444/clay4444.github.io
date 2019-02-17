---
title: Java8系列之新时间和日期API
tags:
  - Java8系列
abbrlink: 6d17d8e0
date: 2018-01-06 14:46:46
---



### 传统的日期时间API会有线程安全问题，

```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMdd");

Callable<Date> task = new Callable<Date>() {

  @Override
  public Date call() throws Exception {
    return sdf.parse("20161121");
  }

};

ExecutorService pool = Executors.newFixedThreadPool(10);

List<Future<Date>> results = new ArrayList<>();

for (int i = 0; i < 10; i++) {
  results.add(pool.submit(task));
}

for (Future<Date> future : results) {
  System.out.println(future.get());
}

pool.shutdown();
```

<br/>

### 解决方案：使用ThreadLocal

~~~java
//解决多线程安全问题
Callable<Date> task = new Callable<Date>() {

  @Override
  public Date call() throws Exception {
    return DateFormatThreadLocal.convert("20161121");
  }

};

ExecutorService pool = Executors.newFixedThreadPool(10);

List<Future<Date>> results = new ArrayList<>();

for (int i = 0; i < 10; i++) {
  results.add(pool.submit(task));
}

for (Future<Date> future : results) {
  System.out.println(future.get());
}

pool.shutdown();
~~~

<br/>

## Java8 新增的日期、时间API

<br/>

### 1. LocalDate、LocalTime、LocalDateTime

~~~java
@Test
public void test1(){
  LocalDateTime ldt = LocalDateTime.now();
  System.out.println(ldt);

  LocalDateTime ld2 = LocalDateTime.of(2016, 11, 21, 10, 10, 10);
  System.out.println(ld2);

  LocalDateTime ldt3 = ld2.plusYears(20);
  System.out.println(ldt3);

  LocalDateTime ldt4 = ld2.minusMonths(2);
  System.out.println(ldt4);

  System.out.println(ldt.getYear());
  System.out.println(ldt.getMonthValue());
  System.out.println(ldt.getDayOfMonth());
  System.out.println(ldt.getHour());
  System.out.println(ldt.getMinute());
  System.out.println(ldt.getSecond());
}
~~~

<br/>

### 2. Instant : 时间戳。 （使用 Unix 元年  1970年1月1日 00:00:00 所经历的毫秒值）

```java
@Test
public void test2(){
  Instant ins = Instant.now();  //默认使用 UTC 时区
  System.out.println(ins);

  OffsetDateTime odt = ins.atOffset(ZoneOffset.ofHours(8));
  System.out.println(odt);

  System.out.println(ins.getNano());

  Instant ins2 = Instant.ofEpochSecond(5);
  System.out.println(ins2);
}
```

<br/>

### 3. Duration : 用于计算两个“时间”间隔。Period : 用于计算两个“日期”间隔

~~~java
@Test
public void test3(){
  Instant ins1 = Instant.now();

  System.out.println("--------------------");
  try {
    Thread.sleep(1000);
  } catch (InterruptedException e) {
  }

  Instant ins2 = Instant.now();

  System.out.println("所耗费时间为：" + Duration.between(ins1, ins2).toMillis);

  System.out.println("----------------------------------");

  LocalDate ld1 = LocalDate.now();
  LocalDate ld2 = LocalDate.of(2011, 1, 1);

  Period pe = Period.between(ld2, ld1);
  System.out.println(pe.getYears());
  System.out.println(pe.getMonths());
  System.out.println(pe.getDays());
}
~~~

<br/>

### 4. TemporalAdjuster : 时间校正器

~~~java
@Test
public void test4(){
  LocalDateTime ldt = LocalDateTime.now();
  System.out.println(ldt);

  LocalDateTime ldt2 = ldt.withDayOfMonth(10);	//把月份时间的某一天设置为10号
  System.out.println(ldt2);

  LocalDateTime ldt3 = ldt.with(TemporalAdjusters.next(DayOfWeek.SUNDAY));
  System.out.println(ldt3);

  //自定义：下一个工作日
  LocalDateTime ldt5 = ldt.with((l) -> {
    LocalDateTime ldt4 = (LocalDateTime) l;

    DayOfWeek dow = ldt4.getDayOfWeek();

    if(dow.equals(DayOfWeek.FRIDAY)){
      return ldt4.plusDays(3);
    }else if(dow.equals(DayOfWeek.SATURDAY)){
      return ldt4.plusDays(2);
    }else{
      return ldt4.plusDays(1);
    }
  });

  System.out.println(ldt5);
}
~~~

<br/>

### 5. DateTimeFormatter : 解析和格式化日期或时间

~~~java
@Test
public void test5(){
  //DateTimeFormatter dtf = DateTimeFormatter.ISO_LOCAL_DATE;

  DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy年MM月dd日 HH:mm:ss E");

  LocalDateTime ldt = LocalDateTime.now();
  String strDate = ldt.format(dtf);

  System.out.println(strDate);

  LocalDateTime newLdt = ldt.parse(strDate, dtf);
  System.out.println(newLdt);
}
~~~

<br/>

### 6.ZonedDate、ZonedTime、ZonedDateTime ： 带时区的时间或日期

~~~java
@Test
public void test7(){
  LocalDateTime ldt = LocalDateTime.now(ZoneId.of("Asia/Shanghai"));
  System.out.println(ldt);

  ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("US/Pacific"));
  System.out.println(zdt);
}

@Test
public void test6(){
  Set<String> set = ZoneId.getAvailableZoneIds();
  set.forEach(System.out::println);
}
~~~