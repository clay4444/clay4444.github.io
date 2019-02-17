---
title: MR之自定义GroupingComparator
categories:
  - big-data
tags:
  - MapReduce
abbrlink: af70e59d
date: 2018-02-12 18:22:21
---

### 需求

有如下订单数据

| 订单id          | 商品id   | 成交金额  |
| ------------- | ------ | ----- |
| Order_0000001 | Pdt_01 | 222.8 |
| Order_0000001 | Pdt_05 | 25.8  |
| Order_0000002 | Pdt_03 | 522.8 |
| Order_0000002 | Pdt_04 | 122.4 |
| Order_0000003 | Pdt_01 | 222.8 |

现在需要求出每一个订单中成交金额最大的一笔交易

<br/>

### 分析

1、利用“订单id和成交金额”作为key，可以将map阶段读取到的所有订单数据按照id分区，按照金额排序，发送到reduce

2、在reduce端利用groupingcomparator将订单id相同的kv聚合成组，然后取第一个即是最大值

<br/>

### 实现

#### ItemidGroupingComparator

```java
/**
 * 利用reduce端的GroupingComparator来实现将一组bean看成相同的key
 * @author clay
 *
 */
public class ItemidGroupingComparator extends WritableComparator {

  //传入作为key的bean的class类型，以及制定需要让框架做反射获取实例对象
  protected ItemidGroupingComparator() {
    super(OrderBean.class, true);
  }

  @Override
  public int compare(WritableComparable a, WritableComparable b) {
    OrderBean abean = (OrderBean) a;
    OrderBean bbean = (OrderBean) b;

    //比较两个bean时，指定只比较bean中的orderid
    return abean.getItemid().compareTo(bbean.getItemid());
  }
}
```
<br/>

#### ItemIdPartitioner

~~~~java
public class ItemIdPartitioner extends Partitioner<OrderBean, NullWritable>{

    @Override
    public int getPartition(OrderBean bean, NullWritable value, int numReduceTasks) {
        //相同id的订单bean，会发往相同的partition
        //而且，产生的分区数，是会跟用户设置的reduce task数保持一致
        return (bean.getItemid().hashCode() & Integer.MAX_VALUE) % numReduceTasks;
    }
}
~~~~

<br/>

#### 定义订单信息bean 

~~~~java
/**
 * 订单信息bean，实现hadoop的序列化机制
 */
public class OrderBean implements WritableComparable<OrderBean>{
    private Text itemid;
    private DoubleWritable amount;

    public OrderBean() {
    }
    public OrderBean(Text itemid, DoubleWritable amount) {
        set(itemid, amount);
    }

    public void set(Text itemid, DoubleWritable amount) {

        this.itemid = itemid;
        this.amount = amount;

    }

    public Text getItemid() {
        return itemid;
    }

    public DoubleWritable getAmount() {
        return amount;
    }

    @Override
    public int compareTo(OrderBean o) {
        int cmp = this.itemid.compareTo(o.getItemid());
        if (cmp == 0) {

            cmp = -this.amount.compareTo(o.getAmount());
        }
        return cmp;
    }

    @Override
    public void write(DataOutput out) throws IOException {
        out.writeUTF(itemid.toString());
        out.writeDouble(amount.get());

    }

    @Override
    public void readFields(DataInput in) throws IOException {
        String readUTF = in.readUTF();
        double readDouble = in.readDouble();

        this.itemid = new Text(readUTF);
        this.amount= new DoubleWritable(readDouble);
    }


    @Override
    public String toString() {
        return itemid.toString() + "\t" + amount.get();
    }
}
~~~~

<br/>

#### mapreduce处理流程 

~~~~java
/**
 * 利用secondarysort机制输出每种item订单金额最大的记录
 * @author clay
 *
 */
public class SecondarySort {

    static class SecondarySortMapper extends Mapper<LongWritable, Text, OrderBean, NullWritable>{

        OrderBean bean = new OrderBean();

        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

            String line = value.toString();
            String[] fields = StringUtils.split(line, "\t");

            bean.set(new Text(fields[0]), new DoubleWritable(Double.parseDouble(fields[1])));

            context.write(bean, NullWritable.get());

        }

    }

    static class SecondarySortReducer extends Reducer<OrderBean, NullWritable, OrderBean, NullWritable>{


        //在设置了groupingcomparator以后，这里收到的kv数据 就是：  <1001 87.6>,null  <1001 76.5>,null  .... 
        //此时，reduce方法中的参数key就是上述kv组中的第一个kv的key：<1001 87.6>
        //要输出同一个item的所有订单中最大金额的那一个，就只要输出这个key
        @Override
        protected void reduce(OrderBean key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
            context.write(key, NullWritable.get());
        }
    }


    public static void main(String[] args) throws Exception {

        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);

        job.setJarByClass(SecondarySort.class);

        job.setMapperClass(SecondarySortMapper.class);
        job.setReducerClass(SecondarySortReducer.class);


        job.setOutputKeyClass(OrderBean.class);
        job.setOutputValueClass(NullWritable.class);

        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        //指定shuffle所使用的GroupingComparator类
        job.setGroupingComparatorClass(ItemidGroupingComparator.class);
        //指定shuffle所使用的partitioner类
        job.setPartitionerClass(ItemIdPartitioner.class);

        job.setNumReduceTasks(3);

        job.waitForCompletion(true);	
    }
}
~~~~
