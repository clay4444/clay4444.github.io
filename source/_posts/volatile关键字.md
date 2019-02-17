---
title: volatile关键字
abbrlink: 111e02e0
date: 2017-06-18 14:18:33
categories:
- 多线程编程
---

- valatile关键字的主要作用是使变量在多个线程间可见
- 没有valatile之前的做法，在变量或者方法上加锁，特点(`弊端`)： 每次只有一个线程来修改，效率低
- JDK 1.5之后针对多线程做了优化，每启动一个新线程，都会把主内存中的数据copy到线程单独的内存中，所以不加volatile关键字的话，即使主内存中的数据发生了改变，线程内存中的数据也不会发生变化
- 适用的场景可以是`很多的线程都在读，一个线程在写`的情况，看图示

{% asset_img volatile关键字线程执行流程图.png %}

测试代码：

~~~java
public class Runhread extends Thread{

  private volatile boolean isRunning = true;
  private void setRunning(boolean isRunning){
    this.isRunning = isRunning;
  }

  public void run(){
    System.out.println("进入run方法..");
    int i = 0;
    while(isRunning == true){
      //..
    }
    System.out.println("线程停止");
  }

  public static void main(String[] args) throws InterruptedException {
    RunThread rt = new RunThread();
    rt.start();
    Thread.sleep(1000);
    rt.setRunning(false);
    System.out.println("isRunning的值已经被设置了false");//永远不会打印
  }
}
~~~

- 但是valatile关键字只有可见性，没有原子性[同步]
- 如果把一个事务可看作是一个程序,它要么完整的被执行,要么完全不执行。这种特性就叫原子性
- 实例代码
- 分析：如果不采用AtomicInteger的话，那么最终的count值不会是10000，说明了valatile关键字没有原子性
- 如果想让它有原子性，那么必须采用AtomicInteger原子类进行操作


~~~java
public class VolatileNoAtomic extends Thread{
  //private static volatile int count;
  private static AtomicInteger count = new AtomicInteger(0);
  private static void addCount(){
    for (int i = 0; i < 1000; i++) {
      //count++ ;
      count.incrementAndGet();
    }
    System.out.println(count);
  }
  public void run(){
    addCount();
  }

  public static void main(String[] args) {
    VolatileNoAtomic[] arr = new VolatileNoAtomic[100];
    for (int i = 0; i < 10; i++) {
      arr[i] = new VolatileNoAtomic();
    }
    for (int i = 0; i < 10; i++) {
      arr[i].start();
    }
  }
}
~~~

- valatile关键字虽然拥有多个线程之间的可见性，但是不具备同步性(也就是原子性)，可以算得上是一个轻量级的synchronezed，性能要比synchronized强很多，不会造成阻塞(在很多的开源的架构中，比如netty的底层代码就大量使用valatile，可见netty性能一定是非常不错的)，这里有一点需要注意，一般valatile用于只针对于多个线程可见的变量操作，并不能代替synchronized的同步功能

  <br/>

- valatile关键字只具有可见性，没有原子性，要实现原子性建议使用atomic类的系列对象，支持原子性操作


- 注意atomic类只保证本身方法原子性，并不保证多次操作的原子性
- 实例代码

~~~java
public class AtomicUse {
  private static AtomicInteger count = new AtomicInteger(0);

  //多个addAndGet在一个方法内是非原子性的，需要加synchronized进行修饰，保证4个addAndGet整体原子性
  /**synchronized*/
  public synchronized int multiAdd(){
    try {
      Thread.sleep(100);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    count.addAndGet(1);
    count.addAndGet(2);
    count.addAndGet(3);
    count.addAndGet(4); //+10
    return count.get();
  }
  public static void main(String[] args) {

    final AtomicUse au = new AtomicUse();

    List<Thread> ts = new ArrayList<Thread>();
    for (int i = 0; i < 100; i++) {
      ts.add(new Thread(new Runnable() {
        @Override
        public void run() {
          System.out.println(au.multiAdd());
        }
      }));
    }

    for(Thread t : ts){
      t.start();
    }
  }
}
~~~

