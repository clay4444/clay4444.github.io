---
title: BlockQueue队列内部实现剖析及简单模拟
categories:
  - 多线程编程
abbrlink: 3c6f5864
date: 2017-07-08 18:36:14
---

## BlockQueue:

- 顾名思义，首先它是一个队列，并且支持阻塞的队列，阻塞的放入和得到数据，我们要实现LinkedBlockingQueue下面两个简单的方法put和take
- put(anObject): 把anObject加到BlockingQueue里,如果BlockQueue没有空间,则调用此方法的线程被阻断，直到BlockingQueue里面有空间再继续.
- take: 取走BlockingQueue里排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到BlockingQueue有新的数据被加入.
- 实现代码

~~~java
public class MyQueueL {

  //要有存储对象的集合
  private LinkedList<Object> queue = new LinkedList<>();

  //要有一个计数器
  private AtomicInteger count = new AtomicInteger(0);

  //最小长度
  private final int minSize = 0;

  //最大长度
  private final int maxSize;

  //构造器
  public MyQueueL(int len){
    this.maxSize = len;
  }

  Object lock = new Object();

  public void put(Object obj){
    synchronized (lock) {
      //先判断是否还有存储空间，如果没有
      if(queue.size() == maxSize){
        try {
          lock.wait();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }

      //如果有，集合中元素+1
      queue.add(obj);
      //计数器+1
      count.incrementAndGet();

      //在考虑一个极端情况，另外一个线程取的时候没有元素，那么他就在阻塞，所以我们要唤醒他
      lock.notify();

      System.out.println("新加入的元素为:" + obj);
    }
  }

  public Object get(){
    Object res = null;

    synchronized (lock) {
      //取的时候先判断是否有元素,如果此时没有元素
      while(queue.size() == minSize){
        try {
          lock.wait();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
      //如果此时有元素的话，直接取
      res = queue.removeFirst();
      //技术器-1
      count.decrementAndGet();
      //还要在考虑一种极端情况，如果此时队列中的元素是满的话，则put的线程在阻塞，此时取出一个需要唤醒
      lock.notify();
    }
    return res;
  }

  public static void main(String[] args) {

    final MyQueueL queue = new MyQueueL(5);
    queue.put("a");
    queue.put("b");
    queue.put("c");
    queue.put("d");
    queue.put("e");

    System.out.println("当前queue中的长度是  " + queue.queue.size());

    Thread put = new Thread(new Runnable(){

      @Override
      public void run() {
        queue.put("f");
        queue.put("g");
      }

    });

    put.start();

    Thread get = new Thread(new Runnable(){

      @Override
      public void run() {
        Object res1 = queue.get();
        System.out.println("get一个元素   " + res1);
        Object res2 = queue.get();
        System.out.println("get一个元素  " + res2);
      }

    });

    try {
      TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }

    get.start();
  }

}
~~~

