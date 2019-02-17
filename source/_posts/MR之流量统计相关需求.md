---
title: MR之流量统计相关需求
categories:
  - big-data
tags:
  - MapReduce
abbrlink: cc1873ed
date: 2018-02-12 16:35:23
---

### 数据：

~~~
id，phoneNumber，upFlow，dFlow
0,  138----	   ,  12,     11
1,  134----	   ,  13,     10
2,  136----	   ,  15,     16
3,  135----	   ,  17,     11
~~~



## 需求一：统计上，下行流量

#### FlowBean

~~~java
public class FlowBean implements Writable{

  private long upFlow;
  private long dFlow;
  private long sumFlow;

  //反序列化时，需要反射调用空参构造函数，所以要显示定义一个
  public FlowBean(){}

  public FlowBean(long upFlow, long dFlow) {
    this.upFlow = upFlow;
    this.dFlow = dFlow;
    this.sumFlow = upFlow + dFlow;
  }


  public long getUpFlow() {
    return upFlow;
  }
  public void setUpFlow(long upFlow) {
    this.upFlow = upFlow;
  }
  public long getdFlow() {
    return dFlow;
  }
  public void setdFlow(long dFlow) {
    this.dFlow = dFlow;
  }


  public long getSumFlow() {
    return sumFlow;
  }


  public void setSumFlow(long sumFlow) {
    this.sumFlow = sumFlow;
  }


  /**
	 * 序列化方法
	 */
  @Override
  public void write(DataOutput out) throws IOException {
    out.writeLong(upFlow);
    out.writeLong(dFlow);
    out.writeLong(sumFlow);

  }


  /**
	 * 反序列化方法
	 * 注意：反序列化的顺序跟序列化的顺序完全一致
	 */
  @Override
  public void readFields(DataInput in) throws IOException {
    upFlow = in.readLong();
    dFlow = in.readLong();
    sumFlow = in.readLong();
  }

  @Override
  public String toString() {

    return upFlow + "\t" + dFlow + "\t" + sumFlow;
  }
}
~~~

#### FlowCount

~~~~java
public class FlowCount {

  static class FlowCountMapper extends Mapper<LongWritable, Text, Text, FlowBean>{

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

      //将一行内容转成string
      String line = value.toString();
      //切分字段
      String[] fields = line.split("\t");
      //取出手机号
      String phoneNbr = fields[1];
      //取出上行流量下行流量
      long upFlow = Long.parseLong(fields[fields.length-3]);
      long dFlow = Long.parseLong(fields[fields.length-2]);

      context.write(new Text(phoneNbr), new FlowBean(upFlow, dFlow));

    }

  }

  static class FlowCountReducer extends Reducer<Text, FlowBean, Text, FlowBean>{

    //<183323,bean1><183323,bean2><183323,bean3><183323,bean4>.......
    @Override
    protected void reduce(Text key, Iterable<FlowBean> values, Context context) throws IOException, InterruptedException {

      long sum_upFlow = 0;
      long sum_dFlow = 0;

      //遍历所有bean，将其中的上行流量，下行流量分别累加
      for(FlowBean bean: values){
        sum_upFlow += bean.getUpFlow();
        sum_dFlow += bean.getdFlow();
      }

      FlowBean resultBean = new FlowBean(sum_upFlow, sum_dFlow);
      context.write(key, resultBean);
    }
  }

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    /*conf.set("mapreduce.framework.name", "yarn");
		conf.set("yarn.resoucemanager.hostname", "mini1");*/
    Job job = Job.getInstance(conf);

    /*job.setJar("/home/hadoop/wc.jar");*/
    //指定本程序的jar包所在的本地路径
    job.setJarByClass(FlowCount.class);

    //指定本业务job要使用的mapper/Reducer业务类
    job.setMapperClass(FlowCountMapper.class);
    job.setReducerClass(FlowCountReducer.class);

    //指定mapper输出数据的kv类型
    job.setMapOutputKeyClass(Text.class);
    job.setMapOutputValueClass(FlowBean.class);

    //指定最终输出的数据的kv类型
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(FlowBean.class);

    //指定job的输入原始文件所在目录
    FileInputFormat.setInputPaths(job, new Path(args[0]));
    //指定job的输出结果所在目录
    FileOutputFormat.setOutputPath(job, new Path(args[1]));

    //将job中配置的相关参数，以及job所用的java类所在的jar包，提交给yarn去运行
    /*job.submit();*/
    boolean res = job.waitForCompletion(true);
    System.exit(res?0:1);

  }
}
~~~~



## 需求二：统计流量并按照大小倒序排序

第一个job：负责流量统计，和需求一相同

第二个job：读入第一个job的输出，然后排序

要将 flowBean 作为map的key输出，这样 mapreduce 就会自动排序。

此时 flowBean 要实现WritableComparable接口，实现其中的compareTo方法

#### FlowCountSort

~~~java
public class FlowCountSort {

  static class FlowCountSortMapper extends Mapper<LongWritable, Text, FlowBean, Text> {

    FlowBean bean = new FlowBean();
    Text v = new Text();

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

      // 拿到的是上一个统计程序的输出结果，已经是各手机号的总流量信息
      String line = value.toString();

      String[] fields = line.split("\t");

      String phoneNbr = fields[0];

      long upFlow = Long.parseLong(fields[1]);
      long dFlow = Long.parseLong(fields[2]);

      bean.set(upFlow, dFlow);
      v.set(phoneNbr);

      context.write(bean, v);
    }

  }

    /**
	 * 根据key来掉, 传过来的是对象, 每个对象都是不一样的, 所以每个对象都调用一次reduce方法
	 */
  static class FlowCountSortReducer extends Reducer<FlowBean, Text, Text, FlowBean> {

    // <bean(),phonenbr>
    @Override
    protected void reduce(FlowBean bean, Iterable<Text> values, Context context) throws IOException, InterruptedException {

      context.write(values.iterator().next(), bean);
    }

  }

  public static void main(String[] args) throws Exception {

    Configuration conf = new Configuration();
    /*conf.set("mapreduce.framework.name", "yarn");
	conf.set("yarn.resoucemanager.hostname", "mini1");*/
    Job job = Job.getInstance(conf);

    /*job.setJar("/home/hadoop/wc.jar");*/
    //指定本程序的jar包所在的本地路径
    job.setJarByClass(FlowCountSort.class);

    //指定本业务job要使用的mapper/Reducer业务类
    job.setMapperClass(FlowCountSortMapper.class);
    job.setReducerClass(FlowCountSortReducer.class);

    //指定mapper输出数据的kv类型
    job.setMapOutputKeyClass(FlowBean.class);
    job.setMapOutputValueClass(Text.class);

    //指定最终输出的数据的kv类型
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(FlowBean.class);

    //指定job的输入原始文件所在目录
    FileInputFormat.setInputPaths(job, new Path(args[0]));
    //指定job的输出结果所在目录

    Path outPath = new Path(args[1]);
    /*FileSystem fs = FileSystem.get(conf);
		if(fs.exists(outPath)){
			fs.delete(outPath, true);
		}*/
    FileOutputFormat.setOutputPath(job, outPath);

    //将job中配置的相关参数，以及job所用的java类所在的jar包，提交给yarn去运行
    /*job.submit();*/
    boolean res = job.waitForCompletion(true);
    System.exit(res?0:1);

  }
}
~~~

#### FlowBean

~~~~java
public class FlowBean implements WritableComparable<FlowBean>{

  private long upFlow;
  private long dFlow;
  private long sumFlow;

  //反序列化时，需要反射调用空参构造函数，所以要显示定义一个
  public FlowBean(){}

  public FlowBean(long upFlow, long dFlow) {
    this.upFlow = upFlow;
    this.dFlow = dFlow;
    this.sumFlow = upFlow + dFlow;
  }

  public void set(long upFlow, long dFlow) {
    this.upFlow = upFlow;
    this.dFlow = dFlow;
    this.sumFlow = upFlow + dFlow;
  }

  public long getUpFlow() {
    return upFlow;
  }
  public void setUpFlow(long upFlow) {
    this.upFlow = upFlow;
  }
  public long getdFlow() {
    return dFlow;
  }
  public void setdFlow(long dFlow) {
    this.dFlow = dFlow;
  }

  public long getSumFlow() {
    return sumFlow;
  }

  public void setSumFlow(long sumFlow) {
    this.sumFlow = sumFlow;
  }

  /**
	* 序列化方法
	*/
  @Override
  public void write(DataOutput out) throws IOException {
    out.writeLong(upFlow);
    out.writeLong(dFlow);
    out.writeLong(sumFlow);

  }

  /**
	 * 反序列化方法
	 * 注意：反序列化的顺序跟序列化的顺序完全一致
	 */
  @Override
  public void readFields(DataInput in) throws IOException {
    upFlow = in.readLong();
    dFlow = in.readLong();
    sumFlow = in.readLong();
  }
  
  @Override
  public String toString() {

    return upFlow + "\t" + dFlow + "\t" + sumFlow;
  }

  @Override
  public int compareTo(FlowBean o) {
    return this.sumFlow>o.getSumFlow()?-1:1;	//从大到小, 当前对象和要比较的对象比, 如果当前对象大, 返回-1, 交换他们的位置(自己的理解)
  }
}
~~~~



## 需求三：统计流量并按照手机号的归属地，将结果输出到不同的省份文件中

自定义Partition

#### FlowBean

~~~~java
public class FlowBean implements Writable{

  private long upFlow;
  private long dFlow;
  private long sumFlow;

  //反序列化时，需要反射调用空参构造函数，所以要显示定义一个
  public FlowBean(){}

  public FlowBean(long upFlow, long dFlow) {
    this.upFlow = upFlow;
    this.dFlow = dFlow;
    this.sumFlow = upFlow + dFlow;
  }


  public long getUpFlow() {
    return upFlow;
  }
  public void setUpFlow(long upFlow) {
    this.upFlow = upFlow;
  }
  public long getdFlow() {
    return dFlow;
  }
  public void setdFlow(long dFlow) {
    this.dFlow = dFlow;
  }


  public long getSumFlow() {
    return sumFlow;
  }


  public void setSumFlow(long sumFlow) {
    this.sumFlow = sumFlow;
  }


  /**
	 * 序列化方法
	 */
  @Override
  public void write(DataOutput out) throws IOException {
    out.writeLong(upFlow);
    out.writeLong(dFlow);
    out.writeLong(sumFlow);
  }

  /**
	 * 反序列化方法
	 * 注意：反序列化的顺序跟序列化的顺序完全一致
	 */
  @Override
  public void readFields(DataInput in) throws IOException {
    upFlow = in.readLong();
    dFlow = in.readLong();
    sumFlow = in.readLong();
  }

  @Override
  public String toString() {

    return upFlow + "\t" + dFlow + "\t" + sumFlow;
  }
}
~~~~

#### FlowCount

~~~java
public class FlowCount {

  static class FlowCountMapper extends Mapper<LongWritable, Text, Text, FlowBean>{
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

      String line = value.toString();		//将一行内容转成string
      String[] fields = line.split("\t");	//切分字段
      String phoneNbr = fields[1];			//取出手机号

      long upFlow = Long.parseLong(fields[fields.length-3]);	//取出上行流量下行流量
      long dFlow = Long.parseLong(fields[fields.length-2]);

      context.write(new Text(phoneNbr), new FlowBean(upFlow, dFlow));
    }
  }


  static class FlowCountReducer extends Reducer<Text, FlowBean, Text, FlowBean>{
    //<183323,bean1><183323,bean2><183323,bean3><183323,bean4>.......
    @Override
    protected void reduce(Text key, Iterable<FlowBean> values, Context context) throws IOException, InterruptedException {

      long sum_upFlow = 0;
      long sum_dFlow = 0;

      //遍历所有bean，将其中的上行流量，下行流量分别累加
      for(FlowBean bean: values){
        sum_upFlow += bean.getUpFlow();
        sum_dFlow += bean.getdFlow();
      }

      FlowBean resultBean = new FlowBean(sum_upFlow, sum_dFlow);
      context.write(key, resultBean);
    }
  }



  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    /*conf.set("mapreduce.framework.name", "yarn");
		conf.set("yarn.resoucemanager.hostname", "mini1");*/
    Job job = Job.getInstance(conf);

    /*job.setJar("/home/hadoop/wc.jar");*/
    //指定本程序的jar包所在的本地路径
    job.setJarByClass(FlowCount.class);

    //指定本业务job要使用的mapper/Reducer业务类
    job.setMapperClass(FlowCountMapper.class);
    job.setReducerClass(FlowCountReducer.class);

    //指定我们自定义的数据分区器
    job.setPartitionerClass(ProvincePartitioner.class);
    //同时指定相应“分区”数量的reducetask
    job.setNumReduceTasks(5);

    //指定mapper输出数据的kv类型
    job.setMapOutputKeyClass(Text.class);
    job.setMapOutputValueClass(FlowBean.class);

    //指定最终输出的数据的kv类型
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(FlowBean.class);

    //指定job的输入原始文件所在目录
    FileInputFormat.setInputPaths(job, new Path(args[0]));
    //指定job的输出结果所在目录
    FileOutputFormat.setOutputPath(job, new Path(args[1]));

    //将job中配置的相关参数，以及job所用的java类所在的jar包，提交给yarn去运行
    /*job.submit();*/
    boolean res = job.waitForCompletion(true);
    System.exit(res?0:1);
  }
}
~~~

#### ProvincePartitioner

~~~~java
/**
 * K2  V2  对应的是map输出kv的类型
 * @author
 */
public class ProvincePartitioner extends Partitioner<Text, FlowBean>{

  public static HashMap<String, Integer> proviceDict = new HashMap<String, Integer>();
  static{
    proviceDict.put("136", 0);
    proviceDict.put("137", 1);
    proviceDict.put("138", 2);
    proviceDict.put("139", 3);
  }

  @Override
  public int getPartition(Text key, FlowBean value, int numPartitions) {
    String prefix = key.toString().substring(0, 3);
    Integer provinceId = proviceDict.get(prefix);

    return provinceId==null?4:provinceId;
  }
}
~~~~

