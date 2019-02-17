---
title: 多线程的设计模式之Master-Worker模式
categories:
  - 设计模式
abbrlink: c319249
date: 2017-07-28 13:38:03
---

## Master-Worker模式

- 简易图：

{% asset_img Master-Worker设计模式.png %}

- Master-Worker模式是常用的并行计算的模式，它的核心思想是系统由两类进程协助工作，Master进程和Worker进程，Master负责接收和分配任务，Worker负责处理子任务，当各个Worker子进程处理完成后，会将结果返回给Master，由Master做归纳和总结，其好处就是能将一个大任务分解成若干个小任务，并行执行，从而提高系统的吞吐量
- 设计思想
- Master必须要有一个容器乘装所有的任务，因为Client要提交所有的任务到Master，这个容器要不一定要是阻塞的，但是要是一个队列，先进来的任务先处理，所以可以采用ConcurrentLinkedQueue
- Worker必要是要实现Runnable接口，因为每一个Worker都要去启动一个线程来执行任务
- Master要有一个容器乘装所有的Worker(线程)，因为Master只要Execture，所有的Worker就会执行，但是这个容器不会并发访问，所以可以用一个HashMap<String，Thread>
- Worker对象需要有Master的ConcurrentLinkedQueue的引用，因为要从这个队列中获取任务
- Master需要一个容器来乘装每一个Worker并发处理的任务的结果集，这个容器容器需要并发的被Worker访问，所以需要采用ConcurrentHashMap<String，Object>
- Worker对象需要有Master的ConcurrentHashMap的引用，因为处理完的任务的结果集需要放入其中， -实现代码

```java
public class Main {

  public static void main(String[] args) {

    Master master = new Master(new Worker(), Runtime.getRuntime().availableProcessors());

    Random random = new Random();

    for(int i = 1; i <= 100; i++){
      Task task = new Task();
      task.setId(i);
      task.setName("任务 " + i);
      task.setPrice(random.nextInt(1000));

      master.submit(task);
    }
    master.execute();

    long start = System.currentTimeMillis();

    while(true){
      if(master.isComplete()){
        long end = System.currentTimeMillis() - start;
        int result = master.getResult();
        System.out.println("任务全部执行完成，结果是" +result + "总共耗时" +end);
        break;
      }
    }
  }
}
```

```java
public class Master {

  //1.首先要有一个容器乘装所有的任务
  private ConcurrentLinkedQueue<Task> taskQueue = new ConcurrentLinkedQueue<>();

  //2.要有一个容器乘装所有的Worker，即Thread
  private HashMap<String, Thread> workers = new HashMap<>();

  //3.要有一个乘装所有Worker执行结果集的容器，
  private ConcurrentHashMap<String, Object> ResultMap = new ConcurrentHashMap<>();

  //4.  有一个构造方法
  // 注意这里有一点，我一直以为传进来的应该是workers，即总共的workers的数量，然后去遍历的启动这些worker，但是这种理解是错误的，
  //worker只是一种角色，真正执行任务的是这些Thread，所以只需要把这个Worker角色传进来即可
  public Master(Worker worker, int workerCount) {

    //worker中要有一个存放所有任务的引用
    worker.setTaskQueue(taskQueue);
    //还要有一个乘装所有结果集的引用
    worker.setResultMap(ResultMap);

    for(int i = 0; i < workerCount; i++){
      workers.put("子线程 " + Integer.toString(i), new Thread(worker));
    }
  }

  //5，有一个提交任务的方法
  public void submit(Task task){
    taskQueue.add(task);
  }

  //6.要有一起启动的方法
  public void execute(){

    for(Entry<String, Thread> thread: workers.entrySet()){
      thread.getValue().start();
    }
  }

  //7.判断是否执行完毕
  public boolean isComplete() {
    for(Entry<String, Thread> thread: workers.entrySet()){
      if(thread.getValue().getState() != Thread.State.TERMINATED){
        return false;
      }
    }
    return true;
  }

  //8.获取数据
  public int getResult(){

    int res = 0;
    for(Entry<String, Object> map: ResultMap.entrySet()){
      res += (Integer)map.getValue();
    }

    return res;
  }
}
```

~~~~java
public class Worker implements Runnable{

  private ConcurrentHashMap<String, Object> resultMap;

  private ConcurrentLinkedQueue<Task> taskQueue;

  public void setResultMap(ConcurrentHashMap<String, Object> resultMap) {
    this.resultMap = resultMap;
  }

  public void setTaskQueue(ConcurrentLinkedQueue<Task> taskQueue) {
    this.taskQueue = taskQueue;
  }

  @Override
  public void run() {
    while(true){
      Task input = taskQueue.poll();
      if(input == null) break;
      Object result = MyWorker.handle(input);

      this.resultMap.put(Integer.toString(input.getId()), result);
    }
  }

  //数据处理的方法
  private static Object handle(Task input) {
    return null;
  }

}
public class MyWorker extends Worker{

  //数据处理的方法
  static Object handle(Task input) {
    Object output = null;
    try {
      //模拟数据的处理时间
      Thread.sleep(500);
      output = input.getPrice();
    } catch (InterruptedException e) {
      e.printStackTrace();
    }

    return output;
  }
}
public class Task {

  private int id;
  private String name;
  private int price;
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
  public int getPrice() {
    return price;
  }
  public void setPrice(int price) {
    this.price = price;
  }

}
~~~~

