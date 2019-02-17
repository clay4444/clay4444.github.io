---
title: Flink的Window与Time
tags:
  - Flink
abbrlink: 827d698b
date: 2018-10-06 15:54:24
---

### Window类型

<br/>

#### Tumbling Window  

```scala
@PublicEvolving
public class TumblingEventTimeWindows extends WindowAssigner<Object, TimeWindow> {
    private static final long serialVersionUID = 1L;
    ...
    @Override
    public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
        if (timestamp > Long.MIN_VALUE) {
            // Long.MIN_VALUE is currently assigned when no timestamp is present
            long start = TimeWindow.getWindowStartWithOffset(timestamp, offset, size);
            return Collections.singletonList(new TimeWindow(start, start + size));
        } else {
            throw new RuntimeException("Record has Long.MIN_VALUE timestamp (= no timestamp marker). " +
                                       "Is the time characteristic set to 'ProcessingTime', or did you forget to call " +
                                       "'DataStream.assignTimestampsAndWatermarks(...)'?");
        }
    }
}
```

<br/>

#### Sliding Window

```scala
@PublicEvolving
public class SlidingEventTimeWindows extends WindowAssigner<Object, TimeWindow> {
    private static final long serialVersionUID = 1L;
    ...
    @Override
    public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
        if (timestamp > Long.MIN_VALUE) {
            List<TimeWindow> windows = new ArrayList<>((int) (size / slide));
            long lastStart = TimeWindow.getWindowStartWithOffset(timestamp, offset, slide);
            for (long start = lastStart;
                 start > timestamp - size;
                 start -= slide) {
                windows.add(new TimeWindow(start, start + size));
            }
            return windows;
        } else {
            throw new RuntimeException("Record has Long.MIN_VALUE timestamp (= no timestamp marker). " +
                                       "Is the time characteristic set to 'ProcessingTime', or did you forget to call " +
                                       "'DataStream.assignTimestampsAndWatermarks(...)'?");
        }
    }
}
```

<br/>

#### SessionWindow

由一系列事件组和一个指定时间长度的timeout间隙组成，类似于web应用的session，也就是一段时间没有接收到新数据就会生成新的窗口

时间无对齐

使用于线上用户行为分析

```scala
public class ProcessingTimeSessionWindows extends MergingWindowAssigner<Object, TimeWindow> {
    private static final long serialVersionUID = 1L;
    ...
    @Override
    public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
        long currentProcessingTime = context.getCurrentProcessingTime();
        return Collections.singletonList(new TimeWindow(currentProcessingTime, currentProcessingTime + sessionTimeout));
    }
}
```

<br/>

### Window api 定义

```scala
stream
.keyby(...)
.window(...)
[.trigger(..)]
[.evictor(..)]
[.allowedLateness]
.reduce/fold/apply()
```

<br/>

### 预定义的KeyedWindow

<br/>

#### Tumbling time window

.timeWindow(Time.seconds(30))

<br/>

#### Sliding time window

.timewindow(Time.seconds(30),Time.seconds(10))

<br/>

#### Tumbling count window

.countWindow(1000)

<br/>

#### Sliding count window

.countWindow(1000,10)

<br/>

#### Session Window

.window(SessionWindows.withGap(Time.minutes(10)))

<br/>

### windows  聚合分类

<br/>

#### 全量聚合

等属于窗口的数据到齐，才开始进行聚合计算

.apply(windowFunction)

.process(processWindowFunction)  1.3 之后新增加的，基本替代了apply，因为可以获取到context

适合场景：求九分位，排序等

{% asset_img 1.png %}

<br/>

```scala
public abstract class Context implements java.io.Serializable {
    /**
		 * Returns the window that is being evaluated.
		 */
    public abstract W window();

    /** Returns the current processing time. */
    public abstract long currentProcessingTime();

    /** Returns the current event-time watermark. */
    public abstract long currentWatermark();

    /**
		 * State accessor for per-key and per-window state.
		 *
		 * <p><b>NOTE:</b>If you use per-window state you have to ensure that you clean it up
		 * by implementing {@link ProcessWindowFunction#clear(Context)}.
		 */
    public abstract KeyedStateStore windowState();		//状态存储

    /**
		 * State accessor for per-key global state.
		 */
    public abstract KeyedStateStore globalState();		//状态存储

    /**
		 * Emits a record to the side output identified by the {@link OutputTag}.
		 *
		 * @param outputTag the {@code OutputTag} that identifies the side output to emit to.
		 * @param value The record to emit.
		 */
    public abstract <X> void output(OutputTag<X> outputTag, X value);
}
```

通过context可以获取带key的状态存储，本身window的状态管理都是flink的引擎在做的，现在有了processfunction，就可以通过windowState() 注册一些状态管理需要保存的数据，更可控，操作性更强，但是需要注意的是状态需要做清理，如果不清理，backend很可能会oom

<br/>

状态变化过程

{% asset_img 2.png %}

<br/>

#### 增量聚合

窗口每进入一条，就进行一次计算

reduce(reduceFunction)

fold

Aggregate(aggregateFunctiomin)

sum(key),min(key),max(key)

sumBy(key),minBy(key),maxBy(key)

<br/>

状态变化过程

{% asset_img 3.png %}

<br/>

<br/>

### 全量和增量的底层实现

#### RocksDB   Reducing State 实现

第一次插入，直接put进去一条

如果有key对应的value，取出来做一次计算，产生新的值替换老的值

{% asset_img 4.png %}

<br/>

#### RocksDB   List State 实现

比较简单，直接插入，存储的是全部的值

{% asset_img 5.png %}



<br/>

### EventTime & WaterMark

问题：使用eventtime怎么处理乱序数据

<br/>

watermark（水位线）

- 参考谷歌的DataFow
- 是eventtime处理进度的标志
- 表示比eventtime更早的事件都已经到达 ( 没有比水位线更低的数据 )
- 基于watermark来进行窗口的触发计算的判断

<br/>

### watermark 生成方式

1. timestamp
2. watermark generator

<br/>

提取时间戳和生成generator可以在source处开始的任意一个阶段，如果指定多次，后面指定的会覆盖前面的

<br/>

### watermark 两种生成方式

#### periodic Watermarks

1. 基于timer
2. ExecutionConfig.setAutoWatermarkInterval ( mesc )  默认是100ms，设置watermark 发送的周期
3. 实现AssignerWithPeriodicWatermarks接口

<br/>

#### puncuated  Watermarks

1. 基于某些事件触发watermark的生成和发送（用户代码实现）
2. 实现PunctuatedAssigner 接口
3. 也就是说在纤细中看到某个标识了，就确定，之前的数据已经到齐了，可以通过这种方式来生成

<br/>

#### periodic Watermarks 设置实例

```scala
/**
  * watermark
  */
class BoundedLatenessWatermarkAssigner(maxOutOfOrder: Int) extends AssignerWithPeriodicWatermarks[Map[String, String]] {
  private var maxTimestamp = -1L
  private var pattern = "yyyy-MM-dd HH:mm:ss"

  // 当前所有事件中的最大值减去maxOutOfOrder(最大延迟时间),就是水位线，也就是说再等maxOutOfOrder时间的延迟数据
  override def getCurrentWatermark: Watermark = {   //默认是200ms
    new Watermark(maxTimestamp - maxOutOfOrder * 1000L)
  }

  override def extractTimestamp(map: Map[String, String], l: Long): Long = {

    val dateTimeFormatter = DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss")
    val dateTime = dateTimeFormatter.parseDateTime(map("__executeTime"))

    val timestamp = dateTime.getMillis

    if (timestamp > maxTimestamp) {
      maxTimestamp = timestamp
    }
    timestamp
  }
}
```



<br/>

### 延迟数据的处理方式

1. allowedLateness（）
2. SideOutputTag()

<br/>

第一种：所能接收的最大延迟时间，延缓窗口内置状态清理时间

第二种：提供了延迟数据获取的一种方式，这样就不会丢弃数据了（另外使用一种通道，专门接收你的延迟数据）

```java
final OutputTag<T> lateOutputTag = new OutputTag<T>("late-data"){}

DataStream<T> input = ...
   
SingleOutputStreamOperator<T> result = input
.keyBY(..)
.window(...)
.allowLateness(...)
.sideOutputLateData(lateOutputTag)
.window Transform

//最后可以拿到延迟数据，进行后续的分析
DataStream<T> lateStream = result.getSideOutput(lateOutputTag)
```



<br/>

### watermark 例子

代码片段

```java
public static void main(String[] args) throws Exception {

    String hostName = "localhost";
    Integer port = 8000;

    // set up the execution environment
    final StreamExecutionEnvironment env = StreamExecutionEnvironment
        .createLocalEnvironment().setParallelism(1);

    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

    // get input data
    DataStream<String> text = env.socketTextStream(hostName, port);

    DataStream<Tuple2<String, String>> counts =
        text.flatMap(new LineSplitter()).setParallelism(1)
        //为这条流指定watermark，对乱序进行处理，
        .assignTimestampsAndWatermarks(new BoundedWaterMark(60))
        .keyBy(new KeySelector<HashMap<String,String>, Object>() {
            @Override
            public Object getKey(HashMap<String, String> map) throws Exception {
                return map.get("uid");
            }
        })
        .window(TumblingEventTimeWindows.of(Time.seconds(10)))  //翻滚窗口
        //                              .allowedLateness(Time.milliseconds(10000))
        .apply(         //apply 方法能够拿到这个窗口中的所有的元素。
        new WindowFunction<HashMap<String,String>, Tuple2<String,String>, Object, TimeWindow>() {
            @Override
            public void apply(Object o, TimeWindow timeWindow, Iterable<HashMap<String, String>> iterable, Collector<Tuple2<String, String>> collector) throws Exception {
                System.out.println("this window is fired:" + timeWindow);
                for(HashMap map : iterable){
                    System.out.println("this fired has element:" + map.get("timestamp"));
                }
            }
        }).setParallelism(1);

    // execute program
    env.execute("Window example");
}
```

<br/>

```java
/**
 * 自定义的周期性的watermarker 生成方式，每隔一段时间，就往下游发送一次。
 */
public class BoundedWaterMark extends BoundedOutOfOrdernessTimestampExtractor<HashMap<String, String>> {

    private final Logger LOG = LoggerFactory.getLogger(getClass());

    public BoundedWaterMark(long maxOutOfOrder){   //这个参数的意思就是说，接受的数据延迟的时间最多是这个时间间隔，超过就丢弃
        super(Time.seconds(maxOutOfOrder));
    }

    @Override
    public long extractTimestamp(HashMap<String, String> data) {
        SimpleDateFormat format = new SimpleDateFormat( "yyyy-MM-dd HH:mm:ss" );
        String d = format.format(Long.valueOf(data.get("timestamp")));
        LOG.info("timestamp:" + data.get("timestamp") + " date:" + d);
        LOG.info("last watermark:" + super.getCurrentWatermark().getTimestamp() + " date: " + format.format(super.getCurrentWatermark().getTimestamp()));

        return Long.valueOf(data.get("timestamp"));
    }
}

```

<br/>

执行结果

{% asset_img 6.png %}

 <br/>



### allowLatency 的表现形式

watermark：本质是强制等多长时间

allowLatency：本质是延缓了数据的清除时间

<br/>

对比执行结果如下：

{% asset_img 7.png %}



<br/>

### window内部实现原理

**window中可能有两种元素：StreamRecod和Watermark**

<br/>

#### StreamRecord

首先设置该operator的key为当前元素

根据element所携带的时间戳 ( process time 或者 event time )，分配元素应该属于的窗口，一个元素可能会隶属于多个窗口，比如slideWindowAssigner

如果这个窗口是一个可merge窗口，例如session Window，因为session Window的窗口不是预定好的，而是一个很随机的值，一条数据进来的窗口可能和上一次产生的窗口有重叠，此时处理的代价就会高很多，比如状态存储，merge之后就可以节省很多的空间，而且可以减少很多不必要的操作，

merge会进行和原有窗口的合并和状态的更新，session Windos继承了MergingWindowAssigner，所以是可merge的window

<br/>

窗口merge的原理

{% asset_img 8.png %}

<br/>

#### watermark

<br/>

{% asset_img 9.png %}



<br/>

### window  遇到的一些问题

#### watermark 不更新

- 数据源问题
- partition问题(broadlatest, forward)

<br/>

#### window出现数据倾斜

- 加盐打散

<br/>

#### 数据计算准确度如何监控

- curretnLowWatermark

<br/>

#### 数据丢弃率过高，如何调优

- 丢弃率指标   skipNumberOfRecords
- maxOutOfOrder ( 数据准确度和延迟之间的权衡 )
- 慎用allowlatency ( 数据重复问题 )

<br/>

#### 任务启动前几个触发窗口数据不完整 (  大数据、监控等场景下，配合kafka的消费逻辑会出现的问题 )

- 对数据完整性要求不是很高，但是要求实时性，从kafka latest offset 开始消费，应用 10:45 启动，window 10分，第一个窗口 10:40 - 10:50 ，此时第一个窗口就会有数据不完整的问题，处理方式是自定义window function 来丢弃一些数据，或者改写window operator 的触发器来丢弃刚刚启动的数据