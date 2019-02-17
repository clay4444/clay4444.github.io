---
title: concurrent包下常用类及其应用
categories:
  - 多线程编程
abbrlink: 417d070a
date: 2017-08-10 14:03:03
---

## CyclicBarrier(障碍)

- 假设一个场景，每个线程代表一个运动员，当运动员都准备好的时候，才一起出发，只要有一个人没有准备好，大家都等待
- 实例代码
- 分析：当zhangsan，lisi，wangwu都准备OK的时候，三个线程才开始同时执行

```java
public class UseCyclicBarrier {

  static class Runner implements Runnable {  
    private CyclicBarrier barrier;  
    private String name;  

    public Runner(CyclicBarrier barrier, String name) {  
      this.barrier = barrier;  
      this.name = name;  
    }  
    @Override  
    public void run() {  
      try {  
        Thread.sleep(1000 * (new Random()).nextInt(5));  
        System.out.println(name + " 准备OK.");  
        barrier.await();  
      } catch (InterruptedException e) {  
        e.printStackTrace();  
      } catch (BrokenBarrierException e) {  
        e.printStackTrace();  
      }  
      System.out.println(name + " Go!!");  
    }  
  } 

  public static void main(String[] args) throws IOException, InterruptedException {  
    CyclicBarrier barrier = new CyclicBarrier(3);  // 3 的意思就是要三个awate()都执行的时候，才能同时启动
    ExecutorService executor = Executors.newFixedThreadPool(3);  

    executor.submit(new Thread(new Runner(barrier, "zhangsan")));  
    executor.submit(new Thread(new Runner(barrier, "lisi")));  
    executor.submit(new Thread(new Runner(barrier, "wangwu")));  

    executor.shutdown();  
  }  

}
```

## CountDownLatch使用

- 经常用于监听某些初始化操作，等初始化执行完毕后，通知主线程继续工作
- 实例代码

```java
public class UseCountDownLatch {

  public static void main(String[] args) {

    final CountDownLatch countDown = new CountDownLatch(2);    这个2代表的是要两次countDown通知t1才能执行

      Thread t1 = new Thread(new Runnable() {
        @Override
        public void run() {
          try {
            System.out.println("进入线程t1" + "等待其他线程处理完成...");
            countDown.await();
            System.out.println("t1线程继续执行...");
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
        }
      },"t1");

    Thread t2 = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          System.out.println("t2线程进行初始化操作...");
          Thread.sleep(3000);
          System.out.println("t2线程初始化完毕，通知t1线程继续...");
          countDown.countDown();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    });
    Thread t3 = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          System.out.println("t3线程进行初始化操作...");
          Thread.sleep(4000);
          System.out.println("t3线程初始化完毕，通知t1线程继续...");
          countDown.countDown();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    });

    t1.start();
    t2.start();
    t3.start();
  }
}
```

- 应用场景
  - 应用程序连接zookeeper的时候，可能需要耗时5s，但是在应用程序主线程中实例化一个对象的时候，就会直接调用，但是这是zookeeper还没有连接成功，就会报错，此时可以在连接zookeeper的时候使用一个whatcher监控它，一旦发现连接成功，就调用CountDownLatch.countDown()方法，主线程中一直在CountDownLatch.awake()方法阻塞，直到CountDownLatch.countDown()方法被调用，

## CountDownLatch和CyclicBarrier的区别

- CountDownLatch是属于一个线程等待，其他的N个线程发出通知，然后等待的线程开始执行
- CyclicBarrier是所有的线程都是参与阻塞的，只有他们都调用了 barrier.await() 方法，才会同时执行
- 即CountDownLatch针对的是一个线程，CyclicBarrier针对的是多个线程

## Future模式和Callable

- Future模式非常适合在处理很耗时很长的业务逻辑时进行使用，可以有效的减小系统的相应时间，提高系统的吞吐量
- 各个子线程都是并行执行的，但是在主线程获取数据的时候，是会阻塞的，这很好理解，因为主线程要使用子线程的数据啊，
- 它的主要作用是在其他子线程执行的时候，我们也暂时不需要子线程返回的结果的时候，我们可以去处理其他的逻辑，等子线程执行完成，我们可以继续使用它返回的数据

```java
public class UseFuture implements Callable<String>{
  private String para;

  public UseFuture(String para){
    this.para = para;
  }

  /**
	 * 这里是真实的业务逻辑，其执行可能很慢
	 */
  @Override
  public String call() throws Exception {
    //模拟执行耗时
    Thread.sleep(5000);
    String result = this.para + "处理完成";
    return result;
  }

  //主控制函数
  public static void main(String[] args) throws Exception {
    String queryStr = "query";
    //构造FutureTask，并且传入需要真正进行业务逻辑处理的类,该类一定是实现了Callable接口的类
    FutureTask<String> future = new FutureTask<String>(new UseFuture(queryStr));

    FutureTask<String> future2 = new FutureTask<String>(new UseFuture(queryStr));
    //创建一个固定线程的线程池且线程数为2,
    ExecutorService executor = Executors.newFixedThreadPool(2);
    //这里提交任务future,则开启线程执行RealData的call()方法执行
    //submit和execute的区别： 第一点是submit可以传入实现Callable接口的实例对象， 第二点是submit方法有返回值

    Future f1 = executor.submit(future);		//单独启动一个线程去执行的
    Future f2 = executor.submit(future2);       //单独启动一个线程去执行的
    System.out.println("请求完毕");

    try {
      //这里可以做额外的数据操作，也就是主程序执行其他业务逻辑
      System.out.println("处理实际的业务逻辑...");
      Thread.sleep(1000);
    } catch (Exception e) {
      e.printStackTrace();
    }
    //调用获取数据方法,如果call()方法没有执行完成,则依然会进行等待
    System.out.println("数据：" + future.get());        //单独启动一个线程去执行的
    System.out.println("数据：" + future2.get());

    executor.shutdown();
  }

}
```

## Semaphore信号量

- Semaphore信号量非常适合高并发访问，新系统上线之前，要对系统的访问量进行评估，当然这个值肯定不是随便拍脑袋就能想出来的，是经过以往的经验，数据，历年的访问量，已经推广力度，进行一个合理的评估，当然评估标准不能太小，太大的话资源达不到实际效果，纯粹浪费资源，太小的话，某个时间点一个高峰值的访问量直接可以压垮系统
- pv(page View) 网站的总访问量，页面浏览量或点击量，用户每刷新一次就会被记录一次
- uv(unique view) 访问网站的一台电脑为一个访问，一般来讲，时间上以00:00-24:00之内相同的IP的客户端值记录一次
- QPS(Query per second) 即每秒查询数目，qps很大程度上代表了系统业务上的繁忙程度，每次请求的背后，可能对应着多次磁盘的IO，多次网络请求，多个CPU时间片等，我们通过qps可以非常直观的了解当前系统业务情况，一旦当前qps超过所设定的预警阈值，可以考虑增加机器对集群扩容，以免压力过大导致宕机，可以根据前期的压力测试得到估值，在结合后期综合运维情况，估算出阈值
- RT(response time) 即请求的响应时间，这个指标非常关键，直接说明前端的用户体验，因此任何系统设计师都想降低RT时间
- 当然还设计CPU，内存，网络，磁盘等情况，更细节问题很多，如select，update，delete、ps等数据库层面的设计

## 如何解决高并发

- 网络层面上
- 服务端(nginx，lvs，hyporxicy)，最主要的是业务的设计，模块化，细粒度话，每个模块单独的做负载均衡
- java代码层面上做限流

## java代码层面上做限流，使用Semaphore信号量

~~~~java
public class UseSemaphore {  

  public static void main(String[] args) {  
    // 线程池  
    ExecutorService exec = Executors.newCachedThreadPool();  
    // 只能5个线程同时访问  
    final Semaphore semp = new Semaphore(5);  
    // 模拟20个客户端访问  
    for (int index = 0; index < 20; index++) {  
      final int NO = index;  
      Runnable run = new Runnable() {  
        public void run() {  
          try {  
            // 获取许可  
            semp.acquire();  
            System.out.println("Accessing: " + NO);  
            //模拟实际业务逻辑
            Thread.sleep((long) (Math.random() * 10000));  
            // 访问完后，释放  
            semp.release();  
          } catch (InterruptedException e) {  
          }  
        }  
      };  
      exec.execute(run);  
    }

    try {
      Thread.sleep(10);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }

    //System.out.println(semp.getQueueLength());
    // 退出线程池  
    exec.shutdown();  
  }  
}
~~~~

