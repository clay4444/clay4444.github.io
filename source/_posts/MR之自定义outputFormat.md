---
title: MR之自定义outputFormat
categories:
  - big-data
tags:
  - MapReduce
abbrlink: 403d9f31
date: 2018-05-16 12:42:15
---



### 需求

现有一些原始日志需要做增强解析处理，流程： 

1.  从原始日志文件中读取数据 
2. 根据日志中的一个URL字段到外部知识库中获取信息增强到原始日志 
3.  如果成功增强，则输出到增强结果目录；如果增强失败，则抽取原始数据中URL字段输出到待爬清单目录 

<br/>

### 分析

实现要点： 

1. 在mapreduce中访问外部资源 
2. 自定义outputformat，改写其中的recordwriter，改写具体输出数据的方法write() 

<br/>

### 实现

#### 数据库获取数据的工具

~~~java
public class DBLoader {

    public static void dbLoader(HashMap<String, String> ruleMap) {
        Connection conn = null;
        Statement st = null;
        ResultSet res = null;

        try {
            Class.forName("com.mysql.jdbc.Driver");
            conn = DriverManager.getConnection("jdbc:mysql://hdp-node01:3306/urlknowledge", "root", "root");
            st = conn.createStatement();
            res = st.executeQuery("select url,content from urlcontent");
            while (res.next()) {
                ruleMap.put(res.getString(1), res.getString(2));
            }
        } catch (Exception e) {
            e.printStackTrace();

        } finally {
            try{
                if(res!=null){
                    res.close();
                }
                if(st!=null){
                    st.close();
                }
                if(conn!=null){
                    conn.close();
                }

            }catch(Exception e){
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        DBLoader db = new DBLoader();
        HashMap<String, String> map = new HashMap<String,String>();
        db.dbLoader(map);
        System.out.println(map.size());
    }
}
~~~

<br/>

#### 自定义一个outputformat 

~~~java
public class LogEnhancerOutputFormat extends FileOutputFormat<Text, NullWritable>{

    @Override
    public RecordWriter<Text, NullWritable> getRecordWriter(TaskAttemptContext context) throws IOException, InterruptedException {

        FileSystem fs = FileSystem.get(context.getConfiguration());
        Path enhancePath = new Path("hdfs://hdp-node01:9000/flow/enhancelog/enhanced.log");
        Path toCrawlPath = new Path("hdfs://hdp-node01:9000/flow/tocrawl/tocrawl.log");

        FSDataOutputStream enhanceOut = fs.create(enhancePath);
        FSDataOutputStream toCrawlOut = fs.create(toCrawlPath);

        return new MyRecordWriter(enhanceOut,toCrawlOut);
    }

    static class MyRecordWriter extends RecordWriter<Text, NullWritable>{

        FSDataOutputStream enhanceOut = null;
        FSDataOutputStream toCrawlOut = null;

        public MyRecordWriter(FSDataOutputStream enhanceOut, FSDataOutputStream toCrawlOut) {
            this.enhanceOut = enhanceOut;
            this.toCrawlOut = toCrawlOut;
        }

        @Override
        public void write(Text key, NullWritable value) throws IOException, InterruptedException {

            //有了数据，你来负责写到目的地  —— hdfs
            //判断，进来内容如果是带tocrawl的，就往待爬清单输出流中写 toCrawlOut
            if(key.toString().contains("tocrawl")){
                toCrawlOut.write(key.toString().getBytes());
            }else{
                enhanceOut.write(key.toString().getBytes());
            }

        }

        @Override
        public void close(TaskAttemptContext context) throws IOException, InterruptedException {

            if(toCrawlOut!=null){
                toCrawlOut.close();
            }
            if(enhanceOut!=null){
                enhanceOut.close();
            }

        }	
    }
}
~~~

<br/>

#### 开发mapreduce处理流程 

~~~~java
/**
 * 这个程序是对每个小时不断产生的用户上网记录日志进行增强(将日志中的url所指向的网页内容分析结果信息追加到每一行原始日志后面)
 */
public class LogEnhancer {

    static class LogEnhancerMapper extends Mapper<LongWritable, Text, Text, NullWritable> {

        HashMap<String, String> knowledgeMap = new HashMap<String, String>();

        /**
		 * maptask在初始化时会先调用setup方法一次 利用这个机制，将外部的知识库加载到maptask执行的机器内存中
		 */
        @Override
        protected void setup(org.apache.hadoop.mapreduce.Mapper.Context context) throws IOException, InterruptedException {

            DBLoader.dbLoader(knowledgeMap);

        }

        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

            String line = value.toString();

            String[] fields = StringUtils.split(line, "\t");

            try {
                String url = fields[26];

                // 对这一行日志中的url去知识库中查找内容分析信息
                String content = knowledgeMap.get(url);

                // 根据内容信息匹配的结果，来构造两种输出结果
                String result = "";
                if (null == content) {
                    // 输往待爬清单的内容
                    result = url + "\t" + "tocrawl\n";
                } else {
                    // 输往增强日志的内容
                    result = line + "\t" + content + "\n";
                }

                context.write(new Text(result), NullWritable.get());
            } catch (Exception e) {

            }
        }
    }

    public static void main(String[] args) throws Exception {

        Configuration conf = new Configuration();

        Job job = Job.getInstance(conf);

        job.setJarByClass(LogEnhancer.class);

        job.setMapperClass(LogEnhancerMapper.class);

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(NullWritable.class);

        // 要将自定义的输出格式组件设置到job中
        job.setOutputFormatClass(LogEnhancerOutputFormat.class);

        FileInputFormat.setInputPaths(job, new Path(args[0]));

        // 虽然我们自定义了outputformat，但是因为我们的outputformat继承自fileoutputformat
        // 而fileoutputformat要输出一个_SUCCESS文件，所以，在这还得指定一个输出目录
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        job.waitForCompletion(true);
        System.exit(0);
    }
}
~~~~