---
title: ThreadLocal详解及其底层实现分析
categories:
  - 多线程编程
tags:
  - 源码解析系列
abbrlink: db40873f
date: 2017-07-13 10:59:32
---

## ThreadLocal

- 概念： 线程局部变量，是一种多线程并发访问变量的解决方案，与其synchronized等加锁的方式不同，Thread完全不提供锁，而使用以空间换时间的手段，为每个线程提供变量的独立副本，以保障线程安全
- 从性能上说，Thread不具有绝对的优势，在并发不是很高的时候，加锁的性能会更好，但作为一套与锁完全无关的线程安全解决方案，在高并发或者竞争激烈的场景下，使用ThreadLocal可以在一定程度上减少锁的竞争
- 实例代码
- 分析：t2中执行ct.getTh()的时候会为空

~~~java
public class ConnThreadLocal {

  public static ThreadLocal<String> th = new ThreadLocal<String>();

  public void setTh(String value){
    th.set(value);
  }
  public void getTh(){
    System.out.println(Thread.currentThread().getName() + ":" + this.th.get());
  }

  public static void main(String[] args) throws InterruptedException {

    final ConnThreadLocal ct = new ConnThreadLocal();
    Thread t1 = new Thread(new Runnable() {
      @Override
      public void run() {
        ct.setTh("张三");
        ct.getTh();
      }
    }, "t1");

    Thread t2 = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          Thread.sleep(1000);
          ct.getTh();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    }, "t2");

    t1.start();
    t2.start();
  }
}
~~~

<br/>

### 底层原理解析

<br/>

#### 1. ThreadLocal是什么？　　

​	ThreadLocal是什么呢？其实ThreadLocal并非是一个线程的本地实现版本，它并不是一个Thread，而是threadlocalvariable(线程局部变量)。也许把它命名为ThreadLocalVar更加合适。ThreadLocal功能非常简单，就是为每一个使用该变量的线程都提供一个变量值的副本，是Java中一种较为特殊的线程绑定机制，是每一个线程都可以独立地改变自己的副本，而不会和其它线程的副本冲突。

<br/>

　　从线程的角度看，每个线程都保持一个对其线程局部变量副本的隐式引用，只要线程是活动的并且ThreadLocal实例是可访问的；在线程消失之后，其线程局部实例的所有副本都会被垃圾回收（除非存在对这些副本的其他引用）。

<br/>

　　通过ThreadLocal存取的数据，总是与当前线程相关，也就是说，JVM 为每个运行的线程，绑定了私有的本地实例存取空间，从而为多线程环境常出现的并发访问问题提供了一种隔离机制。

<br/>

#### 2. ThreadLocal的接口方法　

ThreadLocal类接口很简单，只有4个方法，我们先来了解一下。

<br/>

**void set(Object value)**


**public Object get()**


**public void remove()**


**protected Object initialValue()**


<br/>

set方法：

~~~~java
/**
 * Sets the current thread's copy of this thread-local variable
 * to the specified value.  Most subclasses will have no need to
 * override this method, relying solely on the {@link #initialValue}
 * method to set the values of thread-locals.
 *
 * @param value the value to be stored in the current thread's copy of
 *        this thread-local.
 */
public void set(T value) {
  // 获取当前线程对象
  Thread t = Thread.currentThread();
  // 获取当前线程本地变量Map
  ThreadLocalMap map = getMap(t);
  // map不为空
  if (map != null)
    // 存值
    map.set(this, value);
  else
    // 创建一个当前线程本地变量Map
    createMap(t, value);
}

/**
 * Get the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param  t the current thread
 * @return the map
 */
ThreadLocalMap getMap(Thread t) {
  // 获取当前线程的本地变量Map
  return t.threadLocals;
}
~~~~

这里注意，ThreadLocal中是有一个Map，但这个Map不是我们平时使用的Map，而是ThreadLocalMap，ThreadLocalMap是ThreadLocal的一个内部类，不对外使用的。当使用ThreadLocal存值时，首先是获取到当前线程对象，然后获取到当前线程本地变量Map，最后将当前使用的ThreadLocal和传入的值放到Map中，也就是说ThreadLocalMap中存的值是[ThreadLocal对象, 存放的值]，这样做的好处是，每个线程都对应一个本地变量的Map，所以**一个线程可以存在多个线程本地变量**。

<br/>

get方法：

~~~~java
/**
 * Returns the value in the current thread's copy of this
 * thread-local variable.  If the variable has no value for the
 * current thread, it is first initialized to the value returned
 * by an invocation of the {@link #initialValue} method.
 *
 * @return the current thread's value of this thread-local
 */
public T get() {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null)
      return (T)e.value;
  }
  // 如果值为空，则返回初始值
  return setInitialValue();
}
~~~~

有了之前set方法的分析，get方法也同理，需要说明的是，如果没有进行过set操作，那从ThreadLocalMap中拿到的值就是null，这时get方法会返回初始值，也就是调用initialValue()方法，ThreadLocal中这个方法默认返回null。当我们有需要第一次get时就能得到一个值时，可以继承ThreadLocal，并且覆盖initialValue()方法。

<br/>

#### 3.一个TheadLocal实例 

~~~~java
/**
 * 
 * @ClassName: SequenceNumber
 * @author xingle
 * @date 2015-3-9 上午9:54:23
 */
public class SequenceNumber {
  //①通过匿名内部类覆盖ThreadLocal的initialValue()方法，指定初始值  
  private static ThreadLocal<Integer> seqNum = new ThreadLocal<Integer>(){
    public Integer initialValue(){
      return 0;
    }
  };

  //②获取下一个序列值  
  public int getNextNum(){
    seqNum.set(seqNum.get()+1);
    return seqNum.get();
  }

  public static void main(String[] args){

    SequenceNumber sn = new SequenceNumber();
    //③ 3个线程共享sn，各自产生序列号  
    TestClient t1 = new TestClient(sn);
    TestClient t2 = new TestClient(sn);
    TestClient t3 = new TestClient(sn);
    t1.start();
    t2.start();
    t3.start();
  }

  private static class TestClient extends Thread{
    private SequenceNumber sn;
    public TestClient(SequenceNumber sn){
      this.sn = sn;
    }

    public void run(){
      //④每个线程打出3个序列值  
      for (int i = 0 ;i<3;i++){
        System.out.println("thread["+Thread.currentThread().getName()+"] sn["+sn.getNextNum()+"]");
      }
    }
  }
}
~~~~

通常我们通过匿名内部类的方式定义ThreadLocal的子类，提供初始的变量值，如①处所示。TestClient 线程产生一组序列号，在③处，我们生成3个TestClient，它们共享同一个SequenceNumber实例。运行以上代码，在控制台上输出以下的结果： 

{% asset_img c1.png %}

<br/>

每个线程所产生的序号虽然都共享同一个Sequence Number实例，但它们并没有发生相互干扰的情况，而是各自产生独立的序列号，这是因为我们通过ThreadLocal为每一个线程提供了单独的副本。