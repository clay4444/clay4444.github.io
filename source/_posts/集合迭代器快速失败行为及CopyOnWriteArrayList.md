---
title: 集合迭代器快速失败行为及CopyOnWriteArrayList
categories:
  - 多线程编程
tags:
  - 源码解析系列
abbrlink: 5d37d5dd
date: 2017-07-16 11:04:11
---

## 什么是集合迭代器快速失败行为

- 以ArrayList为例，在多线程并发情况下，如果有一个线程在修改ArrayList集合的结构（插入、移除...），而另一个线程正在用迭代器遍历读取集合中的元素，此时将抛出ConcurrentModificationException异常立即停止迭代遍历操作，而不需要等到遍历结束后再检查有没有出现问题；

## ArrayList.Itr迭代器快速失败源码及例子

- 查看ArrayList的Itr迭代器源码，可以看到Itr为ArrayList的私有内部类，有一个expectedModCount成员属性，在迭代器对象创建的时候初始化为ArrayList的modCount，即当迭代器对象创建的时候，会将集合修改次数modCount存到expectedModCount里，然后每次遍历取值的时候，都会拿ArrayList集合修改次数modCount与迭代器的expectedModCount比较，如果发生改变，说明集合结构在创建该迭代器后已经发生了改变，直接抛出ConcurrentModificationException异常，如下代码；

```java
final void checkForComodification() {
  if (modCount != expectedModCount)
    throw new ConcurrentModificationException();
}
```

- 先举一个不是多线程的简单例子，在创建迭代器后，往ArrayList插入一条数据，然后利用迭代器遍历，如下代码，将抛出ConcurrentModificationException异常：

```java
public class Main {

  public static void main(String[] args) {
    List<Integer> list = new ArrayList<Integer>();
    for(int i = 0; i < 10; i++){
      list.add(i);
    }

    Iterator<Integer> iterator = list.iterator();
    list.add(10);
    while(iterator.hasNext()){
      iterator.next();
    }
  }
}
```

```java
Exception in thread "main" java.util.ConcurrentModificationException
    at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:859)
    at java.util.ArrayList$Itr.next(ArrayList.java:831)
    at com.pichen.basis.col.Main.main(Main.java:18)
```

- 再来个多线程的例子，我们创建一个t1线程，循环往集合插入数据，另外主线程获取集合迭代器遍历集合，如下代码，在遍历的过程中将抛出ConcurrentModificationException异常：

```java
class ThreadTest implements Runnable{
  private List<Integer> list;
  public ThreadTest(List<Integer> list) {
    this.list = list;
  }
  @Override
  public void run() {
    while(true){
      try {
        Thread.sleep(500);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      this.list.add(10);
    }
  }
}
public class Main {
  public static void main(String[] args) throws InterruptedException {
    List<Integer> list = new ArrayList<Integer>();
    for(int i = 0; i < 10; i++){
      list.add(i);
    }

    ThreadTest t1 = new ThreadTest(list);
    new Thread(t1).start();

    Iterator<Integer> iterator = list.iterator();
    while(iterator.hasNext()){
      System.out.print(iterator.next() + " ");
      Thread.sleep(80);
    }
  }
}
```

- 结果打印：

```java
0 1 2 3 4 5 6 Exception in thread "main" java.util.ConcurrentModificationException
    at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:859)
    at java.util.ArrayList$Itr.next(ArrayList.java:831)
    at com.pichen.basis.col.Main.main(Main.java:44)
```

## ConcurrentModificationException异常解决办法

- 对ArrayList集合修改次数modCount加锁或使用CopyOnWriteArrayList替换ArrayList，建议使用CopyOnWriteArrayList；另外除了List集合外，其它集合像ConcurrentHashMap、CopyOnWriteArraySet也可以避免抛出ConcurrentModificationException异常；

## 什么是CopyOnWriteArrayList

- 看名字就知道，在往集合写操作的时候，复制集合；更具体地说，是在对集合结构进行修改的操作时，复制一个新的集合，然后在新的集合里进行结构修改（插入、删除），修改完成之后，改变原先集合内部数组的引用为新集合即可；

## CopyOnWriteArrayList补充说明

- CopyOnWriteArrayList类实现List<E>, RandomAccess, Cloneable, java.io.Serializable接口
- 与ArrayList功能类似，同样是基于动态数组实现的集合；
- 使用CopyOnWriteArrayList迭代器遍历的时候，读取的数据并不是实时的；
- 每次对集合结构进行修改时，都需要拷贝数据，占用内存较大；

## 源码查看

- 先看个简单的例子，add方法，如下，调用了Arrays.copyOf方法，拷贝旧数组到新数组，然后修改新数组的值，并修改集合内部数组的引用：

```java
public boolean add(E e) {
  final ReentrantLock lock = this.lock;
  lock.lock();
  try {
    Object[] elements = getArray();
    int len = elements.length;
    Object[] newElements = Arrays.copyOf(elements, len + 1);
    newElements[len] = e;
    setArray(newElements);
    return true;
  } finally {
    lock.unlock();
  }
}
```

- 再查看CopyOnWriteArrayList迭代器类的实现，部分代码如下,在创建COWIterator迭代器的时候，仔细查看其构造器源码，需要将集合内部数组的引用传给迭代器对象，由于在集合修改的时候，操作都是针对新的拷贝数组，所以迭代器内部旧数组对象不会改变，保证迭代期间数据不会混乱（虽然不是实时的数据）：

```java
private static class COWIterator<E> implements ListIterator<E> {
  /** Snapshot of the array */
  private final Object[] snapshot;
  /** Index of element to be returned by subsequent call to next.  */
  private int cursor;

  private COWIterator(Object[] elements, int initialCursor) {
    cursor = initialCursor;
    snapshot = elements;
  }
  ........
```

## 结果验证

- 利用前面抛出ConcurrentModificationException的例子，验证使用CopyOnWriteArrayList，
- 首先，创建一个t1线程，循环往集合插入数据，另外主线程获取集合迭代器遍历集合，代码如下，成功运行，并打印出了旧集合的数据(注意数据并不是实时的)。

~~~~java
class ThreadTest implements Runnable{

  private List<Integer> list;
  public ThreadTest(List<Integer> list) {
    this.list = list;
  }
  @Override
  public void run() {
    while(true){
      try {
        Thread.sleep(500);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      this.list.add(10);
    }
  }
}

public class Main {

  public static void main(String[] args) throws InterruptedException {

    List<Integer> list = new CopyOnWriteArrayList<Integer>();
    for(int i = 0; i < 10; i++){
      list.add(i);
    }

    ThreadTest t1 = new ThreadTest(list);
    new Thread(t1).start();

    Iterator<Integer> iterator = list.iterator();
    while(iterator.hasNext()){
      System.out.print(iterator.next() + " ");
      Thread.sleep(80);
    }
  }
}
~~~~



