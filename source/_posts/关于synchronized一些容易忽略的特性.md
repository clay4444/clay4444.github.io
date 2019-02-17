---
title: 关于synchronized一些容易忽略的特性
categories:
  - 多线程编程
abbrlink: 623ba864
date: 2017-07-03 17:59:22
---

<br/>

### 对象锁的同步和异步

- 同步 synchronized

  > 同步的概念就是共享，我们要牢牢记住共享这两个字，如果不是共享的资源，就没有必要同步

- 异步asynchronized

  > 异步的概念就是独立，相互之间不受到任何制约，就好像我们学习http的时候，在页面发起的Ajax请求，我们还可以继续浏览或操作页面的内容，二者之间没有任何关系

- 同步的目的就是为了线程安全，其实对于线程安全来说，需要满足两个特性

  > 原子性(同步)
  > 可见性

- 实例代码

~~~java
public class MyObject {

  public synchronized void method1(){
    try {
      System.out.println(Thread.currentThread().getName());
      Thread.sleep(4000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

  /** synchronized */
  public void method2(){
    System.out.println(Thread.currentThread().getName());
  }

  public static void main(String[] args) {

    final MyObject mo = new MyObject();

    /**
		 * 分析：
		 * t1线程先持有object对象的Lock锁，t2线程可以以异步的方式调用对象中的非synchronized修饰的方法
		 * t1线程先持有object对象的Lock锁，t2线程如果在这个时候调用对象中的同步（synchronized）方法则需等待，也就是同步
		 */
    Thread t1 = new Thread(new Runnable() {
      @Override
      public void run() {
        mo.method1();
      }
    },"t1");

    Thread t2 = new Thread(new Runnable() {
      @Override
      public void run() {
        mo.method2();
      }
    },"t2");

    t1.start();
    t2.start();

  }

}
~~~

<br/>

### 脏读

- 对于对象的同步和异步的方法，我们在设计自己的程序的时候，一定要考虑问题的整体，不然就会出现数据不一致的错误，很经典的错误就是脏读(dirtyread)
- 业务整体需要使用完整的synchronized，保持业务的原子性。
- 实例代码
- 分析: 存在两个线程，主线程一直执行，期间开启了第二个线程，第二个线程修改了username和password，但是修改完username的睡了2秒，但是在这期间，主线程也一直在执行，期间只睡了一秒，又因为是一个对象，所以读到的肯定是未修改前的password，为了解决这个问题，需要在getValue方法上也加上synchronized，让整个业务同步
- 所以有些场景下，我们应该保证多个方法，能进行同步，避免出现脏读的错误

~~~java
public class DirtyRead {

  private String username = "bjsxt";
  private String password = "123";

  public synchronized void setValue(String username, String password){
    this.username = username;

    try {
      Thread.sleep(2000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }

    this.password = password;

    System.out.println("setValue最终结果：username = " + username + " , password = " + password);
  }

  public void getValue(){
    System.out.println("getValue方法得到：username = " + this.username + " , password = " + this.password);
  }

  public static void main(String[] args) throws Exception{

    final DirtyRead dr = new DirtyRead();
    Thread t1 = new Thread(new Runnable() {
      @Override
      public void run() {
        dr.setValue("z3", "456");		
      }
    });
    t1.start();
    Thread.sleep(1000);

    dr.getValue();
  }
}
~~~

<br/>

### synchronized的其他概念

- synchronize锁重入 关键字synchronized拥有锁重入的功能，也就是在使用synchronized时，当一个线程得到了一个对象的锁后，再次请求此对象时是可以再次得到该对象的锁，
- 实例代码1
- 分析： 当SyncDubbo1调用method1方法时，使用sd对象这个锁，当method1中调用method2时，又继续请求使用sd这个对象作为锁，调用method3同样如此

~~~java
public class SyncDubbo1 {

  public synchronized void method1(){
    System.out.println("method1..");
    method2();
  }
  public synchronized void method2(){
    System.out.println("method2..");
    method3();
  }
  public synchronized void method3(){
    System.out.println("method3..");
  }

  public static void main(String[] args) {
    final SyncDubbo1 sd = new SyncDubbo1();
    Thread t1 = new Thread(new Runnable() {
      @Override
      public void run() {
        sd.method1();
      }
    });
    t1.start();
  }
}
~~~

<br/>

- 实例代码2
- 分析：调用sub.operationSub();operationSub方法是被synchronized修饰的，由于synchronized用的锁是this，即sub这个对象
- 在operationSub方法中调用父类的this.operationSup();，operationSup方法也是synchronized修饰的，
- Sub是Main的子类，所以就相当于再次请求这个对象作为父类中operationSup方法的锁，
- 当一个线程得到了一个对象的锁后，再次请求此对象时是可以再次得到该对象的锁，
- 所以这样写是线程安全的，

~~~java
public class SyncDubbo2 {

  static class Main {
    public int i = 10;
    public synchronized void operationSup(){
      try {
        i--;
        System.out.println("Main print i = " + i);
        Thread.sleep(100);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }

  static class Sub extends Main {
    public synchronized void operationSub(){
      try {
        while(i > 0) {
          i--;
          System.out.println("Sub print i = " + i);
          Thread.sleep(100);		
          this.operationSup();
        }
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }

  public static void main(String[] args) {

    Thread t1 = new Thread(new Runnable() {
      @Override
      public void run() {
        Sub sub = new Sub();
        sub.operationSub();
      }
    });
    t1.start();
  }
}
~~~

<br/>

### 锁的异常处理

- 对于web应用程序，异常释放锁的情况，如果不及时处理，很可能对你的应用程序造成严重的错误，比如你在执行一个队列任务，很多对象都在等待第一个对象正确执行完毕再去释放锁，但是第一个对象由于异常的出现，导致业务罗没有正确执行完毕再去释放锁，那么可想而知后续的对象执行的都是错误的逻辑，所以这一点一定要引起注意，在编写代码的时候，一定要考虑周全
- 实例代码
- 分析，如果第10个任务和后续的任务没有关联关系的话，可以在catch中日志打印，
- 如果是有关系的话，此时可以抛出InterruptedException打断异常，或者是RunnableException终止线程

~~~java
public class SyncException {

  private int i = 0;
  public synchronized void operation(){
    while(true){
      try {
        i++;
        Thread.sleep(100);
        System.out.println(Thread.currentThread().getName() + " , i = " + i);
        if(i == 20){
          Integer.parseInt("a");  //异常
          throw new RuntimeException();
        }
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }

  public static void main(String[] args) {

    final SyncException se = new SyncException();
    Thread t1 = new Thread(new Runnable() {
      @Override
      public void run() {
        se.operation();
      }
    },"t1");
    t1.start();
  }
}
~~~

<br/>

### 当一个对象充当一把锁的时候，即使这个对象中的属性发生变化了，这把锁也还是不变的

- 实例代码
- 分析：即使t1线程把modifyLock这个对象中的name和age属性修改了，但是这段程序仍然是线程安全的。

~~~java
public class ModifyLock {

  private String name ;
  private int age ;

  public String getName() {
    return name;
  }
  public void setName(String name) {
    this.name = name;
  }
  public int getAge() {
    return age;
  }
  public void setAge(int age) {
    this.age = age;
  }

  public synchronized void changeAttributte(String name, int age) {
    try {
      System.out.println("当前线程 : "  + Thread.currentThread().getName() + " 开始");
      this.setName(name);
      this.setAge(age);

      System.out.println("当前线程 : "  + Thread.currentThread().getName() + " 修改对象内容为： " 
                         + this.getName() + ", " + this.getAge());

      Thread.sleep(2000);
      System.out.println("当前线程 : "  + Thread.currentThread().getName() + " 结束");
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

  public static void main(String[] args) {
    final ModifyLock modifyLock = new ModifyLock();
    Thread t1 = new Thread(new Runnable() {
      @Override
      public void run() {
        modifyLock.changeAttributte("张三", 20);
      }
    },"t1");
    Thread t2 = new Thread(new Runnable() {
      @Override
      public void run() {
        modifyLock.changeAttributte("李四", 21);
      }
    },"t2");

    t1.start();
    try {
      Thread.sleep(100);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    t2.start();
  }
}
~~~

