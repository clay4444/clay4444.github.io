---
title: Storm的分组策略和确保消息送达机制
tags:
  - Storm
categories:
  - big-data
abbrlink: 97e35c7b
date: 2018-03-08 18:06:09
---

### 分组策略

1. shuffle 随机分组

{% asset_img s1.png %}

<br/>

1. field分组

安装指定filed的key进行hash处理，

相同的field，一定进入到同一bolt.

该分组容易产生数据倾斜问题，通过使用二次聚合避免此类问题。

{% asset_img s2.png %}

<br/>

使用二次聚合避免倾斜。

App入口类

```java
public class App {
  public static void main(String[] args) throws Exception {
    TopologyBuilder builder = new TopologyBuilder();
    //设置Spout
    builder.setSpout("wcspout", new WordCountSpout()).setNumTasks(2);
    //设置creator-Bolt
    builder.setBolt("split-bolt", new SplitBolt(),3).shuffleGrouping("wcspout").setNumTasks(3);
    //设置counter-Bolt
    builder.setBolt("counter-1", new CountBolt(),3).shuffleGrouping("split-bolt").setNumTasks(3);
    builder.setBolt("counter-2", new CountBolt(),2).fieldsGrouping("counter-1",new Fields("word")).setNumTasks(2);

    Config conf = new Config();
    conf.setNumWorkers(2);
    conf.setDebug(true);

    /**
	* 本地模式storm
	*/
    LocalCluster cluster = new LocalCluster();
    cluster.submitTopology("wc", conf, builder.createTopology());
    //Thread.sleep(20000);
    //        StormSubmitter.submitTopology("wordcount", conf, builder.createTopology());
    //cluster.shutdown();

  }
}
```

<br/>

聚合bolt

```java
/**
 * countbolt，使用二次聚合，解决数据倾斜问题。
 * 一次聚合和二次聚合使用field分组，完成数据的最终统计。
 * 一次聚合和上次split工作使用
 */
public class CountBolt implements IRichBolt{

  private Map<String,Integer> map ;		//一个task对应一个对象(实例)，一个对象对应这个对象自己的map，

  private TopologyContext context;
  private OutputCollector collector;

  private long lastEmitTime = 0 ;		//上次清分的时间

  private long duration = 5000 ;		//清分的时间片，即多长时间进行一次清分

  public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
    this.context = context;
    this.collector = collector;
    map = new HashMap<String, Integer>();
    map = Collections.synchronizedMap(map);
    //分线程，执行清分工作,	-----线程安全问题------
    Thread t = new Thread(){
      public void run() {
        while(true){
          emitData();
        }
      }
    };
    //守护进程
    t.setDaemon(true);	//守护进程，否则就是死循环了，主线程结束，守护线程就结束
    t.start();
  }

  private void emitData(){
    //清分map
    synchronized (map){			//在清分过程中，很有可能主线程正在往其他map中添加数据。所以要把map变成线程安全的，Collections.synchronizedMap(map);
      //判断是否符合清分的条件
      for (Map.Entry<String, Integer> entry : map.entrySet()) {
        //向下一环节发送数据
        collector.emit(new Values(entry.getKey(), entry.getValue()));
      }
      //清空map		已经把数据发出去了，如果不清空，就会再原来的基础上再次进行计算，就会出现二次计算的问题。
      map.clear();
    }
    //休眠      休眠的作用是5秒清分一次
    try {
      Thread.sleep(5000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

  public void execute(Tuple tuple) {
    //提取单词
    String word = tuple.getString(0);
    Util.sendToLocalhost(this, word);
    //提取单词个数
    Integer count = tuple.getInteger(1);
    if(!map.containsKey(word)){
      map.put(word, count);	//这里要放置count，不能再放置1了，因为第二次count的时候，过来的count已经计算过了，不能直接置为1.
    }
    else{
      map.put(word,map.get(word) + count);
    }
  }

  public void cleanup() {
    for(Map.Entry<String,Integer> entry : map.entrySet()){
      System.out.println(entry.getKey() + " : " + entry.getValue());
    }
  }

  public void declareOutputFields(OutputFieldsDeclarer declarer) {
    declarer.declare(new Fields("word","count"));
  }

  public Map<String, Object> getComponentConfiguration() {
    return null;
  }
}
```



<br/>

1. all分组

使用广播分组。

```java
builder.setBolt("split-bolt", new SplitBolt(),2).allGrouping("wcspout").setNumTasks(2);
```

{% asset_img s3.png %}



<br/>

1. direct(特供)

只发送给指定的一个bolt.

```java
//a.通过emitDirect()方法发送元组
//可以通过context.getTaskToComponent()方法得到所有taskId和组件名(任务名)的映射
collector.emitDirect(taskId,new Values(line));
```

<br/>

```java
//b.指定directGrouping方式。
builder.setBolt("split-bolt", new SplitBolt(),2).directGrouping("wcspout").setNumTasks(2);
```

<br/>

1. global分组

对目标target tasked进行排序，选择最小的taskId号进行发送tuple

类似于direct,可以是特殊的direct分组。

<br/>

1. 自定义分组

自定义CustomStreamGrouping类

```java
/**
 * 自定义分组
 */
public class MyGrouping implements CustomStreamGrouping {

  //接受目标任务的id集合
  private List<Integer> targetTasks ;

  public void prepare(WorkerTopologyContext context, GlobalStreamId stream, List<Integer> targetTasks) {
    this.targetTasks = targetTasks ;
  }

  public List<Integer> chooseTasks(int taskId, List<Object> values) {
    List<Integer> subTaskIds = new ArrayList<Integer>();
    //取出一半来往后执行
    for(int i = 0 ; i <= targetTasks.size() / 2 ; i ++){
      subTaskIds.add(targetTasks.get(i));
    }
    return subTaskIds;
  }
}
```

<br/>
设置分组策略

```java
public class App {
  public static void main(String[] args) throws Exception {
    TopologyBuilder builder = new TopologyBuilder();
    //设置Spout
    builder.setSpout("wcspout", new WordCountSpout()).setNumTasks(2);
    //设置creator-Bolt
    builder.setBolt("split-bolt", new SplitBolt(),4).customGrouping("wcspout",new MyGrouping()).setNumTasks(4);

    Config conf = new Config();
    conf.setNumWorkers(2);
    conf.setDebug(true);

    /**
	 * 本地模式storm
	*/
    LocalCluster cluster = new LocalCluster();
    cluster.submitTopology("wc", conf, builder.createTopology());
    System.out.println("hello world");
  }
}
```

<br/>



### storm确保消息如何被完全处理

WordCountSpout：通过回调函数

```java
public class WordCountSpout implements IRichSpout{
  private TopologyContext context ;
  private SpoutOutputCollector collector ;

  private List<String> states ;

  private Random r = new Random();

  private int index = 0;

  //消息集合, 存放所有消息
  private Map<Long,String> messages = new HashMap<Long, String>();

  //失败消息
  private Map<Long,Integer> failMessages = new HashMap<Long, Integer>();

  public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {
    this.context = context ;
    this.collector = collector ;
    states = new ArrayList<String>();
    states.add("hello world tom");
    states.add("hello world tomas");
    states.add("hello world tomasLee");
    states.add("hello world tomson");
  }

  public void close() {

  }

  public void activate() {

  }

  public void deactivate() {

  }

  public void nextTuple() {
    if(index < 3){		//只发送3条消息作为测试
      String line = states.get(r.nextInt(4));
      //取出时间戳
      long ts = System.currentTimeMillis() ;
      messages.put(ts,line);

      //发送元组，使用ts作为消息id
      collector.emit(new Values(line),ts);
      System.out.println(this + "nextTuple() : " + line + " : " + ts);
      index ++ ;
    }
  }

  /**
     * 回调处理
     */
  public void ack(Object msgId) {
    //成功处理，删除失败重试.
    Long ts = (Long)msgId ;
    failMessages.remove(ts) ;
    messages.remove(ts) ;
  }

  public void fail(Object msgId) {
    //时间戳作为msgId
    Long ts = (Long)msgId;
    //判断消息是否重试了3次
    Integer retryCount = failMessages.get(ts);
    retryCount = (retryCount == null ? 0 : retryCount) ;

    //超过最大重试次数
    if(retryCount >= 3){
      failMessages.remove(ts) ;
      messages.remove(ts) ;
    }
    else{
      //重试
      collector.emit(new Values(messages.get(ts)),ts);
      //之所以要有一个messages来存放所有的消息，就是因为此时还要从messages中取出要重试的消息。
      System.out.println(this + "fail() : " + messages.get(ts) + " : " + ts);
      retryCount ++ ;
      failMessages.put(ts,retryCount);
    }
  }

  public void declareOutputFields(OutputFieldsDeclarer declarer) {
    declarer.declare(new Fields("line"));
  }

  public Map<String, Object> getComponentConfiguration() {
    return null;
  }
}
```

<br/>

SplitBolt:

```java
public class SplitBolt implements IRichBolt {

  private TopologyContext context ;
  private OutputCollector collector ;

  public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
    this.context = context ;
    this.collector = collector ;
  }

  public void execute(Tuple tuple) {
    String line = tuple.getString(0);
    if(new Random().nextBoolean()){
      //确认
      collector.ack(tuple);
      System.out.println(this + " : ack() : " + line + " : "+ tuple.getMessageId().toString());
    }
    else{
      //失败
      collector.fail(tuple);
      System.out.println(this + " : fail() : " + line + " : " + tuple.getMessageId().toString());
    }
  }

  public void cleanup() {

  }

  public void declareOutputFields(OutputFieldsDeclarer declarer) {
    declarer.declare(new Fields("word","count"));

  }

  public Map<String, Object> getComponentConfiguration() {
    return null;
  }
}
```

<br/>

App:

```java
public class App {
  public static void main(String[] args) throws Exception {
    TopologyBuilder builder = new TopologyBuilder();
    //设置Spout
    builder.setSpout("wcspout", new WordCountSpout()).setNumTasks(1);
    //设置creator-Bolt
    builder.setBolt("split-bolt", new SplitBolt(),2).shuffleGrouping("wcspout").setNumTasks(2);

    Config conf = new Config();
    conf.setNumWorkers(2);
    conf.setDebug(true);

    /**
     * 本地模式storm
     */
    LocalCluster cluster = new LocalCluster();
    cluster.submitTopology("wc", conf, builder.createTopology());
    System.out.println("hello world llll");
  }
}
```

<br/>

测试结果：

{% asset_img s4.png %}