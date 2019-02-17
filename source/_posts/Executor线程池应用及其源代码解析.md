---
title: Executor线程池应用及其源代码解析
categories:
  - 多线程编程
tags:
  - 源码解析系列
abbrlink: 8b920ef3
date: 2017-08-03 13:47:32
---

## Executor框架

- 为了更好的控制多线程，JDK提供了一套线程框架Executor，帮助开发人员更有效的进行线程控制，他们都在java.util.concurrent包中，是JDK并发包的核心，其中有一个比较重要的类，Executors，它扮演这线程工厂的角色，我们通过Executors可以创建特定功能的线程池
- Executors创建线程池的方法

1. newFixedThreadPool()方法，该方法返回一个固定数量的线程池，该方法的线程数始终不变，当有一个任务提交时，若线程池中空闲，则立即执行，若没有，则会被暂缓在一个任务队列中等待有空闲的线程去执行
2. newSingleThreadExecutor()方法，创建一个线程的线程池，若空闲则执行，若没有空闲线程，则暂缓在任务队列中，
3. newCachedThreadPool()方法，返回一个可根据实际情况调整线程个数的线程池，不限制最大的线程数量，若有空闲线程则执行任务，若无任务则不创建线程，并且每一个空闲线程会在60s后自动回收
4. newScheduledThreadPool()方法，该方法返回一个SchededExecutorService对象，但该线程池可以指定线程的数量

<br/>

## 线程池的体系结构：

java.util.concurrent.Executor : 负责线程的使用与调度的根接口

```java
|--ExecutorService 子接口: 线程池的主要接口
	|--ThreadPoolExecutor 线程池的实现类
	|--ScheduledExecutorService 子接口：负责线程的调度
		|--ScheduledThreadPoolExecutor ：继承 ThreadPoolExecutor， 实现 ScheduledExecutorService
```
<br/>

## 自定义线程池

- 若Executors工厂类无法满足我们的需求，可以自己去创建自定义的线程池，其实Executors工厂类里面的创建线程方法其内部实现均是用了ThreadOPoolExecutor这个类，这个类可以自定义线程池，构造方法如下

```java
public ThreadPoolExecutor(int corePoolSize              当前核心线程数(线程池初始化的时候就存在的线程数目)
                          int maximumPoolSize,          最大线程数
                          long keepAliveTime,           其中的每一个线程保持或者的时间
                          TimeUnit unit,                时间戳
                          BlockingQueue<Runnable> workQueue         如果没有线程可以执行任务的话，任务的存储队列
                          ThreadFactory threadFactory,              
                          RejectedExecutionHandler handler){...}    线程满了，队列也满了，拒绝的方法
```

<br/>

## 各个线程池实现源代码

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
  return new ThreadPoolExecutor(nThreads, nThreads,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>());
}
```

```java
public static ExecutorService newSingleThreadExecutor() {
  return new FinalizableDelegatedExecutorService
    (new ThreadPoolExecutor(1, 1,
                            0L, TimeUnit.MILLISECONDS,
                            new LinkedBlockingQueue<Runnable>()));
}
```

```java
public static ExecutorService newCachedThreadPool() {
  return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                60L, TimeUnit.SECONDS,
                                new SynchronousQueue<Runnable>());
}
```

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, TimeUnit.NANOSECONDS,
          new DelayedWorkQueue());
}
```

<br/>

## Executors.newScheduledThreadPool使用实例

- 类似于timer定时功能

```java
class Temp extends Thread {
  public void run() {
    System.out.println("run");
  }
}

public class ScheduledJob {

  public static void main(String args[]) throws Exception {

    Temp command = new Temp();
    ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

    ScheduledFuture<?> scheduleTask = scheduler.scheduleWithFixedDelay(command, 5, 1, TimeUnit.SECONDS);

  }
}
```

<br/>

## 自定义线程池使用详细说明

### 1. 使用有界队列时

- 在使用有界队列时，若有新的任务需要执行，如果线程池实际线程数小于corePoolSize，则优先创建线程，
- 若大于corePoolSize，则会将任务加入队列，
- 若队列已满，则在总线程数不大于maximumPoolSize的前提下，创建新的线程，
- 若线程数大于maximumPoolSize，则执行拒绝策略。或其他自定义方式。

```java
public class UseThreadPoolExecutor1 {

  public static void main(String[] args) {
    ThreadPoolExecutor pool = new ThreadPoolExecutor(
      1, 				//coreSize
      2, 				//MaxSize
      60, 			//60
      TimeUnit.SECONDS, 
      new ArrayBlockingQueue<Runnable>(3)			//指定一种队列 （有界队列）
      //new LinkedBlockingQueue<Runnable>()
      , new MyRejected()
      //, new DiscardOldestPolicy()
    );

    MyTask mt1 = new MyTask(1, "任务1");
    MyTask mt2 = new MyTask(2, "任务2");
    MyTask mt3 = new MyTask(3, "任务3");
    MyTask mt4 = new MyTask(4, "任务4");
    MyTask mt5 = new MyTask(5, "任务5");
    MyTask mt6 = new MyTask(6, "任务6");

    pool.execute(mt1);
    pool.execute(mt2);
    pool.execute(mt3);
    pool.execute(mt4);
    pool.execute(mt5);
    pool.execute(mt6);

    pool.shutdown();

  }
}
```

### 2. 使用无界队列时

- 在使用无界队列时，LinkedBlockingQueue，与有界队列相比，除非系统资源耗尽，否则无界的任务队列无存在任务入队失败的情况，当有新任务到来，系统的线程数小于corePoolSize时，则新建线程执行任务，当达到corePoolSize后，就不会继续增加，若后续仍有新的任务加入，而又没有空闲的线程资源，则任务直接进入队列等待，若任务创建和处理的速度差异很大，无界队列会保持持续增长，直到耗尽系统内存

```java
public class UseThreadPoolExecutor2 implements Runnable{

  private static AtomicInteger count = new AtomicInteger(0);

  @Override
  public void run() {
    try {
      int temp = count.incrementAndGet();
      System.out.println("任务" + temp);
      Thread.sleep(2000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

  public static void main(String[] args) throws Exception{
    //System.out.println(Runtime.getRuntime().availableProcessors());
    BlockingQueue<Runnable> queue = 
      //new LinkedBlockingQueue<Runnable>();
      new ArrayBlockingQueue<Runnable>(10);
    ExecutorService executor  = new ThreadPoolExecutor(
      5, 		//core
      10, 	//max           在无界队列中，这个参数是没用的
      120L, 	//2fenzhong
      TimeUnit.SECONDS,
      queue);

    for(int i = 0 ; i < 20; i++){
      executor.execute(new UseThreadPoolExecutor2());
    }
    Thread.sleep(1000);
    System.out.println("queue size:" + queue.size());		//10
    Thread.sleep(2000);
  }

}
```

<br/>

### 3. 使用同步移交时

对于非常大的或者无界的线程池，可以通过使用 SynchronousQueue 来避免任务排队，以及直接将任务从生产者移交给工作者线程。SynchronousQueue 不是一个真正的队列，而是一种在线程之间进行移交的机制。要将一个元素放入SynchronousQueue 中，必须有另一个元素正在等待接受这个元素。如果没有线程正在等待，并且线程池的当前大小小于最大值，那么ThreadPoolExecutor 将创建一个新的线程，否则根据饱和策略，这个任务将被拒绝. 使用直接移交将更高效，因为任务会直接移交给执行它的线程，而不是被首先放到队列中，然后由工作者线程从队到中提取该任务。只有当线程池是无界的或者可以拒绝任务时， SynchronousQueue才有实际价值。在newCachedThreadPool 工厂方能中就使用了SynchronousQueue.

~~~~java
public static ExecutorService newCachedThreadPool() {
  return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                60L, TimeUnit.SECONDS,
                                new SynchronousQueue<Runnable>());
}
~~~~

<br/>

## 拒绝策略

### 1.JDK提供的策略

- AbortPolicy 直接抛出异常组织系统正常工作

AbortPolicy是默认的饱和策略，就是中止任务，该策略将抛出RejectedExecutionException。调用者可以捕获这个异常然后去编写代码处理异常。

<br/>

- CollerRunsPolicy 只要线程池未关闭，该策略直接在调用者线程中，运行当前被丢弃的任务

CallerRunsPolicy是 “调用者运行” 策略，实现了一种调节机制 。它不会抛弃任务，也不会抛出异常。 而是将任务回退到调用者。它不会在线程池中执行任务，而是在一个调用了Executor的线程中执行该任务。

<br/>

我们可以将WebServer 示例修改为使用在界队列和 “调用者运行” 饱和策略，当线程池中的所有线程都被占用，并且工作队列被填满后，下一个任务会在调用 execute 时在主线程中执行。由于执行任务需要一定的时间，因此主线程至少在一段时间内不能提交任何任务，从而使得工作者线程有时间来处理完正在执行的任务。在这期间，主线程不会调用accept，因此到达的请求将被保存在TCP 层的队列中而不是在应用程序的队列中。如果持续过载，那么TCP 层将最终发现它的请求队列被填满，因此同样会开始抛弃请求。当服务器过载时，这种过载情况会逐渐向外蔓延开来，从线程池到工作队列到应用程序再到TCP 层，最终要达到客户端，导致服务器在高负载下实现一种平缓的性能降低。

~~~java
/**
 * 调用者运行的饱和策略
 */
public class ThreadDeadlock2 {
  ExecutorService exec = new ThreadPoolExecutor(0,
                                                2,
                                                60L,
                                                TimeUnit.SECONDS,
                                                new SynchronousQueue<Runnable>(),
                                                new ThreadPoolExecutor.CallerRunsPolicy());

  private void putrunnable() {
    for (int i = 0; i < 4; i++) {
      exec.submit(new Runnable() {

        @Override
        public void run() {
          // TODO Auto-generated method stub
          while (true) {
            System.out.println(Thread.currentThread().getName());
            try {
              Thread.sleep(500);
            } catch (InterruptedException e) {
              // TODO Auto-generated catch block
              e.printStackTrace();
            }
          }
        }
      });
    }
  }
  public static void main(String[] args) {
    new ThreadDeadlock2().putrunnable();
  }
}
~~~

<br/>

执行结果：

{% asset_img 八1.png %}

<br/>

 当工作队列被填满之后，没有预定义的饱和策略来阻塞execute。通过使用 Semaphore （信号量）来限制任务的到达率，就可以实现这个功能。在下面的 BoundedExecutor 中给出了这种方法，该方法使用了一个无界队列，并设置信号量的上界设置为线程池的大小加上可排队任务的数量，这是因为信号量需要控制正在执行的和正在等待执行的任务数量。

~~~java
/**
 * 8.4 使用Semaphore来控制任务的提交速率
 */
public class BoundedExecutor {
  private final Executor exec;
  private final Semaphore semaphore;
  int bound;

  public BoundedExecutor(Executor exec,int bound){
    this.exec = exec;
    this.semaphore = new Semaphore(bound);
    this.bound = bound;
  }

  public void submitTask(final Runnable command) throws InterruptedException{
    //通过 acquire() 获取一个许可
    semaphore.acquire();
    System.out.println("----------当前可用的信号量个数:"+semaphore.availablePermits());
    try {
      exec.execute(new Runnable() {

        @Override
        public void run() {
          try {
            System.out.println("线程" + Thread.currentThread().getName() +"进入，当前已有" + (bound-semaphore.availablePermits()) + "个并发");
            command.run();
          } finally {
            //release() 释放一个许可
            semaphore.release();
            System.out.println("线程" + Thread.currentThread().getName() +
                               "已离开，当前已有" + (bound-semaphore.availablePermits()) + "个并发");
          }
        }
      });
    } catch (RejectedExecutionException e) {
      semaphore.release();
    }
  }
}
~~~

<br/>

测试程序：

~~~java
public class BoundedExecutor_main {
  public static void main(String[] args) throws InterruptedException{
    ExecutorService exec = Executors.newCachedThreadPool();    
    BoundedExecutor e = new BoundedExecutor(exec, 3);

    for(int i = 0;i<5;i++){
      final int c = i;
      e.submitTask(new Runnable() {

        @Override
        public void run() {
          System.out.println("执行任务:" +c);
        }
      });
    }
  }
}
~~~

<br/>

执行结果：

{% asset_img 八2.png %}

<br/>

- DisCardOldestPolicy 丢弃最老的一个请求，尝试再次提交新的任务

当新提交的任务无法保存到队列中等待执行时，DiscardPolicy会稍稍的抛弃该任务，DiscardOldestPolicy则会抛弃最旧的（下一个将被执行的任务），然后尝试重新提交新的任务。如果工作队列是那个优先级队列时，搭配DiscardOldestPolicy饱和策略会导致优先级最高的那个任务被抛弃，所以两者不要组合使用。

<br/>

> 分析：因为coreSize是1，所以1执行，2,3,4进入队列，突然发现MaxSize是2，所以又启动一个线程，5也执行，6被拒绝，但是拒绝策略是删除最老的任务，因为2是最先进入队列的，所以2被拒绝，所以执行顺序是15346

```java
public class UseThreadPoolExecutor1 {

  public static void main(String[] args) {
    ThreadPoolExecutor pool = new ThreadPoolExecutor(
      1, 				//coreSize
      2, 				//MaxSize
      60, 			//60
      TimeUnit.SECONDS, 
      new ArrayBlockingQueue<Runnable>(3)			//指定一种队列 （有界队列）
      //new LinkedBlockingQueue<Runnable>()
      //, new MyRejected()
      , new DiscardOldestPolicy()
    );

    MyTask mt1 = new MyTask(1, "任务1");
    MyTask mt2 = new MyTask(2, "任务2");
    MyTask mt3 = new MyTask(3, "任务3");
    MyTask mt4 = new MyTask(4, "任务4");
    MyTask mt5 = new MyTask(5, "任务5");
    MyTask mt6 = new MyTask(6, "任务6");

    pool.execute(mt1);
    pool.execute(mt2);
    pool.execute(mt3);
    pool.execute(mt4);
    pool.execute(mt5);
    pool.execute(mt6);

    pool.shutdown();

  }
}
```

- DiscardPolicy 丢弃无法处理的任务，不给予任何处理

### 2.自定义拒绝策略

- 如果需要自定义拒绝策略可以实现RejectedExecutionHandler接口

```java
public class UseThreadPoolExecutor1 {

  public static void main(String[] args) {
    ThreadPoolExecutor pool = new ThreadPoolExecutor(
      1, 				//coreSize
      2, 				//MaxSize
      60, 			//60
      TimeUnit.SECONDS, 
      new ArrayBlockingQueue<Runnable>(3)			//指定一种队列 （有界队列）
      //new LinkedBlockingQueue<Runnable>()
      , new MyRejected()
      //, new DiscardOldestPolicy()
    );

    MyTask mt1 = new MyTask(1, "任务1");
    MyTask mt2 = new MyTask(2, "任务2");
    MyTask mt3 = new MyTask(3, "任务3");
    MyTask mt4 = new MyTask(4, "任务4");
    MyTask mt5 = new MyTask(5, "任务5");
    MyTask mt6 = new MyTask(6, "任务6");

    pool.execute(mt1);
    pool.execute(mt2);
    pool.execute(mt3);
    pool.execute(mt4);
    pool.execute(mt5);
    pool.execute(mt6);

    pool.shutdown();

  }
}
```

```java
public class MyRejected implements RejectedExecutionHandler{

  public MyRejected(){
  }

  @Override
  public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
    System.out.println("自定义处理..");
    System.out.println("当前被拒绝任务为：" + r.toString());

  }
}
```

- 自定义拒绝策略的两种解决方案

1. 拒绝任务的时候通过HTTP请求把被拒绝的任务传给客户端，(不建议，因为如果在高峰期的话，发送http请求会很占资源)
2. 推荐的解决策略是记录日志，然后在不是高峰期的时候，通过定时的job去解析日志，再次执行。