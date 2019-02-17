---
title: Flink的Metrics与监控
tags:
  - Flink
abbrlink: dfa909ab
date: 2018-10-07 11:55:03
---

### Metrics的种类

#### Counter

计数器

<br/>

#### Gauge

直接赋值

<br/>

#### Histogram

会存一部分历史数据，然后取一段时间内的中位数等操作

<br/>

#### Meter

时间相关，一段时间内的比率值，速率，每秒的数据流入量啊，等等

<br/>

### 按照作用划分

> MetricOption包下包含所有的metrics指标

JobManager

JobManager Job

TaskManager

TaskManager Job

Task

Operator

<br/>

{% asset_img 1.png %}

<br/>

### Metrics 的获取方式

MetricsReporter

Restful api

<br/>

### MetricsReporter

1. jmx
2. Ganglia
3. Graphite

<br/>

#### 自定义reporter

```java
public class Log4JReporter implements MetricReporter, Scheduled,Serializable {

    private static final long serialVersionUID = -6849794470754667710L;

    private static Logger LOG = LoggerFactory.getLogger(Log4JReporter.class);

    private final Map<Counter, String> counters = new HashMap();
    private final Map<Gauge<?>, String> gauges = new HashMap();
    private final Map<Histogram, String> histograms = new HashMap();
    private final Map<Meter, String> meters = new HashMap();

    @Override
    public void open(MetricConfig metricConfig) {
    }

    @Override
    public void close() {
    }

    @Override
    public void notifyOfAddedMetric(Metric metric, String metricName, MetricGroup group) {
        String name = group.getMetricIdentifier(metricName);
        synchronized (this) {
            if (metric instanceof Counter) {
                counters.put((Counter) metric, name);
            } else if (metric instanceof Gauge<?>) {
                gauges.put((Gauge<?>) metric, name);
            } else if (metric instanceof Meter) {
                meters.put((Meter) metric, name);
            } else if (metric instanceof Histogram) {
                histograms.put((Histogram) metric, name);
            }
        }
    }

    @Override
    public void notifyOfRemovedMetric(Metric metric, String metricName, MetricGroup group) {
        synchronized (this) {
            if (metric instanceof Counter) {
                counters.remove(metric);
            } else if (metric instanceof Gauge<?>) {
                this.gauges.remove(metric);
            } else if (metric instanceof Histogram) {
                this.histograms.remove(metric);
            } else if (metric instanceof Meter) {
                this.meters.remove(metric);
            }
        }
    }

    @Override
    public void report() {
        LOG.info("========= Starting metric report, T={} =========", System.currentTimeMillis());
        System.out.println("========= Starting metric report, T={ "+ System.currentTimeMillis()+" } =========");
        for (Map.Entry<Counter, String> metric : counters.entrySet()) {
            LOG.info("{}: {}", metric.getValue(), metric.getKey());
            System.out.println(">>>>>counter!!>>>>>>"+metric.getValue() + ">>>>>>>>>>>" + metric.getKey());
        }
        for (Map.Entry<Gauge<?>, String> metric : gauges.entrySet()) {
            LOG.info("{}: {}", metric.getValue(), metric.getKey().getValue().toString());
            System.out.println(">>>>>gauges!!!>>>>>>"+metric.getValue() + ">>>>>>>>>>>" + metric.getKey().getValue().toString());
        }
        for (Map.Entry<Meter, String> metric : meters.entrySet()) {
            //            LOG.info("{}: {}", metric.getValue(), metric.getKey().getRate());
            //            System.out.println(">>>>>>>>>>>"+metric.getValue() + ">>>>>>>>>>>" + metric.getKey().getRate());
        }
        for (Map.Entry<Histogram, String> metric : histograms.entrySet()) {
            //            HistogramStatistics stats = metric.getKey().getStatistics();
            //            LOG.info("{}: count:{} min:{} max:{} mean:{} stddev:{} p50:{} p75:{} p95:{} p98:{} p99:{} p999:{}",
            //                    metric.getValue(), stats.size(), stats.getMin(), stats.getMax(), stats.getMean(), stats.getStdDev(),
            //                    stats.getQuantile(0.50), stats.getQuantile(0.75), stats.getQuantile(0.95),
            //                    stats.getQuantile(0.98), stats.getQuantile(0.99), stats.getQuantile(0.999));
        }
        LOG.info("========= Finished metric report =========");
        System.out.println("========= Finished metric report =========");
    }
}
```

<br/>

### 用户自定义指标上报

UDF中定义

```java
public class ChooseEvenNumber {

    public static void main(String[] args) throws Exception {
        final StreamExecutionEnvironment env = StreamExecutionEnvironment
            .getExecutionEnvironment().setParallelism(1);

        env.addSource(new LongSource())
            .setParallelism(2)
            .filter(new RichFilterFunction<Integer>() {
                private Counter filterOutNumber;
                @Override
                public void open(Configuration parameters) throws Exception {
                    filterOutNumber = getRuntimeContext()
                        .getMetricGroup()
                        .addGroup("filter")
                        .counter("filterOutNumber");
                }

                @Override
                public boolean filter(Integer integer) throws Exception {
                    if (integer % 2 == 0) {
                        filterOutNumber.inc();
                    }
                    return !(integer % 2 == 0);
                }
            }).print();

        env.execute();
    }
}
```

<br/>

#### 社区重要指标含义解释

系统类指标

{% asset_img 2.png %}

{% asset_img 3.png %}

<br/>

checkpoint 相关指标

{% asset_img 4.png %}

<br/>

流量相关指标

{% asset_img 5.png %}

<br/>

<br/>

### 线上业务监控大盘

{% asset_img 6.png %}

{% asset_img 7.png %}

<br/>

<br/>

### flink作业的延时监控

<br/>

#### 端到端(source到sink)的延迟监控怎么做？

<br/>

1. source定时向下游发送LatencyMarker（也就是说latencyMarker也是一个元素）
2. LatencyMarker向下游发送的过程中不会超越records（不会超车，那么LatencyMarker从source到sink话费的时间就等于端到端的延迟时间）
3. LatencyMarker不会记录operator算子的计算时长（window操作10分钟，LatencyMarker不会持续10分钟）
4. 每个算子维护一份Latency列表用于衡量延迟（每个operator都会上报一个allowLatency指标，这个指标就是指的当前source到operator的平均延迟大小）
5. 整体延迟500ms以下

<br/>

#### 具体代码实现

发送LatencyMarker

{% asset_img 8.png %}

<br/>

StreamInputProcessor 处理LatencyMarker

{% asset_img 9.png %}

<br/>

{% asset_img 10.png %}

<br/>

{% asset_img 11.png %}

<br/>

<br/>

### Flink 作业中的反压机制

<br/>

{% asset_img 12.png %}

<br/>

{% asset_img 13.png %}

<br/>

### 反压容易带来的问题

<br/>

1. 数据延迟 / 窗口数据丢弃
2. 堆外内存溢出
3. checkpoint不断失败

<br/>

#### 解决： 

1. 找到DAG中的瓶颈点，细粒度设置并发调优
2. 数据均衡打散

<br/>

### Flink Metrics与druid的联合使用案例

<br/>

{% asset_img 14.png %}

<br/>

{% asset_img 15.png %}

<br/>

{% asset_img 16.png %}