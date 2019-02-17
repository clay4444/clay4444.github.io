---
title: Flink状态管理与恢复机制
tags:
  - Flink
abbrlink: 6a73b0f6
date: 2018-10-06 21:43:18
---

### 状态 （state）

#### 定义

某task / operator 在某时刻的一个中间结果

快照

#### 作用

State可以被记录，在失败的情况下恢复

#### 基本类型

- Operator state
- keyed state

<br/>

### State 类型

<br/>

#### Operator State

- 唯一绑定到特定operator，对应一个算子，
- 与key无关
- 无论怎么配置backend，都是存储在java堆中，
- 数据结构 ： ListState\<T>
- 代码实例

```java
/*
 *   假设事件场景为某业务事件流中有事件 1、2、3、4、5、6、7、8、9 ......
 *
 *   现在，想知道两次事件1之间，一共发生了多少次其他的事件，分别是什么事件，然后输出相应结果。
 *   如下:
 *    事件流 1 2 3 4 5 1 3 4 5 6 7 1 4 5 3 9 9 2 1 ....
 *    输出  4     2 3 4 5
 *          5     3 4 5 6 7
 *          6     4 5 3 9 9 2
 *
 * 实现的是at least once 的语义
 */
public class CountWithOperatorState extends RichFlatMapFunction<Long,String> implements CheckpointedFunction {
    /*
     *  保存结果状态
     */
    private transient ListState<Long>  checkPointCountList;
    private List<Long> listBufferElements;

    @Override
    public void flatMap(Long r, Collector<String> collector) throws Exception {
        if (r == 1) {
            if (listBufferElements.size() > 0) {
                StringBuffer buffer = new StringBuffer();
                for(int i = 0 ; i < listBufferElements.size(); i ++) {
                    buffer.append(listBufferElements.get(i) + " ");
                }
                collector.collect(buffer.toString());
                listBufferElements.clear();
            }
        } else {
            listBufferElements.add(r);
        }
    }

    //checkpoint的时候调用，
    @Override
    public void snapshotState(FunctionSnapshotContext functionSnapshotContext) throws Exception {
        checkPointCountList.clear();
        for (int i = 0 ; i < listBufferElements.size(); i ++) {
            checkPointCountList.add(listBufferElements.get(i));
        }
    }

    @Override
    public void initializeState(FunctionInitializationContext functionInitializationContext) throws Exception {

        //flink的状态的数据结构是由flink自己的runtime维护的，这里就相当于new了一个arraylist，
        ListStateDescriptor<Long> listStateDescriptor =
            new ListStateDescriptor<Long>(
            "listForThree",
            TypeInformation.of(new TypeHint<Long>() {}));

        //相当于new了一块内存，
        checkPointCountList = functionInitializationContext.getOperatorStateStore().getListState(listStateDescriptor);

        if (functionInitializationContext.isRestored()) {
            for (Long element : checkPointCountList.size()) {
                listBufferElements.add(element);
            }
        }
    }
}
```

<br/>

#### Keyed State

<br/>

- 基于KeyStream之上的状态，datastream.keyBy()
- keyBy() 之后的 Operator State
- Organized by keyGroups
- 数据结构
  1. ValueState\<T>  ： update
  2. ListState\<T>：add / get clear
  3. ReduceState\<T>： add / reduceFunction
  4. MapState\<UK, UV>：put(UK, UV) / putAll(UK, UV) / get(UK)
- 代码实例

```java
/*
 * without implements checkpoint
 *
 * FlatMapFunction 为无状态函数
 *
 * RichFlatMapFunction 为有状态函数，因为可以获取getRuntimeContext().getstate
 */
public class CountWithKeyedState extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // access the state value
        Tuple2<Long, Long> currentSum = sum.value();   //过了三个数字之后，在这里把 f0 重新置为0的，

        // update the count
        currentSum.f0 += 1;

        // add the second field of the input value
        currentSum.f1 += input.f1;

        // update the state
        sum.update(currentSum);     //保存状态，

        // if the count reaches 2, emit the average and clear the state
        if (currentSum.f0 >= 3) {
            out.collect(new Tuple2<Long,Long>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();   //发射出去之后，就清除状态，
        }
    }


    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
            new ValueStateDescriptor<Tuple2<Long, Long>>(
            "average", // the state name
            TypeInformation.of(new TypeHint<Tuple2<Long, Long>>(){})); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}
```

<br/>

<br/>

### RePartition Key State

- 将key分为group
- 每个key分配到唯一的group
- 将group分配给task
- Key Group 由最大并行度的大小所决定
- [ K，G ] 取值方式
  1. hash = hash(key)
  2. KG = hash % numberOfKeyGroup
  3. Subtask  = KG * parallelism / numberOfKeyGroup ： SubtaskId 就代表当前的key应该被哪一个subtask加载
- 扩容，缩容之后重启需要恢复的数据会存储到另外一个subtask上去计算，但是又不希望恢复之后所在的task更分散，所以使用了如此计算，如果直接使用hash%parallelism，数据会变得太分散，也是引入keygroup的原因，但是总会有一个key的数据会发生数据迁移，到其他的subtask上，这个过程就是**rescale**

<br/>

{% asset_img 1.png %}

<br/>

### Recale 重新分配

{% asset_img 2.png %}

<br/>

<br/>

### 状态容错

- 依赖checkpoint机制
- 保证exactly-once ( 只能保证flink系统内部，对于sink和source需要依赖的外部组件一同保证 )

{% asset_img 3.png %}

<br/>

<br/>

### checkpointing

<br/>

#### 概念

- 全局快照，持久化保存所有的task / operator的State
- 序列化数据集合

#### 特点

- 异步：jobmanager发送checkpoint消息，所有jm触发checkpoint操作，保存到backend，最后回复ack消息这些都是同步的，写入到hdfs时是异步的
- 全量 / 增量：一般情况下都是每次都是全量，只有在状态存储非常大的时候才是增量
- Barrier机制
- 失败情况下可回滚致最近一次的checkpoint
- 周期性

<br/>

### checkpointing 参数

- 默认checkpointing是disable的
- checkpointing开启后，默认的checkpointMode是exactly-once
- checkpointing的checkpointMode有两种，一种是AT_LEAST_ONCE，另一种是EXACTLY_ONCE，两种的实现原理不一样，区别在于checkpointing的时候是否需要对齐
- checkpoint周期
- checkpointMode
- minPauseBetweenCheckpoints（checkpoint最小间隔）
- CheckpointTimeout （checkpoint的超时时间）

<br/>

### barrier的作用

- 异步化全局snapshot
- source可以继续记录的offset
- Operator可以check状态
- sink可以事务的提交 add by 1.4.0 **twoPhaseCommitSinkFunction**
- 隔一段时间，在数据流上插入一个屏障，当这个屏障到当前算子的时候，前面所有的operator都流到了当前的算子， 此时可以认为前面所有的算子的数据都已经到齐了，此时当前算子就可以认为，它的数据也已经收集完成，此时就可以进行操作，然后处理快照，

<br/>

{% asset_img 4.png %}

<br/>

<br/>

### checkpoint 的过程

1. jobmanager 发送Snapshot消息到source0
2. source0收到消息以后确认收到消息
3. source0向flatmap0，flatmap1 发送barrier
4. source0执行snapshot，发送成功消息致JobManager
5. flatmap0和flatmap1收齐所有的barrier，并立刻分别向sink0, sink1 发送barrier
6. 然后flatmap0，flatmap1执行snapshot，并发送成功确认消息致jobmanager
7. 5个task都打成功了，就可以在flink ui上看到checkpoint成功了

<br/>

{% asset_img 5.png %}

<br/>

### Restart strategies

#### None Restart

<br/>

#### Fixed Delay

- Flink-conf.yml 

  Restart-strategy.fixed-delay.attempts:3

  Restart-strategy.fixed-delay.delay:10s

- 应用代码设置

  ```java
  env.setRestartStrategy(RestartStrategy.fixedelayRestart(3,Time.of(10,TimeUnit,Seconds)))
  ```

<br/>

#### Failure Rate

<br/>

### savePoint

#### 概念

- 让时光倒流的语法糖
- 全局，一致性快照，如数据源offset，并行操作状态
- 可以从应用在过去的任意做了savepoint的时刻开始继续消费

<br/>

#### 作用

- 开发新版本
- 业务迁移，集群需要迁移，不容许数据丢失

<br/>

### savePoint vs checkpoint

#### checkpoint

应用定时触发

内部应用restart时使用

<br/>

#### savePoint

用户在特定时刻手动触发

checkpoint的软链

在升级等情况使用

<br/>

<br/>

### savePoint 触发方式

#### 直接触发方式

```shell
bin/flink savepoint:jobId[:targetDirectory]
```

<br/>

#### canle with savePoint

```shell
bin/flink canle -s [:targetDirectory]:jobId
```

<br/>

### savePoint

#### 获取jobId

- 命令

  ```shell
  bin/flink list -r -yid application_id
  ```

- flinkUI

{% asset_img 6.png %}

<br/>

#### 获取savepoint url

{% asset_img 7.png %}