---
title: MR之倒排索引的建立
categories:
  - big-data
tags:
  - MapReduce
abbrlink: c8369a3e
date: 2017-07-15 11:28:38
---

### 需求

很多个文件，里面有很多单词，要求的结果是统计每个单词，倒排索引，在每个文件中出现的次数

<br/>

# 代码实现

## InverIndexStepOne

```java
public class InverIndexStepOne {

  static class InverIndexStepOneMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

    Text k = new Text();
    IntWritable v = new IntWritable(1);

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

      String line = value.toString();

      String[] words = line.split(" ");

      FileSplit inputSplit = (FileSplit) context.getInputSplit();
      String fileName = inputSplit.getPath().getName();
      for (String word : words) {
        k.set(word + "--" + fileName);
        context.write(k, v);

      }

    }

  }

  static class InverIndexStepOneReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {

      int count = 0;
      for (IntWritable value : values) {

        count += value.get();
      }

      context.write(key, new IntWritable(count));

    }

  }

  public static void main(String[] args) throws Exception {

    Configuration conf = new Configuration();

    Job job = Job.getInstance(conf);
    job.setJarByClass(InverIndexStepOne.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);

    FileInputFormat.setInputPaths(job, new Path("D:/srcdata/inverindexinput"));
    FileOutputFormat.setOutputPath(job, new Path("D:/temp/out"));
    // FileInputFormat.setInputPaths(job, new Path(args[0]));
    // FileOutputFormat.setOutputPath(job, new Path(args[1]));

    job.setMapperClass(InverIndexStepOneMapper.class);
    job.setReducerClass(InverIndexStepOneReducer.class);

    job.waitForCompletion(true);
  }
}
```

<br/>

## IndexStepTwo

```java
/*
hello--a.txt	3
hello--b.txt	1
hello--c.txt	2
jerry--a.txt	2
jerry--b.txt	1
jerry--c.txt	3
*/
public class IndexStepTwo {
  public static class IndexStepTwoMapper extends Mapper<LongWritable, Text, Text, Text>{
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
      String line = value.toString();
      String[] files = line.split("--");
      context.write(new Text(files[0]), new Text(files[1]));
    }
  }
  public static class IndexStepTwoReducer extends Reducer<Text, Text, Text, Text>{
    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
      StringBuffer sb = new StringBuffer();
      for (Text text : values) {
        sb.append(text.toString().replace("\t", "-->") + "\t");
      }
      context.write(key, new Text(sb.toString()));
    }
  }
  public static void main(String[] args) throws Exception {

    if (args.length < 1 || args == null) {
      args = new String[]{"D:/temp/out/part-r-00000", "D:/temp/out2"};
    }

    Configuration config = new Configuration();
    Job job = Job.getInstance(config);

    job.setMapperClass(IndexStepTwoMapper.class);
    job.setReducerClass(IndexStepTwoReducer.class);
    //		job.setMapOutputKeyClass(Text.class);
    //		job.setMapOutputValueClass(Text.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(Text.class);

    FileInputFormat.setInputPaths(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));

    System.exit(job.waitForCompletion(true) ? 1:0);
  }
}
```