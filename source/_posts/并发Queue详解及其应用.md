---
title: 并发Queue详解及其应用
categories:
  - 多线程编程
abbrlink: f1a00836
date: 2017-07-23 12:11:26
---

## 总结：

- ArrayBlockingQueue 没有读分离 有界队列
- LinkedBlockingQueue 读写分类 无界队列
- SynchronousQueue 没有缓冲，没有容量
- PriorityBlockingQueue 优先级队列，Comparable接口判断 无界队列
- DelayQueue 延迟队列， Delayed接口判断 无界队列

## 在并发队列上提供了两套实现，一个是以ConcurrentLinkedQueue为代表的高性能队列，一个是以BlockQueue接口为代表的阻塞队列，不论哪种，都继承自Queue

<br/>

### （1.）ConcurrentLinkedQueue是一个适用于高并发场景下的队列，通过无锁的方式，实现了高并发状态下的高性能，通常ConcurrentQueue性能好于BlockingQueue，他是一个基于连接节点的无界线程安全队列，该队列的元素遵循先进先出的原则，，头是最先加入的，尾是最近加入的，该队列不允许null元素

<br/>

- ConcurrentLinkedQueue重要方法
- add()和offer()都是加入元素的方法，(在ConcurrentLinkedQueue中，这两个方法没有任何区别)
- poll()和peek()都是去取元素节点，区别在于前者会删除元素，后者不会
- 实例代码

```java
ConcurrentLinkedQueue<String> q = new ConcurrentLinkedQueue<String>();
	q.offer("a");
	q.offer("b");
	q.offer("c");
	q.offer("d");
	q.add("e");
	
	System.out.println(q.poll());	//a 从头部取出元素，并从队列里删除
	System.out.println(q.size());	//4
	System.out.println(q.peek());	//b
	System.out.println(q.size());	//4
```

<br/>

### （2.）BlockingQuue接口

- (1.) ArrayBlockingQueue，基于数组的阻塞式队列，在ArrayBlockingQueue内部，维护了一个定长数组，以便缓存队列中的数据对象，其内部没实现读写分离，也就意味这生产和消费不能完全并行，长度是需要定义的，可以指定先进先出或者先进后出，也叫有界队列，在很多场合都非常适合使用

```java
ArrayBlockingQueue<String> array = new ArrayBlockingQueue<String>(5);
	array.put("a");
	array.put("b");
	array.add("c");
	array.add("d");
	array.add("e");
	array.add("f");
	//System.out.println(array.offer("a", 3, TimeUnit.SECONDS));
```

<br/>

- (2.)LinkedBlockingQueue，基于链表的阻塞队列，同ArrayBlockingQueue类似，其内部也维持着一个数据缓冲队列(该队列由一个链表构成)，LinkedBlockingQueue之所以能够高效的处理并发数据，是因为其内部实现采用分离锁(读写分离两个锁)，从而实现生产者和消费者操作的完全并行运行，他是一个无界队列

```java
LinkedBlockingQueue<String> q = new LinkedBlockingQueue<String>();
	q.offer("a");
	q.offer("b");
	q.offer("c");
	q.offer("d");
	q.offer("e");
	q.add("f");
	//System.out.println(q.size());
	
//		for (Iterator iterator = q.iterator(); iterator.hasNext();) {
//			String string = (String) iterator.next();
//			System.out.println(string);
//		}
	
	List<String> list = new ArrayList<String>();
	System.out.println(q.drainTo(list, 3));
	System.out.println(list.size());
	for (String string : list) {
		System.out.println(string);
	}
```

<br/>

- (3.)SynchronousQueue：一种没有缓冲的队列，生产者产生的数据直接会被消费者获取并消费。
- 其实SynchronousQueue是没有容量的，调用add方法只是会把数据直接扔给要消费的线程，不会经过Queue

```java
final SynchronousQueue<String> q = new SynchronousQueue<String>();
Thread t1 = new Thread(new Runnable() {
  @Override
  public void run() {
    try {
      System.out.println(q.take());
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }
});
t1.start();
Thread t2 = new Thread(new Runnable() {

  @Override
  public void run() {
    q.add("asdasd");
  }
});
t2.start();
```

<br/>

- (4.)PriorityBlockingQueue：基于优先级的阻塞队列(优先级的判断通过构造函数传入的Cpmpator对象来决定，也就是说传入队列的对象必须实现Comparable接口)，在实现PriorityBlockingQueue时，内部控制线程同步的锁采用的是公平锁，它也是一个无界的队列

```java
public class Task implements Comparable<Task>{

  private int id ;
  private String name;
  public int getId() {
    return id;
  }
  public void setId(int id) {
    this.id = id;
  }
  public String getName() {
    return name;
  }
  public void setName(String name) {
    this.name = name;
  }

  @Override
  public int compareTo(Task task) {
    return this.id > task.id ? 1 : (this.id < task.id ? -1 : 0);  
  }

  public String toString(){
    return this.id + "," + this.name;
  }

}
```

> 分析：它并不是在添加的时候就排序的，而是在每次take的时候才排序，每次take就把最小的数据拿出来，如此循环

<br/>

```java
public static void main(String[] args) throws Exception{

  PriorityBlockingQueue<Task> q = new PriorityBlockingQueue<Task>();

  Task t1 = new Task();
  t1.setId(3);
  t1.setName("id为3");
  Task t2 = new Task();
  t2.setId(4);
  t2.setName("id为4");
  Task t3 = new Task();
  t3.setId(1);
  t3.setName("id为1");

  //return this.id > task.id ? 1 : 0;
  q.add(t1);	//3
  q.add(t2);	//4
  q.add(t3);  //1

  // 1 3 4
  System.out.println("容器：" + q);
  System.out.println(q.take().getId());
  System.out.println("容器：" + q);
  //	System.out.println(q.take().getId());
  //	System.out.println(q.take().getId());

}
```

<br/>

- (5.)DelayQueue： 带有延迟时间的Queue，其中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素，DelayQueue中的元素必须实现Delayed接口，DelayQueue是一个没有大小限制的队列，应用场景很多，比如对缓存超时的数据进行移除，任务超时处理，空闲连接的关闭等等

~~~~java
public class Wangmin implements Delayed {  

  private String name;  
  //身份证  
  private String id;  
  //截止时间  
  private long endTime;  
  //定义时间工具类
  private TimeUnit timeUnit = TimeUnit.SECONDS;

  public Wangmin(String name,String id,long endTime){  
    this.name=name;  
    this.id=id;  
    this.endTime = endTime;  
  }  

  public String getName(){  
    return this.name;  
  }  

  public String getId(){  
    return this.id;  
  }  

  /** 
     * 用来判断是否到了截止时间 
     */  
  @Override  
  public long getDelay(TimeUnit unit) { 
    //return unit.convert(endTime, TimeUnit.MILLISECONDS) - unit.convert(System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    return endTime - System.currentTimeMillis();
  }  

  /** 
     * 相互批较排序用 
     */  
  @Override  
  public int compareTo(Delayed delayed) {  
    Wangmin w = (Wangmin)delayed;  
    return this.getDelay(this.timeUnit) - w.getDelay(this.timeUnit) > 0 ? 1:0;  
  }  
}
public class WangBa implements Runnable {  

  private DelayQueue<Wangmin> queue = new DelayQueue<Wangmin>();  

  public boolean yinye =true;  

  public void shangji(String name,String id,int money){  
    Wangmin man = new Wangmin(name, id, 1000 * money + System.currentTimeMillis());  
    System.out.println("网名"+man.getName()+" 身份证"+man.getId()+"交钱"+money+"块,开始上机...");  
    this.queue.add(man);  
  }  

  public void xiaji(Wangmin man){  
    System.out.println("网名"+man.getName()+" 身份证"+man.getId()+"时间到下机...");  
  }  

  @Override  
  public void run() {  
    while(yinye){  
      try {  
        Wangmin man = queue.take();  
        xiaji(man);  
      } catch (InterruptedException e) {  
        e.printStackTrace();  
      }  
    }  
  }  

  public static void main(String args[]){  
    try{  
      System.out.println("网吧开始营业");  
      WangBa siyu = new WangBa();  
      Thread shangwang = new Thread(siyu);  
      shangwang.start();  

      siyu.shangji("路人甲", "123", 1);  
      siyu.shangji("路人乙", "234", 10);  
      siyu.shangji("路人丙", "345", 5);  
    }  
    catch(Exception e){  
      e.printStackTrace();
    }  

  }  
}
~~~~

