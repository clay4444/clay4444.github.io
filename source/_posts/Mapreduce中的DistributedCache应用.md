---
title: Mapreduce中的DistributedCache应用
categories:
  - big-data
tags:
  - MapReduce
abbrlink: 68d67dd4
date: 2018-05-16 12:51:55
---



### Map端join案例 

<br/>

#### 需求 

实现两个“表”的join操作，其中一个表数据量小，一个表很大，这种场景在实际中非常常见，比如“订单日志” join “产品信息” 

<br/>

#### 分析

-原理阐述

适用于关联表中有小表的情形；

可以将小表分发到所有的map节点，这样，map节点就可以在本地对自己所读到的大表数据进行join并输出最终结果

可以大大提高join操作的并发度，加快处理速度

<br/>

####  示例

1. 先在mapper类中预先定义好小表，进行join
2. 并用distributedcache机制将小表的数据分发到每一个maptask执行节点，从而每一个 ,maptask节点可以从本地加载到小表的数据，进而在本地即可实现join。

<br/>

#### 实现

~~~java
public class MapSideJoin {

    public static class MapSideJoinMapper extends Mapper<LongWritable, Text, Text, NullWritable> {
        // 用一个hashmap来加载保存产品信息表
        Map<String, String> pdInfoMap = new HashMap<String, String>();

        Text k = new Text();

        /**
		 * 通过阅读父类Mapper的源码，发现 setup方法是在maptask处理数据之前调用一次 可以用来做一些初始化工作
		 */
        @Override
        protected void setup(Context context) throws IOException, InterruptedException {
            BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("pdts.txt")));
            String line;
            while (StringUtils.isNotEmpty(line = br.readLine())) {
                String[] fields = line.split(",");
                pdInfoMap.put(fields[0], fields[1]);
            }
            br.close();
        }

        // 由于已经持有完整的产品信息表，所以在map方法中就能实现join逻辑了
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            String orderLine = value.toString();
            String[] fields = orderLine.split("\t");
            String pdName = pdInfoMap.get(fields[1]);
            k.set(orderLine + "\t" + pdName);
            context.write(k, NullWritable.get());
        }

    }

    public static void main(String[] args) throws Exception {

        Configuration conf = new Configuration();

        Job job = Job.getInstance(conf);

        job.setJarByClass(MapSideJoin.class);

        job.setMapperClass(MapSideJoinMapper.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(NullWritable.class);

        FileInputFormat.setInputPaths(job, new Path("D:/srcdata/mapjoininput"));
        FileOutputFormat.setOutputPath(job, new Path("D:/temp/output"));

        // 指定需要缓存一个文件到所有的maptask运行节点工作目录
        /* job.addArchiveToClassPath(archive); */// 缓存jar包到task运行节点的classpath中
        /* job.addFileToClassPath(file); */// 缓存普通文件到task运行节点的classpath中
        /* job.addCacheArchive(uri); */// 缓存压缩包文件到task运行节点的工作目录
        /* job.addCacheFile(uri) */// 缓存普通文件到task运行节点的工作目录

        // 将产品表文件缓存到task工作节点的工作目录中去
        job.addCacheFile(new URI("file:/D:/srcdata/mapjoincache/pdts.txt"));

        //map端join的逻辑不需要reduce阶段，设置reducetask数量为0
        job.setNumReduceTasks(0);

        boolean res = job.waitForCompletion(true);
        System.exit(res ? 0 : 1);
    }
}
~~~

<br/>

<br/>

### Reduce端join案例

- 思路就是通过pid相同的两者拼接到一起。

图示：

{% asset_img 11.png %}

<br/>

代码：

#### InfoBean

~~~~~java
public class InfoBean implements Writable {

    private int order_id;
    private String dateString;
    private String p_id;
    private int amount;
    private String pname;
    private int category_id;
    private float price;

    // flag=0表示这个对象是封装订单表记录
    // flag=1表示这个对象是封装产品信息记录
    private String flag;

    public InfoBean() {
    }

    public void set(int order_id, String dateString, String p_id, int amount, String pname, int category_id, float price, String flag) {
        this.order_id = order_id;
        this.dateString = dateString;
        this.p_id = p_id;
        this.amount = amount;
        this.pname = pname;
        this.category_id = category_id;
        this.price = price;
        this.flag = flag;
    }

    public int getOrder_id() {
        return order_id;
    }

    public void setOrder_id(int order_id) {
        this.order_id = order_id;
    }

    public String getDateString() {
        return dateString;
    }

    public void setDateString(String dateString) {
        this.dateString = dateString;
    }

    public String getP_id() {
        return p_id;
    }

    public void setP_id(String p_id) {
        this.p_id = p_id;
    }

    public int getAmount() {
        return amount;
    }

    public void setAmount(int amount) {
        this.amount = amount;
    }

    public String getPname() {
        return pname;
    }

    public void setPname(String pname) {
        this.pname = pname;
    }

    public int getCategory_id() {
        return category_id;
    }

    public void setCategory_id(int category_id) {
        this.category_id = category_id;
    }

    public float getPrice() {
        return price;
    }

    public void setPrice(float price) {
        this.price = price;
    }

    public String getFlag() {
        return flag;
    }

    public void setFlag(String flag) {
        this.flag = flag;
    }

    /**
	 * private int order_id; private String dateString; private int p_id;
	 * private int amount; private String pname; private int category_id;
	 * private float price;
	 */
    @Override
    public void write(DataOutput out) throws IOException {
        out.writeInt(order_id);
        out.writeUTF(dateString);
        out.writeUTF(p_id);
        out.writeInt(amount);
        out.writeUTF(pname);
        out.writeInt(category_id);
        out.writeFloat(price);
        out.writeUTF(flag);

    }

    @Override
    public void readFields(DataInput in) throws IOException {
        this.order_id = in.readInt();
        this.dateString = in.readUTF();
        this.p_id = in.readUTF();
        this.amount = in.readInt();
        this.pname = in.readUTF();
        this.category_id = in.readInt();
        this.price = in.readFloat();
        this.flag = in.readUTF();

    }

    @Override
    public String toString() {
        return "order_id=" + order_id + ", dateString=" + dateString + ", p_id=" + p_id + ", amount=" + amount + ", pname=" + pname + ", category_id=" + category_id + ", price=" + price + ", flag=" + flag;
    }

}
~~~~~

<br/>

#### RJoin

~~~~java
/**
 * 订单表和商品表合到一起
order.txt(订单id, 日期, 商品编号, 数量)
	1001	20150710	P0001	2
	1002	20150710	P0001	3
	1002	20150710	P0002	3
	1003	20150710	P0003	3
product.txt(商品编号, 商品名字, 价格, 数量)
	P0001	小米5	1001	2
	P0002	锤子T1	1000	3
	P0003	锤子	1002	4
 * @author: clay
 */
public class RJoin {

    static class RJoinMapper extends Mapper<LongWritable, Text, Text, InfoBean> {
        InfoBean bean = new InfoBean();
        Text k = new Text();

        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

            String line = value.toString();

            FileSplit inputSplit = (FileSplit) context.getInputSplit();
            String name = inputSplit.getPath().getName();
            // 通过文件名判断是哪种数据
            String pid = "";
            if (name.startsWith("order")) {
                String[] fields = line.split("\t");
                // id date pid amount
                pid = fields[2];
                bean.set(Integer.parseInt(fields[0]), fields[1], pid, Integer.parseInt(fields[3]), "", 0, 0, "0");

            } else {
                String[] fields = line.split("\t");
                // id pname category_id price
                pid = fields[0];
                bean.set(0, "", pid, 0, fields[1], Integer.parseInt(fields[2]), Float.parseFloat(fields[3]), "1");

            }
            k.set(pid);
            context.write(k, bean);
        }

    }

    static class RJoinReducer extends Reducer<Text, InfoBean, InfoBean, NullWritable> {

        @Override
        protected void reduce(Text pid, Iterable<InfoBean> beans, Context context) throws IOException, InterruptedException {
            InfoBean pdBean = new InfoBean();
            ArrayList<InfoBean> orderBeans = new ArrayList<InfoBean>();

            for (InfoBean bean : beans) {
                if ("1".equals(bean.getFlag())) {	//产品的
                    try {
                        BeanUtils.copyProperties(pdBean, bean);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                } else {
                    InfoBean odbean = new InfoBean();
                    try {
                        BeanUtils.copyProperties(odbean, bean);
                        orderBeans.add(odbean);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }

            }

            // 拼接两类数据形成最终结果
            for (InfoBean bean : orderBeans) {

                bean.setPname(pdBean.getPname());
                bean.setCategory_id(pdBean.getCategory_id());
                bean.setPrice(pdBean.getPrice());

                context.write(bean, NullWritable.get());
            }
        }
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();

        conf.set("mapred.textoutputformat.separator", "\t");

        Job job = Job.getInstance(conf);

        // 指定本程序的jar包所在的本地路径
        // job.setJarByClass(RJoin.class);
        //		job.setJar("c:/join.jar");

        job.setJarByClass(RJoin.class);
        // 指定本业务job要使用的mapper/Reducer业务类
        job.setMapperClass(RJoinMapper.class);
        job.setReducerClass(RJoinReducer.class);

        // 指定mapper输出数据的kv类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(InfoBean.class);

        // 指定最终输出的数据的kv类型
        job.setOutputKeyClass(InfoBean.class);
        job.setOutputValueClass(NullWritable.class);

        // 指定job的输入原始文件所在目录
        FileInputFormat.setInputPaths(job, new Path(args[0]));
        // 指定job的输出结果所在目录
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        // 将job中配置的相关参数，以及job所用的java类所在的jar包，提交给yarn去运行
        /* job.submit(); */
        boolean res = job.waitForCompletion(true);
        System.exit(res ? 0 : 1);
    }
}
~~~~

